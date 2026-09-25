# Slurm 詳細編（応用・トラブル対応）

> 基本編: [slurm.md](slurm.md)
> 対象バージョン: Slurm 23.11 系（このマシンは `slurm-wlm 23.11.4`）
> 公式: https://slurm.schedmd.com/documentation.html
> 注: 基本編の `sacct --me` は 23.11.4 の sacct では使えない（`unrecognized option`）。本シートでは `sacct -u $USER` を使う

## Part 1. 応用・上級機能

### 1.1 このクラスタの設定を知る

挙動の多くはクラスタ側の設定で決まる。まず手元で確認する。

```bash
scontrol show config | grep -iE "KillWait|OverTimeLimit|JobRequeue|DefMemPer|SelectTypeParameters|PriorityType|PriorityWeight|EnforcePartLimits|MaxArraySize|MinJobAge"
scontrol show partition <partition>        # MaxTime, PriorityTier, TRES など
scontrol show node <node>                  # CPU数, RealMemory, Gres, Features
sinfo -N -O "NodeList:12,Partition:20,StateCompact:8,CPUsState:16,Memory:10,AllocMem:10,Gres:12,GresUsed:24"
```

2026-09-25 時点の主な値:

| 項目 | 値 | 意味 |
|:---|:---|:---|
| `KillWait` | 30 sec | 時間切れ・キャンセル時、SIGTERM から SIGKILL までの猶予 |
| `OverTimeLimit` | 0 min | `--time` を1分も超過できない |
| `JobRequeue` | 1 | ノード障害時などに自動で再キューされうる（`--no-requeue` で無効化） |
| `DefMemPerNode` | UNLIMITED | `--mem` を省略するとノードのメモリ全体を要求する扱いになる |
| `SelectTypeParameters` | CR_CORE_MEMORY | CPU コアとメモリの両方が消費リソース。どちらかが足りないと PD |
| `PriorityWeight*` | すべて 0 | multifactor だが重みが 0。優先度（`squeue` の PRIORITY）は実質使われない |
| `EnforcePartLimits` | NO | パーティション上限を超える `--time` でも投入は通り、PD のまま残る |
| `MaxArraySize` | 1001 | アレイのインデックスは 0〜1000 まで |
| `MinJobAge` | 300 sec | 終了後5分で `squeue` / `scontrol show job` から消える。以降は `sacct` で見る |

パーティションの構成（`sinfo -s`）:

| 規模 | MaxTime | PriorityTier（andre01 の例） |
|:---|:---|---:|
| `small-*` | 1:00:00 | 1 |
| `medium-*` | 2:00:00 | 20 |
| `large-*` | 4:00:00 | 50 |
| `x-large-*` | 無制限 | 100 |

- 同じノードを取り合うときは PriorityTier が高いパーティション（長いほう）のジョブが先に選ばれる
- `andre01` ノードは 32 CPU / 240000M / GPU (RTX A6000) 4枚。他のノードは GPU 1枚
- ノードに Features が設定されていないので、`--constraint` はこのクラスタでは使い道がない

### 1.2 リソース指定の追加オプション

| オプション | 例 | 説明 |
|:---|:---|:---|
| `--ntasks-per-node` | `--ntasks-per-node=4` | ノードあたりのタスク数。1ノード複数GPUで1GPU=1プロセスにするとき |
| `--gpus-per-node` | `--gpus-per-node=2` | ノードあたりのGPU数 |
| `--gpus-per-task` | `--gpus-per-task=1` | タスクあたりのGPU数。`--ntasks` と組み合わせる |
| `--cpus-per-gpu` | `--cpus-per-gpu=8` | GPU 1枚あたりのCPU数 |
| `--mem-per-gpu` | `--mem-per-gpu=50G` | GPU 1枚あたりのメモリ。`--mem` / `--mem-per-cpu` と排他 |
| `--exclusive` | | ノードを占有する。他人のジョブと同居しない（共有ノードでは使わない） |
| `--constraint` / `-C` | `-C a100` | ノードの Feature で絞る（Feature が設定されている環境のみ） |
| `--qos` / `-q` | `-q normal` | QOS の指定。一覧は `sacctmgr show qos` |
| `--account` / `-A` | `-A <account>` | 課金先アカウント。自分の所属は `sshare -U` で見える |
| `--reservation` | `--reservation=<name>` | 予約枠で実行する |
| `--hold` / `-H` | | 保留状態で投入。`scontrol release` で開始 |
| `--begin` | `--begin=now+1hour` / `--begin=22:00` | 開始時刻を遅らせる |
| `--deadline` | `--deadline=2026-10-01T09:00` | この時刻までに終われないなら取り消す |
| `--time-min` | `--time-min=2:00:00` | `--time` より短くてもよい下限。backfill で早く入りやすくなる |
| `--nice` | `--nice=100` | 自分の優先度を下げる |
| `--test-only` | | 実際には投入せず、開始見込みだけ表示する |
| `--wait` / `-W` | | ジョブ終了まで sbatch が戻らない。スクリプトから同期実行したいとき |
| `--wrap` | `--wrap="python a.py"` | スクリプトファイルを作らずに1行コマンドを投入する |
| `--kill-on-invalid-dep` | `--kill-on-invalid-dep=yes` | 依存が満たせなくなったら自動で取り消す（2.9節） |

```bash
sbatch --test-only job.sh                             # 開始見込み時刻と割当ノードを表示
sbatch -p small-andre01 -c 4 --mem=8G -t 30 --wrap="python scripts/check.py"
sbatch --parsable --wait job.sh && echo "done"        # 終了まで待つ
```

### 1.3 シグナルとチェックポイント

`--signal=[{R|B}:]<sig>[@<秒>]` で、時間切れの `<秒>` 前にシグナルを送らせる。

| 指定 | 送り先 |
|:---|:---|
| `--signal=USR1@300` | ジョブステップ（`srun` で起動したプロセス）。バッチシェル自身には送られない |
| `--signal=B:USR1@300` | バッチシェルだけ。子プロセスには送られない |
| `--signal=R:USR1@300` | 予約（reservation）との衝突でも送る |

- 送信タイミングは最大60秒程度ずれることがある。余裕を持って設定する
- bash は**フォアグラウンドのコマンドが終わるまで trap を実行しない**。`B:` で受けるときは、本体を `&` で起動して `wait` で待つ

```bash
#!/bin/bash
#SBATCH --time=4:00:00
#SBATCH --signal=B:USR1@600
#SBATCH --requeue
#SBATCH --open-mode=append
#SBATCH --output=logs/%x_%j.out

on_usr1() {
    echo "USR1 received $(date), restart_count=${SLURM_RESTART_COUNT:-0}"
    kill -USR1 "${PID}"          # Python 側でチェックポイントを保存させる
    wait "${PID}"                # 保存が終わるまで待つ
    scontrol requeue "${SLURM_JOB_ID}"   # 同じジョブIDで並び直す
}
trap on_usr1 USR1

python -u train.py --resume auto &
PID=$!
wait "${PID}"
```

Python 側:

```python
import signal
stop = False
def handler(signum, frame):
    global stop
    stop = True                       # ハンドラ内では重い処理をしない
signal.signal(signal.SIGUSR1, handler)

for step in range(start, total):
    train_step()
    if stop:
        save_checkpoint(step)
        sys.exit(0)
```

| 関連オプション・変数 | 説明 |
|:---|:---|
| `--requeue` / `--no-requeue` | 再キューを許可する / しない |
| `--open-mode=append` | 再キュー後もログを上書きせず追記する（既定は truncate） |
| `SLURM_RESTART_COUNT` | 再キューされた回数（再キューされたジョブでのみ設定される） |
| `scontrol requeue <id>` | 実行中・終了済みのバッチジョブを並び直す |
| `scontrol requeuehold <id>` | 並び直して保留にする |

### 1.4 ジョブステップの並列実行

1つのジョブの中で `srun` を複数回呼ぶと、それぞれがステップ（`<jobid>.0`, `<jobid>.1`, ...）になる。

```bash
#SBATCH --ntasks=4
#SBATCH --cpus-per-task=4
#SBATCH --gpus=4

for i in 0 1 2 3; do
    srun --ntasks=1 --cpus-per-task=4 --gpus=1 --exact \
        python eval.py --fold "$i" > "logs/fold_${i}.log" 2>&1 &
done
wait
```

| オプション | 意味 |
|:---|:---|
| `--exact` | ステップに指定した分だけのリソースを使う。並列ステップでは付ける |
| `--overlap` | 他のステップとリソースを共有してよい。監視用のステップに使う |
| `-l` / `--label` | 出力の行頭にタスク番号を付ける |
| `-K` / `--kill-on-bad-exit` | 1タスクでも失敗したら全タスクを止める |
| `--multi-prog` | タスクごとに別コマンドを割り当てる（設定ファイルを渡す） |

実行中のジョブのノードに入る（別ターミナルから）:

```bash
srun --jobid=<id> --overlap --pty bash      # そのジョブの割当の中でシェルを開く
srun --jobid=<id> --overlap nvidia-smi      # GPU の使用状況だけ見る
```

### 1.5 salloc を使った対話作業

```bash
salloc -p small-andre01 -c 8 --mem=32G --gpus=1 -t 1:00:00
# 確保されると、ログインしたノードのシェルに戻る（計算ノードではない）
srun --pty bash                    # 計算ノードに入る
srun python quick_test.py          # ステップとして実行
exit                               # 確保を解放
```

- `salloc` 直後のシェルは投入元のノードで動いている。計算は `srun` 経由で行う
- 確保中は `squeue --me` に1本のジョブとして見える。`exit` し忘れると時間切れまで GPU を占有する

### 1.6 アレイジョブの応用

```bash
scontrol update JobId=<array_id> ArrayTaskThrottle=4   # 同時実行数を後から変える（%N の変更）
scontrol hold <array_id>_[5-9]                         # 一部のタスクだけ保留
scancel <array_id>_[10-19]                             # 範囲でキャンセル
squeue --me -r                                         # タスクを1行ずつ展開して表示
```

| 変数 | 内容 |
|:---|:---|
| `SLURM_ARRAY_TASK_COUNT` | タスク数 |
| `SLURM_ARRAY_TASK_MIN` / `_MAX` | インデックスの最小 / 最大 |
| `SLURM_ARRAY_TASK_STEP` | 刻み幅 |

失敗したタスクだけ投げ直す:

```bash
sacct -j <array_id> -X -n -P --format=JobID,State \
  | awk -F'|' '$2!="COMPLETED"{split($1,a,"_"); print a[2]}' | paste -sd, -
# → "3,7,12" などが出るので
sbatch --array=3,7,12 job.sh
```

アレイと依存関係:

| 指定 | 開始条件 |
|:---|:---|
| `afterok:<array_id>` | アレイの全タスクが正常終了したら |
| `afterok:<array_id>_<n>` | 特定タスクが正常終了したら |
| `aftercorr:<array_id>` | 同じインデックスのタスクが正常終了したら（タスク同士を1対1で連結） |

### 1.7 依存関係の応用

```bash
sbatch -d afterok:111:222 c.sh                 # 111 と 222 の両方が正常終了したら
sbatch -d "afterok:111?afterok:222" c.sh       # どちらか一方でよい（? は OR）
sbatch -d afterany:111 --kill-on-invalid-dep=yes c.sh
scontrol update JobId=<id> Dependency=afterok:333   # 待機中の依存を付け替える
scontrol update JobId=<id> Dependency=              # 依存を外す
scontrol show job <id> | grep -E "Dependency|Reason"
```

パイプラインをまとめて投入する例:

```bash
j1=$(sbatch --parsable prep.sh)
j2=$(sbatch --parsable -d afterok:$j1 --array=0-9 train.sh)
j3=$(sbatch --parsable -d afterok:$j2 eval.sh)
echo "prep=$j1 train=$j2 eval=$j3"
```

### 1.8 sacct / sstat の使い分け

| コマンド | 対象 | 取れるもの |
|:---|:---|:---|
| `sstat` | 実行中のジョブ | 各ステップの現在のメモリ・CPU 使用量 |
| `sacct` | 終了済み（実行中も可） | 状態、終了コード、経過時間、最大メモリなど |

```bash
sstat -j <id>.batch --format=JobID,MaxRSS,AveRSS,AveCPU     # sbatch 本体は .batch ステップ
sstat -j <id> -a --format=JobID,MaxRSS                      # srun ステップを全部

sacct -u $USER -X -S now-7days --format=JobID%14,JobName%24,Partition%16,State%14,ExitCode,Elapsed
sacct -u $USER -X -S 2026-09-01 -s OOM,TO,F,NF              # 状態で絞る
sacct -j <id> -P --format=JobID,State,ExitCode,MaxRSS,ReqMem --units=G   # | 区切り（スクリプト向け）
sacct -j <id> -D --format=JobID,State,Start,End             # 再キュー前の記録も表示
sacct -j <id> -X --format=SubmitLine%80,WorkDir%60          # 投入コマンドと作業ディレクトリ
```

- `sacct` は `-S` を省略すると当日0時以降のジョブしか出さない
- `-X`（`--allocations`）でステップ行を省く
- 利用できる列名の一覧: `sacct --helpformat`

`ExitCode` は `終了コード:シグナル番号` の形:

| 表示 | 意味 |
|:---|:---|
| `0:0` | 正常終了 |
| `1:0` | スクリプトが exit 1 した |
| `0:9` | SIGKILL で終了（OOM、KillWait 超過など） |
| `0:15` | SIGTERM で終了（キャンセル、時間切れ） |
| `137:0` / `143:0` | 子プロセスが 128+9 / 128+15 で終わり、その値をシェルが返した |

基本編にない状態:

| 状態 | 意味 |
|:---|:---|
| `NODE_FAIL` (NF) | ノード障害で終了 |
| `BOOT_FAIL` (BF) | ノードの起動に失敗 |
| `PREEMPTED` (PR) | 優先ジョブに追い出された |
| `REQUEUED` (RQ) | 再キューされた |
| `DEADLINE` (DL) | `--deadline` に間に合わず取り消された |
| `SUSPENDED` (S) | 一時停止中 |

### 1.9 squeue / sinfo の表示形式

`-o` は `%` 記法、`-O` は列名で指定する。

```bash
squeue --me -O "JobID:10,Partition:18,Name:30,State:10,Reason:24,TimeUsed:12,TimeLeft:12,NodeList:10"
squeue --me -t PD -o "%.10i %.18P %.30j %.20R %.19S"   # %S: 開始見込み
squeue -p x-large-andre01                              # パーティションを指定（他人のジョブも見える）
squeue -w andre01                                      # ノードを指定
sinfo -o "%.20P %.5a %.10l %.6D %.8t %N"               # パーティション・状態・ノード
sinfo -R                                               # down / drain のノードと理由
```

| `-o` 記号 | 内容 | `-o` 記号 | 内容 |
|:---|:---|:---|:---|
| `%i` | ジョブID | `%T` | 状態（長い名前） |
| `%P` | パーティション | `%R` | 理由 または ノード名 |
| `%j` | ジョブ名 | `%M` | 経過時間 |
| `%u` | ユーザー | `%L` | 残り時間 |
| `%D` | ノード数 | `%S` | 開始時刻（見込み） |
| `%C` | CPU数 | `%b` | GRES（GPU） |
| `%m` | メモリ | `%Q` | 優先度 |

### 1.10 優先度とフェアシェア

```bash
sprio -u $USER                    # 待機中ジョブの優先度の内訳
sshare -U                         # 自分のフェアシェア（RawUsage, FairShare）
scontrol top <id>                 # 自分の待機中ジョブの中で先頭に回す
```

このクラスタは `PriorityWeight*` がすべて 0 なので、sprio / sshare の値は順番にほとんど影響しない。待ち時間を決めるのは空きリソース、PriorityTier、backfill（短いジョブを隙間に入れる）の3つ。`--time` を実態に近い値にすると backfill で早く入りやすい。

### 1.11 環境変数の伝播

| 指定 | 動作 |
|:---|:---|
| `--export=ALL`（既定） | 投入時のシェルの環境変数をすべて渡す |
| `--export=NONE` | 何も渡さない。ジョブ内では最小限の環境になる |
| `--export=ALL,SEED=1` | 全部 + 追加 |
| `--export=SEED=1,CFG=a.yaml` | `ALL` を書かないと列挙したものと SLURM_* だけになる |

- 環境は**投入した時点**のものがコピーされる。投入後に `.bashrc` を変えても待機中のジョブには効かない
- バッチスクリプトは非対話シェルなので、`~/.bashrc` の対話シェル向けの部分は実行されない

ジョブ内で使える追加の変数:

| 変数 | 内容 |
|:---|:---|
| `SLURM_NTASKS` / `SLURM_NNODES` | タスク数 / ノード数 |
| `SLURM_PROCID` / `SLURM_LOCALID` / `SLURM_NODEID` | 全体でのタスク番号 / ノード内の番号 / ノード番号（srun の中） |
| `SLURM_GPUS_ON_NODE` | このノードで割り当てられた GPU 数 |
| `SLURM_JOB_GPUS` / `SLURM_STEP_GPUS` | 割り当てられた GPU の番号 |
| `SLURM_SUBMIT_HOST` | 投入したホスト |
| `SLURM_JOB_PARTITION` | 実行中のパーティション |
| `SLURM_RESTART_COUNT` | 再キューの回数 |

### 1.12 マルチGPU（torchrun）

1ノード内の複数GPU:

```bash
#SBATCH --nodes=1
#SBATCH --gpus=4
#SBATCH --cpus-per-task=16

srun torchrun --standalone --nproc_per_node="${SLURM_GPUS_ON_NODE}" train.py
```

複数ノード（参考。このクラスタのパーティションは各1ノードなので通常は使わない）:

```bash
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --gpus-per-node=4

export MASTER_ADDR=$(scontrol show hostnames "${SLURM_JOB_NODELIST}" | head -n1)
export MASTER_PORT=29500
srun torchrun --nnodes="${SLURM_NNODES}" --nproc_per_node=4 \
    --rdzv_backend=c10d --rdzv_id="${SLURM_JOB_ID}" \
    --rdzv_endpoint="${MASTER_ADDR}:${MASTER_PORT}" train.py
```

- `DataLoader(num_workers=...)` は GPU あたりのプロセスごとに起動される。`-c` は「GPU数 × num_workers」を目安にする
- `OMP_NUM_THREADS` を明示しないと、各プロセスが全コア分のスレッドを作って遅くなる（2.14節）

### 1.13 CPU・GPU の割り当ての確認

```bash
scontrol show job <id> -d | grep -E "CPU_IDs|GRES"   # 例: CPU_IDs=0-3 Mem=16384 GRES=gpu:1(IDX:0)
srun --cpu-bind=verbose,cores python a.py            # どのコアに束縛されたかを出力する
nvidia-smi topo -m                                   # GPU と CPU の接続関係
```

- `GRES=gpu:1(IDX:2)` の `IDX` がノード上の物理 GPU 番号。コンテナ内の `CUDA_VISIBLE_DEVICES` は 0 から振り直される
- `--gpu-bind` / `--cpu-bind` の詳細は `srun --cpu-bind=help` で確認する

### 1.14 scontrol のその他の操作

```bash
scontrol show hostnames "node[01-03]"     # ノード名の範囲表記を展開
scontrol show reservation                  # 予約枠（メンテ予定の確認にも使う）
scontrol write batch_script <id> -         # 投入されたスクリプトの中身を表示（- で標準出力）
scontrol listpids <id>                     # ジョブのプロセス一覧（そのノード上で実行）
scontrol update JobId=<id> Partition=large-andre01 TimeLimit=3:00:00   # 待機中なら変更できる
scontrol update JobId=<id> NumCPUs=8 MinMemoryNode=32G                 # 待機中のリソース変更
scontrol uhold <id> / scontrol release <id>
```

## Part 2. トラブル対応

### 2.1 ずっと PD のまま動かない

- **症状**: `squeue` の ST が PD のまま何時間も変わらない
- **確認**:
```bash
squeue --me -t PD -o "%.10i %.18P %.30j %.24R %.19S"
scontrol show job <id> | grep -E "Reason|TimeLimit|NumCPUs|MinMemory|TRES|Partition|Dependency"
sinfo -N -O "NodeList:12,Partition:20,StateCompact:8,CPUsState:16,Memory:10,AllocMem:10,GresUsed:24" -p <partition>
squeue -w <node>                         # そのノードを使っているジョブ
```
- **原因**: REASON 列で切り分ける

| REASON | 原因 |
|:---|:---|
| `Resources` | CPU・メモリ・GPU のどれかが足りない。GPU が空いていてもメモリや CPU が埋まっていれば入れない |
| `Priority` | 同じノードを待つ、より優先されるジョブ（PriorityTier が高いパーティションなど）がある |
| `PartitionTimeLimit` | `--time` がパーティションの MaxTime を超えている（2.2節） |
| `ReqNodeNotAvail` | 指定ノードが down / drain / 予約中 |
| `Dependency` | 依存先の終了待ち |
| `DependencyNeverSatisfied` | 依存先が失敗した（2.9節） |
| `JobArrayTaskLimit` | アレイの `%N` の上限に達している |
| `BeginTime` | `--begin` の時刻待ち |
| `JobHeldUser` / `JobHeldAdmin` | 保留されている |
| `AssocGrp*` / `QOSMax*` | アカウントや QOS の上限 |
| `launch failed requeued held` | 起動に失敗して保留された。`scontrol show job` の Reason と slurm.out を見る |

- **対処**:
  - `Resources`: 要求を減らす。`andre01` は 32 CPU なので、`-c 20` のジョブは同時に1本しか入らない
  - `--mem` の省略に注意。このクラスタではノードの全メモリを要求する扱いになり、他のジョブと同居できない
  - `Priority`: 待つか、空いている別パーティション・別ノードに `scontrol update JobId=<id> Partition=...` で移す
  - `JobHeld*`: `scontrol release <id>`

### 2.2 PartitionTimeLimit で入らない

- **症状**: REASON が `PartitionTimeLimit`。投入時にエラーは出ていない
- **確認**:
```bash
scontrol show job <id> | grep -E "TimeLimit|Partition"
sinfo -o "%.20P %.10l" | grep <owner>
```
- **原因**: `EnforcePartLimits=NO` のため、MaxTime を超える `--time` でも投入が受け付けられ、永久に PD になる。例: `small-*`（1時間）に `--time=2:00:00`
- **対処**:
```bash
scontrol update JobId=<id> TimeLimit=1:00:00            # 時間を短くする
scontrol update JobId=<id> Partition=medium-andre01     # またはパーティションを変える
```

### 2.3 投入直後に終了し、ログがない・空

- **症状**: 状態が FAILED で Elapsed が数秒。`--output` のファイルがない
- **確認**:
```bash
sacct -j <id> -X --format=JobID,State,ExitCode,Elapsed,WorkDir%60,NodeList
scontrol show job <id> | grep -E "StdOut|StdErr|WorkDir"    # 終了後5分以内なら見える
ls -ld "$(dirname <StdOut のパス>)"
```
- **原因**:
  - `--output` / `--error` のディレクトリが存在しない（ファイルは作られるがディレクトリは作られない）
  - ディレクトリに書き込み権限がない。計算ノードからそのファイルシステムが見えない
  - `--chdir` のパスが計算ノードに存在しない
  - スクリプトの改行コードが CRLF（Windows で編集）で、`#!/bin/bash^M` になっている
- **対処**:
  - `mkdir -p logs` を投入前に行う。`--output` は絶対パスにする
  - `file job.sh` で `CRLF` と出たら `sed -i 's/\r$//' job.sh`
  - 原因が分からないときは `--output=/workspace/.../logs/debug_%j.out` のように確実に書ける場所に出して再投入する

### 2.4 OUT_OF_MEMORY になる（MaxRSS は小さいのに）

- **症状**: 状態 `OUT_OF_MEMORY`、slurm.out に `oom_kill event` や `Killed`。sacct の MaxRSS は `--mem` より小さい
- **確認**:
```bash
sacct -j <id> --format=JobID,State,ExitCode,ReqMem,MaxRSS,MaxVMSize --units=G
seff <id>                                          # 入っていれば
grep -iE "oom|killed|memory" logs/<exp>/<job>/slurm.out | tail
df -h /dev/shm                                     # ジョブ内で実行して共有メモリの使用量を見る
```
- **原因**:
  - MaxRSS は定期サンプリング（既定30秒）の最大値。短時間のピークは記録されない
  - cgroup は RSS だけでなく、ページキャッシュや `/dev/shm`（tmpfs）の使用量もジョブのメモリとして数える。PyTorch の DataLoader はワーカー間の受け渡しに共有メモリを使う
  - 子プロセス（DataLoader のワーカー、並列処理）の分も合算される
- **対処**:
  - `--mem` を実使用量の1.5〜2倍にする
  - `num_workers` や `prefetch_factor`、バッチサイズを減らす
  - 大きい配列は一度に読まず、memmap やチャンク処理にする
  - ピークを知りたいときは `sstat` を短い間隔で回すか、Python 側で `resource.getrusage` を出力する

### 2.5 TIMEOUT で後処理まで届かない

- **症状**: 状態 `TIMEOUT`。最後のチェックポイントや結果ファイルが保存されていない
- **確認**:
```bash
sacct -j <id> -X --format=JobID,State,ExitCode,Elapsed,Timelimit
tail -n 50 <slurm.out>
```
- **原因**: 時間切れで SIGTERM が送られ、`KillWait`（30秒）後に SIGKILL される。`OverTimeLimit=0` なので延長はない。30秒で終わらない保存処理は途中で切られる
- **対処**:
  - `--signal=B:USR1@600` などで10分前に通知させ、そこでチェックポイントを保存して終了する（1.3節）
  - 定期的にチェックポイントを保存し、`--resume` で途中から再開できるようにする
  - 待機中なら `scontrol update JobId=<id> TimeLimit=...` で延ばす。実行中の延長は管理者権限が必要なことが多い

### 2.6 NODE_FAIL / 勝手に再実行された

- **症状**: 状態 `NODE_FAIL`、または同じジョブIDが最初から走り直し、ログが上書きされた
- **確認**:
```bash
sacct -j <id> -D --format=JobID,State,Start,End,NodeList    # 再キュー前の記録も出る
scontrol show node <node> | grep -E "State|Reason"
sinfo -R
```
- **原因**: ノード障害や通信断。`JobRequeue=1` なので、バッチジョブは自動で再キューされることがある。既定の `--open-mode=truncate` だと再実行時にログが消える
- **対処**:
  - 途中から再開できないジョブは `--no-requeue` を付ける
  - 再開できるジョブは `--open-mode=append` にして、`SLURM_RESTART_COUNT` をログに出す
  - ノードが drain のままなら管理者に連絡し、`-x <node>` で除外して再投入する

### 2.7 GPU が見えない・想定と違う GPU を使っている

- **症状**: `torch.cuda.is_available()` が False、または `nvidia-smi` に他人のプロセスが見える / 使う GPU 番号がずれる
- **確認**:
```bash
scontrol show job <id> -d | grep -E "GRES|TresPerNode"     # IDX で物理番号がわかる
srun --jobid=<id> --overlap bash -c 'echo $CUDA_VISIBLE_DEVICES; nvidia-smi -L'
```
- **原因**:
  - `--gres=gpu:N` / `--gpus` の指定漏れ（指定しないと GPU は割り当てられない）
  - Apptainer で `--nv` を付けていない
  - コード内で `cuda:1` など番号を直書きしている。割り当て後は見える GPU が 0 から振り直される
  - `CUDA_VISIBLE_DEVICES` をスクリプト内で上書きしている
- **対処**: GPU 指定を追加する。コードでは `cuda:0` または `torch.device("cuda")` を使い、`CUDA_VISIBLE_DEVICES` を自分で書き換えない

### 2.8 CG（COMPLETING）から抜けない

- **症状**: `scancel` した、または終了したのに CG のまま数分以上残る。後続ジョブがそのノードに入らない
- **確認**:
```bash
squeue --me -t CG
scontrol show job <id> | grep -E "JobState|Reason|NodeList"
sinfo -R | grep <node>
```
- **原因**: プロセスが終了できない。多くは NFS の I/O 待ち（D 状態）。`UnkillableStepTimeout`（60秒）を超えるとノードが drain（`Kill task failed`）になる
- **対処**: ユーザー側でできることは少ない。数分待ち、残るようなら管理者に連絡する。再投入するときは `-x <node>` でそのノードを避ける

### 2.9 DependencyNeverSatisfied のまま残る

- **症状**: REASON が `DependencyNeverSatisfied` で永久に PD
- **確認**:
```bash
scontrol show job <id> | grep -E "Dependency|Reason"
sacct -j <依存先id> -X --format=JobID,State,ExitCode
```
- **原因**: `afterok` の依存先が FAILED / TIMEOUT / CANCELLED で終わった。アレイに `afterok` した場合は1タスクでも失敗すると成立しない
- **対処**:
```bash
scancel <id>                                           # 取り消して投げ直す
scontrol update JobId=<id> Dependency=afterok:<新しいid>   # 依存先を再実行したなら付け替える
sbatch -d afterok:<id> --kill-on-invalid-dep=yes job.sh    # 今後は自動で取り消させる
```

### 2.10 ジョブ内の環境がログインシェルと違う

- **症状**: 手元では動くのに、ジョブ内で `command not found`、別の Python が使われる、環境変数が空
- **確認**:
```bash
sbatch --wrap='env | sort > env_job.txt; which python; echo $PATH' -p small-andre01 -t 5
env | sort > env_login.txt; diff env_login.txt env_job.txt
scontrol show job <id> | grep -i export
```
- **原因**:
  - バッチスクリプトは非対話シェルで、`~/.bashrc` の対話向け部分（`[[ $- == *i* ]]` の中）が実行されない
  - `--export=NONE` やスクリプト内の `--export` で環境が絞られている
  - 投入時点の環境がコピーされるため、投入時に activate していた venv や一時的な export がそのまま持ち込まれる
- **対処**: ジョブに必要な環境変数とパスはスクリプト内で明示的に設定する。`--export=ALL` に頼らない

### 2.11 並列に起動した srun が待たされる

- **症状**: `srun: Job <id> step creation temporarily disabled, retrying (Requested nodes are busy)` と出て、ステップが順番にしか動かない
- **確認**:
```bash
squeue -s -j <id>                       # 実行中のステップ
scontrol show job <id> | grep -E "NumCPUs|TRES"
```
- **原因**: 最初のステップがジョブ全体の CPU・メモリ・GPU を使っていて、次のステップに割り当てる分がない
- **対処**: 各 `srun` に `--ntasks=1 --cpus-per-task=... --gpus=... --mem=...` と `--exact` を付け、合計がジョブの確保量を超えないようにする。監視用など資源を共有してよいステップは `--overlap`

### 2.12 アレイの一部のタスクだけ失敗した

- **症状**: 大半は COMPLETED だが、いくつかが FAILED / OUT_OF_MEMORY
- **確認**:
```bash
sacct -j <array_id> -X --format=JobID%16,State,ExitCode,Elapsed,MaxRSS,NodeList
sacct -j <array_id> -X -n -P --format=JobID,State | grep -v COMPLETED
```
- **原因**: 入力データによってメモリや時間が大きく違う、特定ノードの障害、インデックスと設定ファイルの対応のずれ（`sed -n "$((ID+1))p"` の +1 忘れなど）
- **対処**: 失敗したインデックスだけ `sbatch --array=3,7,12` で投げ直す（1.6節）。ノードが偏っているなら `-x` で除外する

### 2.13 ジョブが squeue からも sacct からも見つからない

- **症状**: 終わったはずのジョブを `squeue` でも `sacct -j` でも見つけられない
- **確認**:
```bash
sacct -u $USER -X -S now-7days --format=JobID,JobName%30,State,End | grep <name>
sacct -j <id> -S 2026-01-01
```
- **原因**: `squeue` / `scontrol show job` は終了後 `MinJobAge`（300秒）で消える。`sacct` は `-S` を省略すると当日0時以降しか探さない
- **対処**: `sacct` には必ず `-S` を付ける

### 2.14 CPU を増やしたのに遅い・負荷が異常に高い

- **症状**: `-c` を増やしても速くならない。`top` で load average が割当 CPU 数を大きく超えている
- **確認**:
```bash
srun --jobid=<id> --overlap bash -c 'nproc; echo OMP=$OMP_NUM_THREADS; ps -eLf | grep -c python'
sstat -j <id>.batch --format=JobID,AveCPU,MaxRSS
```
- **原因**: numpy / torch / OpenMP が物理コア数ぶんのスレッドを作り、DataLoader のワーカー数と掛け算になる。cgroup で割り当てたコアに押し込まれて取り合いになる
- **対処**:
```bash
export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
```
DataLoader 内の処理が並列化されている場合は `OMP_NUM_THREADS=1` にして、並列度は `num_workers` で調整する
