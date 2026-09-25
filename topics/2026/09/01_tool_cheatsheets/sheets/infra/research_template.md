# 内製実験テンプレート（template-daily-research-experiments）チートシート

> 対象: `template-daily-research-experiments` 系のプロジェクト。2026-09-25 時点の実物をもとに書いた
> 元ドキュメント（各プロジェクト直下）: `README.md`（セットアップ）、`USAGE.md`（運用手順）、`FUNCTIONS.md`（関数一覧）、`TEMPLATE_CONCEPT.md`（設計思想）

## 0. このマシンでの使用状況

| プロジェクト | 版 | 実験数 | 備考 |
|:---|:---|---:|:---|
| `02-playground/toxpatho-ssl-comparison` | 現行版 | 27 | `make test` / `make shellcheck` があり、`make review` はない |
| `02-playground/toxpatho-patch-selector` | 現行版 | 9 | |
| `02-playground/toxpatho-perturbation-bench` | 現行版 | 6 | 本シートの主な参照元 |
| `01-toxpatho/patch-ad-cl-selector` | 現行版 | 5 | |
| `01-toxpatho/wsi-ad` | 旧版 | 0 | `make make_sif` があり、`preflight` / `rename_exp` / `log_clean` はない。GRID 方式もない |
| `01-toxpatho/vq-patho-anomaly-detection` | 非テンプレート | — | Makefile なし |

- `scripts/slurm_entry.sh` は現行版の4プロジェクトで中身が少しずつ違う。プロジェクトをまたいでコピーするときは差分を確認する
- **`~/.bashrc` にはシェルヘルパー（`runx` / `cdx` など）の読み込み設定が入っていない。** 使うには第1節の手順3が必要

## 1. セットアップ（プロジェクトごとに1回）

```bash
make setup                        # logs/ data/ outputs/ experiments/ などを作る
export SIF_PATH="$(pwd)/env.sif"  # make uv_sync / make jupyter の前に必要
make uv_sync p=<partition>        # Python 環境の同期を sbatch で実行する
```

- `SIF_PATH` は各実験の `run_slurm.sh` の中でしか定義されていない。`tools/uv_sync.sh` と `tools/start_jupyter.sh` は投入元のシェルの環境変数を使うので、export していないとコンテナを起動できない

シェルヘルパーを使えるようにする（`~/.bashrc` に1行追加）:

```bash
repo_root="$(git rev-parse --show-toplevel)"
loader_line="source \"${repo_root}/shell/template-daily.sh\""
grep -Fqx "${loader_line}" ~/.bashrc || printf '\n%s\n' "${loader_line}" >> ~/.bashrc
source ~/.bashrc
```

- 読み込まれるのは `.bashrc.d/0-bootstrap.sh` と `1-experiments.sh` の2つだけ。`2-git.sh`（`gstart` / `gpush`）は Git のグローバル設定を変えるので、使う場合は別途 source する
- Slack 通知を使う場合: `export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."`

## 2. 大原則

| 原則 | 意味 |
|:---|:---|
| 1実験 = 1フェーズ | 前処理・学習・評価は別の実験ディレクトリに分ける |
| 1ジョブ = 1組み合わせ | `experiment.py` の中で seed やモデルをループしない。複数の組み合わせは `run_slurm.sh` の GRID で展開する |
| 設定は config.yml | `experiment.py` にハイパーパラメータを直書きしない |
| コミットしてから投入 | `runx` は未コミットの変更があると投入を拒否する |

## 3. 1実験の流れ

```mermaid
graph LR
    A["make create_exp name=..."] --> B["experiment.py / config.yml / run_slurm.sh を編集"]
    B --> C["make preflight"]
    C --> D["git commit"]
    D --> E["runx"]
    E --> F["logs/ と outputs/ を確認"]
    F -->|失敗| G["make mark_fail / make resume_exp"]
```

```bash
make create_exp name=train_ssl_dino   # → experiments/0028_20260925_train_ssl_dino/
cdx                                   # 最新の実験ディレクトリへ移動
# experiment.py / config.yml / run_slurm.sh を編集
make preflight                        # 静的チェック
git add -A && git commit -m "..."
runx                                  # 最新の実験を投入（runx 28 で ID 指定）
```

## 4. ディレクトリ構成

```
experiments/{ID}_{YYYYMMDD}_{name}/
    experiment.py            # 実験ロジック本体
    config.yml               # 固定ハイパーパラメータ
    run_slurm.sh             # #SBATCH ヘッダ + 実行コマンド + GRID 定義
    metadata.yaml            # 自動生成（exp_id, created_date, git_commit）
    uncommitted_changes.diff # 作成時に未コミットの変更があった場合だけ
experiments/latest -> 最新の実験

outputs/{実験名}/{variant_key}/completion.json   # 実行状態（running / completed）
outputs/{実験名}/latest_job_id.txt               # 最後に投入したジョブID

logs/{実験名}/{job_id}/
    command.sh               # 実際に実行されたコマンド
    run_metadata.yaml        # status, fail_reason, ノード, git_commit
    slurm.out                # 標準出力と標準エラー
```

- 実験名の付け方は `{動詞}_{対象}`（例: `train_mamba`, `eval_uni_linear`）。`test_xxx` / `new_xxx` / `retry_xxx` / `experiment001` は避ける
- 実験固有の注意事項は、実験ディレクトリにファイルを増やさず、リポジトリ直下の `EXPERIMENT_NOTES.md` に `## {実験名}` の見出しで追記する

## 5. ファイルの役割分担

| ファイル | 書くもの | 書かないもの |
|:---|:---|:---|
| `config.yml` | 探索しない固定値（lr, epochs, データパスなど） | 探索する値 |
| `run_slurm.sh` | リソース（`#SBATCH`）、実行モード、探索する次元（`GRID_ARGS` / `GRID_VALUES`） | ハイパーパラメータの固定値 |
| `experiment.py` | ロジック。探索する次元は argparse で受け取る | 値の直書き、`outputs/` へのパス直書き |

## 6. run_slurm.sh の書き方

作成時に次のプレースホルダが実際の値に置き換えられる: `__EXP_NAME__`, `__PROJECT_ROOT__`, `__PARTITION__`, `__SIGNAL_MARGIN__`。
**`--partition` と `--signal` は作成時の `--time` から一度だけ計算される。** 後で `--time` や GPU 数を大きく変えたら、この2つも手で直す。

### 実行モード（3種類）

| モード | 動き | 必要な設定 |
|:---|:---|:---|
| `single`（デフォルト） | 1回だけ実行 | `RUN_COMMAND` |
| `array` | GRID の組み合わせごとに別ジョブ（並列） | `BASE_COMMAND`, `GRID_ARGS`, `GRID_VALUES`, `#SBATCH --array=0-N`, 出力名を `%A_%a` に |
| `seq` | 1ジョブの中で GRID を順番に実行 | `BASE_COMMAND`, `GRID_ARGS`, `GRID_VALUES`（`--array` は不要） |

```bash
# array の例: 2モデル × 3シード = 6通り → --array=0-5
#SBATCH --output=__PROJECT_ROOT__/logs/__EXP_NAME__/%A_%a___EXP_NAME__.out
#SBATCH --error=__PROJECT_ROOT__/logs/__EXP_NAME__/%A_%a___EXP_NAME__.out
#SBATCH --array=0-5

RUN_MODE="array"
BASE_COMMAND="python ${PYTHON_PATH}"
GRID_ARGS=(
    "--model"
    "--seed"
)
GRID_VALUES=(
    "uni2 dinov2"
    "0 1 2"
)
```

- `GRID_ARGS[i]` と `GRID_VALUES[i]` が対応し、全組み合わせ（直積）が展開される
- `--array=0-N` の N は「組み合わせ数 − 1」。`make preflight` で一致を確認できる
- 組み合わせは最大50まで（preflight で警告される）

### そのほかの設定項目

| 項目 | 既定値 | 意味 |
|:---|:---|:---|
| `SIF_PATH` | `${PROJECT_ROOT}/env.sif` | 使う Apptainer イメージ |
| `USE_LOCAL_SSD_INPUT` | `1` | `data/` をノードのローカル SSD（`/scratch`）にコピーしてから読む |
| `USE_LOCAL_SSD_OUTPUT` | `1` | 出力をローカル SSD に書き、終了時に `outputs/` へ rsync で回収する |
| `#SBATCH --dependency` | コメントアウト | 他の実験の後に実行したいとき。job_id は `outputs/{依存先}/latest_job_id.txt` を見て手で書く |

- 実行中の出力を `/workspace` 側からリアルタイムに見たいときだけ `USE_LOCAL_SSD_OUTPUT=0` にする
- `/scratch/${USER}/${EXP_NAME}_${JOB_ID}` は**自動では削除されない**。`slurm.out` に警告が出るので、手で消す

## 7. experiment.py での出力（lib/output_utils.py）

```python
from lib.output_utils import get_run_dir, write_run_metadata, complete_run

variant_key = f"{args.model}_seed{args.seed}"                 # 結果に影響する次元をすべて入れる
run_dir = get_run_dir(project_root, __file__, variant_key)    # 完了済みならここで exit(0)
write_run_metadata(run_dir, model=args.model, seed=args.seed) # status=running
# ... 処理。結果はメモリに溜めず、少しずつ書き出す ...
complete_run(run_dir)                                         # status=completed
```

- `completion.json` が `completed` の variant は再実行されない。やり直したいときはそのディレクトリを手で消す
- `variant_key` に `/` を含めると、最後の要素だけが使われる（`google/gemma` → `gemma`）
- `PROJECT_ROOT` 環境変数は `run_slurm.sh` が設定する。`python experiment.py` を直接実行するとエラーになる

## 8. コマンド一覧

### make ターゲット

| コマンド | 内容 |
|:---|:---|
| `make setup` | 初期ディレクトリを作る |
| `make uv_sync p=<partition>` | uv の同期を sbatch で実行する |
| `make create_exp name=<name>` | 実験を作る（ID は自動採番。Git 操作はしない） |
| `make preflight` | 最新の実験を静的チェックする（config キーの不整合、GRID の数、`--array` の N、パスの直書き） |
| `make rename_exp name=<id> new=<new>` | 名前を変える |
| `make mark_fail name=<id> reason=<r>` | `experiments/` と `outputs/` を `_FAILED_{date}_{reason}` 付きにリネームする |
| `make resume_exp name=<id> [suffix=<s>]` | 新しい ID でコピーして作り直す（既定の suffix は `resumed`）。`--dependency` は無効化される |
| `make review exp=<id>` | 実験のレビュー（ssl-comparison にはない） |
| `make log_clean` | `logs/` 直下の `.out` を `{job_id}/slurm.out` に整理する |
| `make clean_failed` | FAILED / TIMEOUT / OOM / CANCELLED のログ、24時間以上更新のない RUNNING のログ、実験が存在しない孤立ログを**削除**する |
| `make jupyter p=<partition> mem=<mem>` | 計算ノードで Jupyter を起動する（例: `p=a6000 mem=32g`） |

### シェル関数（`shell/template-daily.sh` を source した場合）

| コマンド | 内容 |
|:---|:---|
| `groot` | リポジトリのルートへ移動 |
| `lsx` | 最新10件の実験を一覧表示 |
| `cdx [id]` | 実験ディレクトリへ移動（省略すると最新） |
| `runx [id] [--allow-dirty]` | `run_slurm.sh` を sbatch で投入する。`logs/{exp}/` も作る |
| `cancelx <job_id> [reason]` | `run_metadata.yaml` を CANCELLED にして Slack に通知し、scancel する |

### Git（`2-git.sh`、個人設定）

| コマンド | 内容 |
|:---|:---|
| `gstart` | main を pull して `daily/YYMMDD` ブランチを作る（朝） |
| `gpush` | `daily/YYMMDD` を push し、main に `--no-ff` でマージして push する（夜） |
| `gpush -gh` | ローカルでマージする代わりに PR を作る（要 `gh auth login`） |

## 9. 失敗したときの見方

確認の順番: `run_metadata.yaml` の `status` / `fail_reason` → `command.sh` → `config.yml` → `slurm.out`

```bash
cat logs/<exp>/latest/run_metadata.yaml
tail -n 100 logs/<exp>/latest/slurm.out
sacct -j <job_id> --format=JobID,State,ExitCode,Elapsed,MaxRSS
```

`run_metadata.yaml` の `status` は、ジョブの実行中に終了コードから決まるので、メモリ超過でも `FAILED`（`NONZERO_EXIT_137` など）と記録されることが多い。**本当の終了理由は `sacct` の `State` で確認する。**

| sacct の State | よくある原因 |
|:---|:---|
| `OUT_OF_MEMORY` | `--mem` が足りない（run_metadata では `FAILED` / 終了コード 137 になりやすい） |
| `TIMEOUT` | `--time` が足りない。延ばしたら `--partition` / `--signal` も見直す |
| `FAILED` | コードのエラー。`slurm.out` の末尾を見る |
| `NODE_FAIL` | ノード側の障害。再投入する |

## 10. エージェント（agentrun / capsule）から使う場合

- capsule 内からは `sbatch` / `srun` / `salloc` を実行できない（hook でブロックされる）
- エージェントは `run_slurm.sh` を `artifacts/proposed_train.sbatch` にコピーし、投入はユーザーが行う
- エージェントができる Git 操作は `daily/YYMMDD` ブランチ上での add / commit まで。checkout、ブランチ作成、merge、rebase、push はブロックされる
- 同じリポジトリで `agentrun` を同時に2つ起動することはできない（`.agentrun.lock`）
