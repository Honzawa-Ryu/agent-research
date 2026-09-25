# 研究環境で使うツール・ライブラリのチートシート集

> **作成日**: 2026-09-25
> **担当Agent**: Claude Code
> **ステータス**: 作成中
> **タグ**: `#チートシート` `#Slurm` `#Apptainer` `#uv` `#PyTorch` `#WSI`

---

## 📌 概要
- **目的**: `/workspace/andre01/honzawa/` 配下の各プロジェクトで使っているツール・ライブラリについて、使い方の理解を揃えるための早見表をまとめる。
- **方針**:
  - ライブラリ・ツールは汎用的な内容で書く（特定プロジェクトに依存しない）。
  - 内製テンプレート（`template-daily-research-experiments` 系）だけは、実際の使われ方に合わせて書く。
  - コピペしやすいように、コマンド例と表を中心にする。
- **構成**: チートシート本体は `sheets/<カテゴリ>/` に置く。1フォルダ10個制限を守るためにカテゴリ別に分けている。本レポートは目次として使う。
  - `format_rules.md` ではテーマフォルダ直下を `report.md` / `papers/` / `assets/` / `notes/` に限定しているが、本テーマでは `sheets/` を追加した（2026-09-25 にユーザー承認済み）。

---

## 1. チートシート一覧

選定の根拠は [notes/search_log.md](notes/search_log.md) を参照（pyproject の宣言と、実際の import 数を集計した）。

### 1.1 実行基盤 (`sheets/infra/`)

| シート | 対象 | 状態 |
|:---|:---|:---:|
| [slurm.md](sheets/infra/slurm.md) | Slurm 23.11（sbatch / srun / squeue / sacct など） | ✅ |
| [apptainer.md](sheets/infra/apptainer.md) | Apptainer 1.4（def ファイル、build、exec、`--nv`、bind） | ✅ |
| [uv.md](sheets/infra/uv.md) | uv（pyproject、lock、sync、run） | ✅ |
| [research_template.md](sheets/infra/research_template.md) | 内製テンプレートの使い方（Makefile、`env/`、`scripts/`、`shell/` など）。**実際の使われ方に合わせて書く** | ✅ |

### 1.2 WSI処理 (`sheets/wsi/`)

| シート | 対象 | 状態 |
|:---|:---|:---:|
| [openslide_tiffslide.md](sheets/wsi/openslide_tiffslide.md) | OpenSlide / TiffSlide（WSI の読み込み、level、read_region） | ✅ |
| [trident.md](sheets/wsi/trident.md) | TRIDENT（組織検出、パッチ抽出、特徴抽出） | ✅ |
| h5py.md | h5py（パッチ座標・特徴量の保存と読み込み） | 未着手 |
| opencv_pil.md | OpenCV / Pillow（画像 I/O、色空間、基本処理） | 未着手 |

### 1.3 深層学習 (`sheets/dl/`)

| シート | 対象 | 状態 |
|:---|:---|:---:|
| pytorch.md | PyTorch / torchvision（Dataset、DataLoader、学習ループ、AMP、保存） | 未着手 |
| timm.md | timm（モデル作成、事前学習重み、特徴抽出） | 未着手 |
| huggingface_hub.md | huggingface_hub（ログイン、gated モデルのダウンロード、キャッシュ） | 未着手 |
| wandb.md | Weights & Biases（ログ記録、オフライン運用） | 未着手 |

### 1.4 データ・解析 (`sheets/data/`)

| シート | 対象 | 状態 |
|:---|:---|:---:|
| numpy.md | NumPy | 未着手 |
| pandas.md | pandas | 未着手 |
| sklearn.md | scikit-learn（前処理、分割、評価指標） | 未着手 |

### 1.5 可視化・対話環境 (`sheets/viz/`)

| シート | 対象 | 状態 |
|:---|:---|:---:|
| matplotlib_seaborn.md | Matplotlib / seaborn | 未着手 |
| jupyter.md | JupyterLab（Slurm ノード上での起動、ポート転送） | 未着手 |

---

## 2. 対象外にしたもの
pyproject には書かれているが、コード中で一度も import されていないもの。テンプレートから引き継いだ依存と思われる。
- `vllm`, `trl`, `peft`, `bitsandbytes`, `accelerate`, `transformers`, `spacy`, `scispacy`, `elasticsearch`
- `xgboost`, `lightgbm`, `optuna`, `dask`, `duckdb`, `polars`

---

## 3. 参考リソース
- 各ツールの公式ドキュメントは [papers/index.md](papers/index.md) にまとめる。
