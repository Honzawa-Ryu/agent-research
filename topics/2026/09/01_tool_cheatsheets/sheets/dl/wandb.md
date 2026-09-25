# Weights & Biases (wandb) チートシート

> 対象: wandb 0.2x 系（このマシンの `.venv` は 0.27）
> 公式: https://docs.wandb.ai/
> 詳細編（応用・トラブル対応）: [wandb_deep.md](wandb_deep.md)

## 1. 基本概念

| 用語 | 意味 |
|:---|:---|
| entity | ユーザー名またはチーム名 |
| project | 実験をまとめる単位 |
| run | 1回の実行。`wandb.init` から `finish` まで |
| config | ハイパーパラメータ。run の条件として記録され、あとで絞り込みに使える |
| summary | run の最終値（例: best_auc）。一覧表に表示される |
| artifact | バージョン管理されたファイル（データセット、モデルの重みなど） |
| group / job_type | run をまとめる・分類するためのラベル（例: seed 違いを同じ group にする） |

## 2. 最初の設定

```bash
wandb login                 # API キーを入力（~/.netrc に保存される）
wandb login --relogin
wandb status                # 設定の確認
```

API キーは https://wandb.ai/authorize で取得する。環境変数 `WANDB_API_KEY` でも渡せる。

## 3. 基本の使い方

```python
import wandb

run = wandb.init(
    project="toxpatho-ssl",
    name="dino_seed0",                 # 表示名（省略するとランダムな名前になる）
    group="dino",                      # seed 違いなどをまとめる
    tags=["ssl", "uni"],
    config={"lr": 1e-4, "epochs": 50, "model": "vit_s"},
    dir="logs/wandb",                  # ローカルに保存する場所
    # mode="offline",                  # 第5節を参照
)

cfg = run.config                        # 辞書のように使える
for epoch in range(cfg.epochs):
    ...
    run.log({"train/loss": loss, "val/auc": auc, "lr": sched.get_last_lr()[0]}, step=epoch)

run.summary["best_val_auc"] = best_auc
run.finish()
```

- `with wandb.init(...) as run:` と書けば、`finish` を忘れない
- キーに `/` を入れると、Web 画面でセクションごとにまとまる（`train/`, `val/`）
- `step` を指定するときは単調に増やす。過去の step に書き込むと無視される

## 4. 画像・表・その他

```python
run.log({"samples": [wandb.Image(img, caption=f"pred={p}") for img, p in zip(imgs, preds)]})
run.log({"heatmap": wandb.Image(fig)})                         # matplotlib の Figure

table = wandb.Table(columns=["slide", "pred", "label"], data=rows)
run.log({"predictions": table})

run.log({"roc": wandb.plot.roc_curve(y_true, y_prob)})
run.log({"cm": wandb.plot.confusion_matrix(y_true=y, preds=pred, class_names=names)})

run.define_metric("val/auc", summary="max")                   # summary に最大値を残す
run.watch(model, log="gradients", log_freq=100)               # 勾配の分布を記録（重い）
```

## 5. オフライン運用（計算ノードが外部に接続できない場合）

```bash
export WANDB_MODE=offline        # または wandb.init(mode="offline")
python train.py                  # logs/wandb/offline-run-*/ に保存される

# あとでネットワークのあるノードからアップロード
wandb sync logs/wandb/offline-run-*
wandb sync --sync-all            # 未アップロードのものをまとめて
```

| mode | 動き |
|:---|:---|
| `online`（既定） | 実行中にアップロードする |
| `offline` | ローカルに保存するだけ。あとで `wandb sync` |
| `disabled` | 何もしない（`wandb.log` などは呼んでもエラーにならない）。デバッグ時に便利 |

## 6. Artifact（ファイルのバージョン管理）

```python
art = wandb.Artifact("uni-features", type="dataset")
art.add_dir("outputs/features")
run.log_artifact(art)

art = run.use_artifact("uni-features:latest")
path = art.download()
```

アップロードは外部への保存になる。データの扱いの規約（患者データ、社外秘データなど）に注意する。

## 7. 過去の run をコードから読む

```python
api = wandb.Api()
runs = api.runs("entity/toxpatho-ssl", filters={"config.model": "vit_s"})
for r in runs:
    print(r.name, r.config["lr"], r.summary.get("best_val_auc"))
hist = runs[0].history(keys=["val/auc"])        # pandas.DataFrame
```

## 8. Sweep（ハイパーパラメータ探索）

```yaml
# sweep.yaml
program: train.py
method: bayes             # grid / random / bayes
metric: {name: val/auc, goal: maximize}
parameters:
  lr: {min: 1e-5, max: 1e-3, distribution: log_uniform_values}
  batch_size: {values: [32, 64]}
```

```bash
wandb sweep sweep.yaml                  # sweep ID が表示される
wandb agent entity/project/<sweep_id>   # 各ジョブでこれを実行する（Slurm のアレイジョブと組み合わせる）
```

内製テンプレートを使っている場合は、GRID 方式で探索するのが基本（[research_template.md](../infra/research_template.md)）。

## 9. 環境変数

| 変数 | 用途 |
|:---|:---|
| `WANDB_API_KEY` | API キー |
| `WANDB_MODE` | `online` / `offline` / `disabled` |
| `WANDB_PROJECT` / `WANDB_ENTITY` | 既定のプロジェクト・entity |
| `WANDB_DIR` | ローカルに保存する場所（既定はカレントの `wandb/`） |
| `WANDB_CACHE_DIR` / `WANDB_DATA_DIR` | キャッシュ・artifact の保存場所（既定はホーム以下） |
| `WANDB_SILENT=true` | ログ出力を減らす |
| `WANDB_RUN_GROUP` | group を外から指定する（アレイジョブで便利） |

Apptainer 内で使うときは、`APPTAINERENV_WANDB_MODE=offline` のようにすると、コンテナ内に環境変数が渡る。

## 10. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| `wandb.init` で止まる | ネットワークに接続できない。`WANDB_MODE=offline` にする |
| ホームの容量が足りない | キャッシュと artifact が大きい。`WANDB_CACHE_DIR` / `WANDB_DATA_DIR` / `WANDB_DIR` を大きいディスクに向ける |
| `wandb/` がリポジトリに入ってしまう | `.gitignore` に `wandb/` を追加する |
| グラフの横軸がずれる | `step` を自分で指定したり、しなかったりしている。どちらかに統一する |
| アレイジョブで run がばらばらで見にくい | `group` に実験名、`name` に seed などを入れる |
| 途中で落ちた run が running のまま | 数分で crashed に変わる。オフラインの run は `wandb sync` し直す |
