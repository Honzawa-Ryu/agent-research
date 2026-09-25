# Slurm チートシート

> 対象: Slurm 23.11 系（このマシンは `slurm-wlm 23.11.4`）
> 公式: https://slurm.schedmd.com/documentation.html

## 1. 基本概念

| 用語 | 意味 |
|:---|:---|
| ノード (node) | 計算機1台 |
| パーティション (partition) | ノードのグループ。キューに相当する |
| ジョブ (job) | `sbatch` / `srun` / `salloc` で確保したリソースの単位 |
| ジョブステップ (step) | ジョブ内で `srun` を1回実行した単位 |
| タスク (task) | プロセス。`--ntasks` で数を指定する |
| GRES | GPU などの汎用リソース |

## 2. ジョブの投げ方（3種類）

| コマンド | 用途 | ブロックするか |
|:---|:---|:---:|
| `sbatch job.sh` | バッチ実行（本番向け） | しない |
| `srun <cmd>` | 1コマンドを即実行。ジョブ内ではステップを起動する | する |
| `salloc` | リソースを確保してシェルを返す（対話作業用） | する |

```bash
sbatch job.sh                          # スクリプト投入 → "Submitted batch job 12345"
sbatch --parsable job.sh               # ジョブIDだけを出力（スクリプトから使う用）
srun --gres=gpu:1 --pty bash           # GPU付きの対話シェル
srun -p <partition> -c 4 --mem=16G python a.py
salloc --gres=gpu:1 -t 2:00:00         # 確保 → その中で srun を複数回実行
```

## 3. sbatch スクリプトの雛形

```bash
#!/bin/bash
#SBATCH --job-name=exp01
#SBATCH --partition=<partition>
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --gres=gpu:1
#SBATCH --time=12:00:00
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err

set -euo pipefail
echo "job=$SLURM_JOB_ID node=$SLURMD_NODENAME gpus=${CUDA_VISIBLE_DEVICES:-none}"

srun python train.py --config config.yaml
```

- `#SBATCH` 行は最初の実行行より前に書く。途中に書いた分は無視される
- `logs/` は事前に作っておく。存在しないと出力されずにジョブが失敗する
- コマンドラインで渡した引数が `#SBATCH` より優先される: `sbatch --time=1:00:00 job.sh`

### よく使うオプション

| オプション | 短縮形 | 例 | 備考 |
|:---|:---|:---|:---|
| `--partition` | `-p` | `-p gpu` | |
| `--time` | `-t` | `-t 1-00:00:00` | `D-HH:MM:SS`。超過すると kill される |
| `--cpus-per-task` | `-c` | `-c 8` | DataLoader の `num_workers` の目安 |
| `--mem` | | `--mem=64G` | ノードあたり。`--mem-per-cpu` とは同時指定不可 |
| `--gres` | | `--gres=gpu:2` / `--gres=gpu:a100:1` | GPU型の指定は環境による |
| `--gpus` | `-G` | `-G 1` | `--gres` の新しい書き方 |
| `--nodelist` | `-w` | `-w node01` | 特定ノードを指定 |
| `--exclude` | `-x` | `-x node03` | 特定ノードを除外 |
| `--array` | `-a` | `-a 0-9%2` | 後述 |
| `--dependency` | `-d` | `-d afterok:12345` | 後述 |
| `--chdir` | `-D` | `-D /path` | 作業ディレクトリ |
| `--export` | | `--export=ALL,SEED=1` | 環境変数を渡す |
| `--mail-type` | | `--mail-type=END,FAIL` | 通知 |

### 出力ファイル名のパターン

| パターン | 置換される値 |
|:---|:---|
| `%j` | ジョブID |
| `%x` | ジョブ名 |
| `%A` / `%a` | アレイの親ID / インデックス |
| `%N` | ノード名 |
| `%u` | ユーザー名 |

## 4. アレイジョブ（パラメータスイープ）

```bash
#SBATCH --array=0-9%3            # 0〜9 の10本、同時実行は最大3本
#SBATCH --output=logs/%x_%A_%a.out

SEEDS=(0 1 2 3 4 5 6 7 8 9)
python train.py --seed "${SEEDS[$SLURM_ARRAY_TASK_ID]}"
```

```bash
# 設定ファイルの一覧を1行ずつ割り当てる
CFG=$(sed -n "$((SLURM_ARRAY_TASK_ID + 1))p" configs.txt)
```

- `--array=1,3,5` / `--array=0-20:2`（2刻み）のようにも書ける
- アレイ全体をキャンセル: `scancel 12345`。1本だけなら `scancel 12345_3`

## 5. 依存関係（ジョブの連結）

```bash
jid=$(sbatch --parsable preprocess.sh)
sbatch --dependency=afterok:$jid train.sh
```

| 指定 | 開始条件 |
|:---|:---|
| `after:<id>` | 対象が開始したら |
| `afterok:<id>` | 対象が正常終了したら |
| `afternotok:<id>` | 対象が失敗したら |
| `afterany:<id>` | 対象が終了したら（結果は問わない） |
| `singleton` | 同名・同ユーザーのジョブが1本ずつ走る |

依存先が失敗すると、`afterok` のジョブは `DependencyNeverSatisfied` のまま残る。手動で `scancel` する。

## 6. 状況確認

```bash
squeue --me                               # 自分のジョブ一覧
squeue --me -o "%.10i %.20j %.8T %.10M %.6D %R"   # 列を指定
squeue --me --start                       # 開始予定時刻
sinfo                                     # パーティションとノードの状態
sinfo -N -o "%N %P %T %G %C %m"           # ノード別の GPU/CPU/メモリ
scontrol show job <id>                    # ジョブの詳細（実行中・待機中）
scontrol show node <node>                 # ノードの詳細
sacct -j <id> --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS,ReqMem
sacct --me -S 2026-09-01                  # 期間を指定して履歴を出す
seff <id>                                 # 終了後の CPU/メモリ効率（入っていれば）
```

### ジョブ状態 (`squeue` の ST 列)

| 略号 | 状態 | 意味 |
|:---:|:---|:---|
| PD | PENDING | 待機中。理由は REASON 列 |
| R | RUNNING | 実行中 |
| CG | COMPLETING | 終了処理中 |
| CD | COMPLETED | 正常終了 |
| F | FAILED | 非0で終了 |
| TO | TIMEOUT | 時間切れ |
| OOM | OUT_OF_MEMORY | メモリ超過で kill |
| CA | CANCELLED | キャンセルされた |

### PD の主な理由 (REASON)

| 理由 | 意味 |
|:---|:---|
| `Resources` | 空きリソース待ち |
| `Priority` | 優先度が高いジョブが先に並んでいる |
| `Dependency` | 依存先の終了待ち |
| `QOSMaxGRESPerUser` など | ユーザー上限に達している |
| `ReqNodeNotAvail` | 指定ノードが使えない（メンテ・down） |

## 7. 操作

```bash
scancel <id>                  # キャンセル
scancel --me                  # 自分のジョブを全部キャンセル（注意）
scancel --me -n exp01         # 名前で指定
scancel --me -t PENDING       # 待機中だけ
scontrol hold <id>            # 待機中のジョブを保留
scontrol release <id>         # 保留を解除
scontrol update JobId=<id> TimeLimit=2:00:00   # 待機中なら変更可（延長は管理者権限が必要な場合あり）
```

## 8. ジョブ内で使える環境変数

| 変数 | 内容 |
|:---|:---|
| `SLURM_JOB_ID` | ジョブID |
| `SLURM_JOB_NAME` | ジョブ名 |
| `SLURM_ARRAY_JOB_ID` / `SLURM_ARRAY_TASK_ID` | アレイの親ID / インデックス |
| `SLURM_CPUS_PER_TASK` | `-c` の値（指定した場合のみ） |
| `SLURM_JOB_NODELIST` | 割り当てノード |
| `SLURMD_NODENAME` | 実行中のノード名 |
| `SLURM_SUBMIT_DIR` | 投入したディレクトリ |
| `CUDA_VISIBLE_DEVICES` | 割り当てGPU（Slurm が設定する） |

## 9. Apptainer と組み合わせる

```bash
srun apptainer exec --nv \
  --bind /data:/data \
  env.sif python train.py
```

- `--nv`: ホストの NVIDIA ドライバをコンテナに渡す。付け忘れると GPU が見えない
- `CUDA_VISIBLE_DEVICES` はコンテナ内にも引き継がれる
- `--bind` でマウントしていないパスはコンテナ内から見えない

## 10. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| 投入直後に FAILED、ログが空 | `--output` のディレクトリが存在しない |
| `OUT_OF_MEMORY` | `--mem` を増やす。`sacct` の `MaxRSS` で実使用量を確認 |
| `TIMEOUT` | `--time` を延ばす。チェックポイントから再開できるようにしておく |
| GPU が見えない | `--gres` の指定漏れ、または Apptainer の `--nv` 漏れ |
| DataLoader が遅い / 固まる | `num_workers` が `-c` を超えている |
| ずっと PD | `squeue --me` の REASON を見る。`sinfo` で空きを確認 |
| ログがなかなか出ない | Python の出力バッファ。`python -u` か `PYTHONUNBUFFERED=1` を使う |
| ログインノードで重い処理をしてしまった | 計算は必ず `srun` / `sbatch` 経由で実行する |
