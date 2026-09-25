# Weights & Biases (wandb) 詳細編（応用・トラブル対応）

> 基本編: [wandb.md](wandb.md)
> 対象バージョン: `wandb 0.27.0`（バックエンドは Go 製の `wandb-core`。API はこの環境の `.venv` で確認済み）

## Part 1. 応用・上級機能

### 1.1 run の再開（resume）

Slurm の時間制限・requeue・プリエンプションで落ちた run を、同じ run として続きから記録する。

```python
import os, wandb

run_id = os.environ.get("RUN_ID") or f"{os.environ['SLURM_ARRAY_JOB_ID']}_{os.environ['SLURM_ARRAY_TASK_ID']}"
run = wandb.init(project="toxpatho-ssl", id=run_id, resume="allow")
print(run.resumed, run.step)          # 再開したか、最後の step
start_epoch = load_ckpt_if_exists()   # モデル側の再開は自分で行う（PyTorch 詳細編 1.8）
```

| `resume=` | 同じ ID の run がある | ない |
|:---|:---|:---|
| `"allow"` | 続きから記録 | 新規作成 |
| `"must"` | 続きから記録 | エラー |
| `"never"` | エラー | 新規作成 |
| `"auto"` | 同じマシンで直前に落ちた run を自動で再開 | 新規作成 |

- `id` はプロジェクト内で一意にする。英数字・`-`・`_` が無難
- 再開後は `step` を最後の値より大きい値から続ける。小さい step は捨てられる（2.5）
- 環境変数なら `WANDB_RUN_ID` と `WANDB_RESUME=allow`
- 途中の step から分岐した新しい run を作るなら `fork_from=f"{run_id}?_step=1000"`、同じ run を途中から巻き戻すなら `resume_from=`（どちらも `_step` のみ対応）

### 1.2 wandb.Settings の主な項目

```python
run = wandb.init(
    project="toxpatho-ssl",
    settings=wandb.Settings(
        init_timeout=300,                  # init の待ち時間（秒、既定 90）
        x_stats_sampling_interval=30,      # システムメトリクスの取得間隔（秒、既定 15）
        x_stats_gpu_device_ids=[0],        # 監視する GPU
        x_disable_stats=False,             # True でシステムメトリクスを取らない
        console="wrap",                    # 標準出力の取り込み方法（"off" で取り込まない）
        quiet=True,                        # 端末への出力を減らす
    ),
)
```

| 項目 | 内容 |
|:---|:---|
| `init_timeout` | `wandb.init` の待ち時間。環境変数 `WANDB_INIT_TIMEOUT` |
| `finish_timeout` | 終了時のアップロードを待つ上限（既定 0 = 無制限）。超えると crashed 扱い |
| `x_service_wait` | `wandb-core` の起動を待つ時間（既定 30 秒） |
| `console` | `"auto"`, `"off"`, `"wrap"`, `"redirect"` など |
| `save_code` | 実行したスクリプトを保存する |
| `x_label` / `x_primary` / `x_update_finish_state` | 複数プロセスで1つの run に書く場合（1.4） |

### 1.3 config を argparse / YAML から入れる

```python
args = parser.parse_args()
run = wandb.init(project="toxpatho-ssl", config=vars(args))     # argparse
run.config.update({"git_commit": commit}, allow_val_change=True)

import yaml
cfg = yaml.safe_load(open("configs/dino.yaml"))
run = wandb.init(config=cfg)                                    # 入れ子の dict もそのまま入る
```

- `config="file.yaml"` と文字列で渡すと、wandb 独自の形式（`key: {value: ...}`）として読まれる。普通の YAML は自分で読んで dict を渡す
- 入れ子の config は Web 画面で `model.name` のようにドットでつながったキーになる
- sweep から起動された場合は、sweep のパラメータが `run.config` に上書きされる。コードでは必ず `run.config` から値を読む

### 1.4 DDP / 複数プロセスからの記録

基本は **rank 0 だけ** `wandb.init` する。

```python
if rank == 0:
    run = wandb.init(project="toxpatho-ssl", config=cfg)
else:
    run = wandb.init(mode="disabled")          # log を呼んでも何もしない
...
run.log({"train/loss": loss_avg})             # 集計（all_reduce）した値を rank 0 が記録
```

全 rank のシステムメトリクスやログを1つの run にまとめたいときは `mode="shared"`:

```python
run_id = broadcast_from_rank0(wandb.util.generate_id())   # 全 rank で同じ ID を共有する
run = wandb.init(
    project="toxpatho-ssl", id=run_id,
    settings=wandb.Settings(
        mode="shared",
        x_primary=(rank == 0),                # 主プロセスだけがファイル・メタデータを保存
        x_label=f"rank_{rank}",               # システムメトリクスとログを rank ごとに区別
        x_update_finish_state=(rank == 0),    # run の終了状態は主プロセスだけが決める
    ),
)
```

### 1.5 横軸を変える（define_metric）

```python
run.define_metric("epoch")
run.define_metric("val/*", step_metric="epoch")          # val/ 以下は epoch を横軸に
run.define_metric("train/*", step_metric="global_step")
run.define_metric("val/auc", summary="max")              # summary に最大値を残す
run.define_metric("val/loss", summary="min")

run.log({"train/loss": l, "global_step": gs})            # step= は指定しない
run.log({"val/auc": auc, "epoch": ep})
```

- `step_metric` を設定したメトリクスは、ログにその値がなければ最後の値が自動で補われる（`step_sync`、既定 True）
- `summary` は `"min"`, `"max"`, `"mean"`, `"last"`, `"first"`, `"none"`
- 同じ step にまとめて書きたい値は `run.log({...}, commit=False)` で溜めて、最後の `run.log` で確定する

### 1.6 アラート

```python
if not math.isfinite(loss):
    run.alert(title="NaN loss", text=f"step={step} lr={lr}", level="ERROR", wait_duration=600)
```

- `level` は `"INFO"`, `"WARN"`, `"ERROR"`。同じ title は `wait_duration`（既定 1 分）以内は再送しない
- 通知先（メール / Slack）は Web のユーザー設定で有効にする。オフラインモードでは届かない

### 1.7 アレイジョブ・requeue での整理

```bash
#SBATCH --array=0-4
export WANDB_RUN_GROUP="dino_${SLURM_ARRAY_JOB_ID}"
export WANDB_NAME="seed${SLURM_ARRAY_TASK_ID}"
export WANDB_RUN_ID="${SLURM_ARRAY_JOB_ID}_${SLURM_ARRAY_TASK_ID}"
export WANDB_RESUME=allow            # requeue されても同じ run に続けて書く
export WANDB_JOB_TYPE=train
```

```python
run = wandb.init(project="toxpatho-ssl", config=cfg)   # 上の環境変数が使われる
run.config.update({"slurm_job_id": os.environ.get("SLURM_JOB_ID")})
```

時間切れで落ちそうなときは `run.mark_preempting()` を呼ぶと、Web 上で「preempted」として区別される。

### 1.8 Sweep を Slurm で回す

```bash
wandb sweep --project toxpatho-ssl sweep.yaml       # ログインノードで1回。sweep ID を控える
```

```bash
#SBATCH --array=0-19%4                              # 20 run、同時に 4 つ
wandb agent --count 1 entity/toxpatho-ssl/<sweep_id>   # 各タスクが 1 run だけ実行して終わる
```

| 操作 | コマンド |
|:---|:---|
| 一時停止 / 再開 | `wandb sweep --pause <id>` / `--resume <id>` |
| 止める（実行中は続ける） | `wandb sweep --stop <id>` |
| 中止（実行中も止める） | `wandb sweep --cancel <id>` |
| 設定を変える | `wandb sweep --update <id> sweep.yaml` |

- agent は sweep の制御のためにサーバーと通信する。計算ノードが外に出られない環境では使えない。その場合はパラメータの組み合わせをアレイジョブの番号から自分で作り、offline で記録する
- `--count` を付けないと、agent は sweep が終わるまで次々に run を実行する。ジョブの時間制限に注意する
- `-f`（`--forward-signals`）で SIGTERM を子プロセスに渡すと、`scancel` 時にきれいに終われる

### 1.9 Artifact の応用

```python
art = wandb.Artifact("uni-features", type="dataset", metadata={"model": "UNI", "mag": "20x"})
art.add_reference("file:///data/share/features/uni_20x")   # 実体はアップロードせず、パスとハッシュだけ記録
run.log_artifact(art, aliases=["latest", "20x"])

best = wandb.Artifact("dino-vits", type="model")
best.add_file("ckpt/best.pt")
run.log_artifact(best, aliases=["best"]).wait()             # アップロード完了まで待つ
```

| 機能 | 使い方 |
|:---|:---|
| 参照（大きなファイル） | `add_reference("file://...")`（`s3://`、`gs://` も可）。共有ディスク上のデータに向く |
| エイリアス | `latest`, `best`, `v3` など。`use_artifact("dino-vits:best")` |
| 系譜（lineage） | `use_artifact` した run と `log_artifact` した run が Web 上でつながる |
| 由来の確認 | `art.logged_by()`（作った run）、`art.used_by()`（使った run） |
| TTL（自動削除） | `art.ttl = timedelta(days=30)`（ログ前に設定） |
| 検証 | `art.verify(root)` でダウンロード済みファイルのハッシュを確認 |
| 部分ダウンロード | `art.download(path_prefix="fold0/")` |

- 参照 artifact の `download()` は元のパスから取得するので、そのパスが見えるマシンでしか使えない
- 患者データなど外部に出せないものは、`add_file` / `add_dir` ではなく `add_reference` か、そもそも artifact にしない

### 1.10 Api の応用（集計・後から修正）

```python
import pandas as pd
api = wandb.Api(timeout=60)

runs = api.runs("entity/toxpatho-ssl",
                filters={"config.model": "vit_s", "state": "finished", "tags": {"$in": ["ssl"]}},
                order="-summary_metrics.val/auc")
df = pd.DataFrame([{"name": r.name, "id": r.id, **r.config, **dict(r.summary)} for r in runs])

r = api.run("entity/toxpatho-ssl/<run_id>")
hist = r.history(keys=["val/auc", "epoch"], samples=500)          # 間引かれる（最大 samples 行）
rows = list(r.scan_history(keys=["val/auc", "epoch"]))            # 全行（遅い）
sys = r.history(stream="system")                                  # システムメトリクス

r.config["note"] = "rerun with fixed seed"; r.update()            # config を後から修正
r.summary["best_val_auc"] = 0.91; r.summary.update()              # summary を後から修正
r.tags.append("paper"); r.update()

for a in r.logged_artifacts():
    print(a.name, a.aliases)
```

- `history` は既定で 500 点に間引く。論文の図の元データは `scan_history` で取る
- `api.runs` は既定で `lazy=True`。config や summary は最初にアクセスしたときに取得される
- 削除は `r.delete()`（`delete_artifacts=True` で紐づく artifact も）。元に戻せない

### 1.11 オフライン同期の詳細

```bash
wandb sync --sync-all                               # 未同期の run をすべて
wandb sync --include-offline --exclude-globs "*debug*" logs/wandb/
wandb sync --id <run_id> logs/wandb/offline-run-2026...   # 既存の run ID に上げる
wandb sync --append --id <run_id> ...                     # 既存 run に追記
wandb sync --clean --clean-old-hours 24                   # 同期済みで 24 時間以上前のローカルデータを削除
wandb sync --skip-console ...                             # 標準出力のログを上げない
```

- 同期済みの run には `run-*.wandb.synced` の目印ファイルができ、`--sync-all` の対象から外れる
- オフライン run の ID は `offline-run-<日時>-<ID>` のディレクトリ名の末尾。同期するとこの ID の run になる

### 1.12 ログの頻度と重さ

| 対象 | 目安 |
|:---|:---|
| スカラー | 1〜数十 step に1回で十分。1 run あたり数万 step までは問題ない |
| 画像 | エポックごとに数枚〜数十枚。大きな画像は縮小してから |
| `wandb.Table` | 行数が多いと重い。数千行程度に抑える |
| `run.watch` | `log_freq` を大きく（既定 1000）。全パラメータの記録は重い |
| ヒストグラム | `wandb.Histogram` は数を絞る |

### 1.13 複数ワーカーで wandb-core を共有する（beta）

1ノードで多数の独立プロセスが `wandb.init` する場合、各プロセスが `wandb-core` を起動するので CPU とメモリを食う。

```bash
export WANDB_SERVICE="$(wandb beta core start)"     # 1つだけ起動し、各プロセスから接続
python launch_many_workers.py
# 接続がない状態が 10 分続くと自動で終了する（start の --idle-timeout で変更、例: 30m）
```

beta 機能なので、仕様が変わる可能性がある。

## Part 2. トラブル対応

### 2.1 wandb.init が止まる / タイムアウトする

- **症状**: `wandb.init` で数十秒〜止まり、`CommError: Run initialization has timed out after 90.0 sec.` などで落ちる
- **確認**:
```bash
curl -sI https://api.wandb.ai | head -1              # ノードから到達できるか
wandb status
env | grep -E "^WANDB_|_PROXY" 
ls logs/wandb/latest-run/logs/                       # debug.log / debug-internal.log
```
- **原因**: 計算ノードから外部に出られない、プロキシ設定がない、サーバーが一時的に遅い、多数のジョブが同時に init している
- **対処**: 外に出られないなら `WANDB_MODE=offline` にして後で `wandb sync`。遅いだけなら `WANDB_INIT_TIMEOUT=300`。プロキシが必要なら `HTTPS_PROXY` を設定する。学習自体を止めたくない場合は init を try で囲み、失敗したら `mode="disabled"` で続行する

### 2.2 wandb-core が起動しない / service のエラー

- **症状**: `ServicePollForTokenError: wandb-core exited with code ...`、`Failed to read port info after 30 seconds`、`Failed to start the wandb service`
- **確認**:
```bash
python -c "import wandb, os; print(os.path.dirname(wandb.__file__))"
ls -l "$(python -c 'import wandb,os;print(os.path.dirname(wandb.__file__))')/bin/wandb-core"
python -c "import tempfile; print(tempfile.gettempdir())"; df -h /tmp; echo $TMPDIR
WANDB_CORE_DEBUG=true python train.py                # wandb-core の詳細ログ
```
- **原因**: `wandb-core` はポート情報を一時ディレクトリに書くので、`$TMPDIR` / `/tmp` が書き込めない・いっぱいだと失敗する。バイナリに実行権限がない（noexec のマウントに venv を置いている）。ノードが高負荷で 30 秒以内に起動しない
- **対処**: 書き込める `TMPDIR` を設定する（Apptainer では `--writable-tmpfs` や `-B` でバインドしたディレクトリ）。負荷が高いなら `wandb.Settings(x_service_wait=120)`。venv を noexec のファイルシステムに置いていないか確認する

### 2.3 オフライン run の sync に失敗する

- **症状**: `wandb sync` で `Error while calling W&B API: ... 409 Conflict`、`run ... already exists`、途中で止まる、データの一部だけ上がる
- **確認**:
```bash
wandb sync --sync-all --show 20                      # 状態の一覧
ls -la logs/wandb/offline-run-*/                     # run-*.wandb があるか、サイズ
ls logs/wandb/offline-run-*/*.wandb.synced 2>/dev/null   # 同期済みの目印
```
- **原因**: 同じ ID の run がすでにオンラインに存在する（再実行で同じ ID を使った）、ジョブが強制終了されて `.wandb` ファイルの末尾が壊れている、ネットワークの断続
- **対処**: 別の run として上げるなら `wandb sync --id <新しいID> <dir>`、既存 run に追記するなら `--append --id <ID>`。壊れたファイルでも読める部分までは同期される。再実行するときは `--include-synced` を付けない限り二重に上がらない

### 2.4 ディスクがいっぱいになる

- **症状**: `No space left on device`、ホームのクオータ超過。`~/.cache/wandb` や `wandb/` が数十 GB
- **確認**:
```bash
du -sh ~/.cache/wandb ~/.local/share/wandb ~/.config/wandb 2>/dev/null
du -sh ${WANDB_DIR:-.}/wandb 2>/dev/null
echo "$WANDB_DIR | $WANDB_CACHE_DIR | $WANDB_DATA_DIR | $WANDB_ARTIFACT_DIR"
```
- **原因**: artifact のキャッシュ（ダウンロード・アップロード両方でキャッシュされる）、`artifacts/` への download、同期済みのオフライン run の残り
- **対処**:
```bash
wandb artifact cache cleanup 10GB                   # artifact キャッシュを 10GB まで減らす
wandb artifact cache cleanup --remove-temp 5GB
wandb purge-cache --age 7d                          # 7日以上前のキャッシュを削除
wandb sync --clean --clean-old-hours 24             # 同期済みのローカル run を削除
```
  場所は `WANDB_DIR` / `WANDB_CACHE_DIR` / `WANDB_DATA_DIR` / `WANDB_ARTIFACT_DIR` で大容量ディスクに移す。大きなファイルは `add_file(..., skip_cache=True)` や `add_reference` を使う

### 2.5 step の警告が出てログが消える

- **症状**: `Step only supports monotonically increasing values, use define_metric to set a custom x axis`、`(User provided step: 10 is less than current step: 500. Dropping entry: ...)`
- **確認**:
```python
print(run.step)                              # 現在の内部 step
```
```bash
grep -rn "less than current step\|monotonically" logs/wandb/latest-run/logs/
```
- **原因**: `step=epoch` と `step=global_step` を混在させている、`step` 指定ありとなしの `run.log` が混ざっている（なしの呼び出しで内部 step が進む）、resume 後に 0 から数え直している
- **対処**: `step=` を使わず、横軸にしたい値をメトリクスとして記録して `define_metric(..., step_metric=...)` で指定する（1.5）。resume 後は `run.step` 以降から続ける

### 2.6 アレイジョブで run が重複する / ばらばらになる

- **症状**: requeue のたびに別の run ができる、同じ設定の run が2つある、group がばらばらで比較しにくい
- **確認**:
```python
runs = wandb.Api().runs("entity/toxpatho-ssl", filters={"group": "dino_123456"})
print([(r.id, r.name, r.state) for r in runs])
```
```bash
sacct -j <jobid> --format=JobID,State,Restarts
```
- **原因**: `id` を指定していないので requeue や再投入で新規 run になる。`name` はランダムまたは重複可能（ID ではない）
- **対処**: 1.7 のように `WANDB_RUN_ID` を Slurm の ID から決め、`WANDB_RESUME=allow`。group は `SLURM_ARRAY_JOB_ID` を含めて揃える。重複した run は `Api` で確認してから削除する（`r.delete()`）

### 2.7 画像・表を記録すると遅い / メモリが増える

- **症状**: 画像を記録するようにしてから学習が遅い、`wandb-core` のメモリ使用量が大きい、終了時のアップロードが終わらない
- **確認**:
```bash
du -sh logs/wandb/latest-run/files/media/
top -u $USER | grep wandb-core
```
```python
t = time.perf_counter(); run.log({"samples": imgs}); print(time.perf_counter() - t)
```
- **原因**: 毎 step 大量の `wandb.Image`（大きな解像度）を記録している。`wandb.Table` の行数が多い。`run.watch(log="all", log_freq=小)` を使っている
- **対処**: 画像はエポックごと・数枚に絞り、縮小してから渡す。表は評価の最後に1回。`watch` は必要なときだけ。終了時に待ちたくない場合は `wandb.Settings(finish_timeout=...)`（超えると crashed 扱いになる点に注意）

### 2.8 コンテナ内で API キーが見つからない

- **症状**: Apptainer 内で API キーが見つからない旨のエラーで落ちる、またはログインのプロンプトで止まる
- **確認**:
```bash
apptainer exec img.sif sh -c 'echo $HOME; ls -la ~/.netrc; env | grep WANDB_API_KEY | cut -c1-20'
apptainer exec img.sif wandb status
```
- **原因**: `wandb login` の結果は `~/.netrc` に保存される。コンテナで `--no-home` / `--containall` を使っている、`HOME` が変わっているなどで `.netrc` が見えない。非対話の実行なのでログインのプロンプトに答えられない
- **対処**: `$HOME` を bind する（既定では bind される）。または `APPTAINERENV_WANDB_API_KEY` で渡す。キーはスクリプトに直接書かず、権限 600 のファイルから読む（`export WANDB_API_KEY=$(cat ~/.wandb_key)`）

### 2.9 権限エラー（プロジェクト・entity）

- **症状**: `wandb.errors.CommError: permission denied`、`403`、`project not found`、run が意図しない個人アカウント側にできる
- **確認**:
```bash
wandb status                                         # 既定の entity
env | grep -E "WANDB_(ENTITY|PROJECT)"
```
```python
print(wandb.Api().default_entity)
```
- **原因**: チームの entity に所属していない、`entity` を指定していないので個人の entity に作られた、API キーが別のアカウント
- **対処**: `wandb.init(entity="<チーム名>", project=...)` か `WANDB_ENTITY` を明示する。`wandb login --relogin` で正しいアカウントのキーに替える

### 2.10 ノートブックで run が切り替わらない / 前の run に書いてしまう

- **症状**: Jupyter で `wandb.init` を再実行しても前の run に書き込まれる、または前の run が勝手に終了する
- **確認**:
```python
print(wandb.run.id if wandb.run else None, run.id)
```
- **原因**: `reinit` の既定は、ノートブックでは `"finish_previous"`（前の run を終わらせて新規作成）、スクリプトでは `"return_previous"`（まだ終わっていない run を返す）
- **対処**: 明示的に `run.finish()` してから次を作る。複数の run を同時に持つなら `wandb.init(reinit="create_new")` にして、`wandb.log` ではなく各 `run.log` を使う

### 2.11 終了後も run が running のまま / crashed になる

- **症状**: ジョブは終わったのに Web 上で running のまま、正常終了したはずなのに crashed
- **確認**:
```bash
tail -50 logs/wandb/latest-run/logs/debug-internal.log
sacct -j <jobid> --format=JobID,State,ExitCode,Elapsed
```
- **原因**: `run.finish()` を呼ぶ前にプロセスが kill された（時間制限・OOM・`scancel`）、終了時のアップロードが時間制限内に終わらなかった、DDP の rank 0 以外も run を持っていて先に終わった
- **対処**: `with wandb.init(...) as run:` か `try/finally` で `finish` を確実に呼ぶ。時間制限の数分前に終わるよう学習を切り上げる（`#SBATCH --signal`）。オフラインで記録していれば `wandb sync` で上げ直せる
