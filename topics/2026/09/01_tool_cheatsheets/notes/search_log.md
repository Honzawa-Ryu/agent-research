# 調査メモ・検索ログ (Search & Investigation Log)

## 🔍 対象ツールの選定方法（2026-09-25）

- 対象: `/workspace/andre01/honzawa/` 配下の 7 プロジェクト（`00-utils/*`, `01-toxpatho/*`, `02-playground/*`）
- 方法1: 各 `pyproject.toml` の dependencies を集計した
- 方法2: `.py` / `.ipynb` 中の `import X` / `from X` を含むファイル数を集計した（`.venv` は除外）
- 環境: `slurm-wlm 23.11.4`, `apptainer 1.4.5`、コンテナのベースは `nvidia/cuda:12.8.1-cudnn-devel-ubuntu24.04`（env.def 3件）

### import を含むファイル数

| ライブラリ | ファイル数 | | ライブラリ | ファイル数 |
|:---|---:|:---|:---|---:|
| numpy | 212 | | tqdm | 54 |
| pandas | 148 | | torchvision | 38 |
| torch | 134 | | seaborn | 25 |
| yaml | 71 | | openslide | 25 |
| sklearn | 60 | | cv2 | 19 |
| PIL | 59 | | timm | 14 |
| matplotlib | 59 | | scipy | 13 |
| h5py | 55 | | wandb | 8 |

少数: umap 6, trident 6, tiffslide 4, hydra 3, omegaconf 3, webdataset 3, huggingface_hub 3, anomalib 2, plotly 1, datasets 1

宣言だけで import 0: vllm, trl, peft, bitsandbytes, accelerate, transformers, spacy, elasticsearch, xgboost, lightgbm, optuna, dask, duckdb, polars
