# TRIDENT 詳細編（応用・トラブル対応）

> 基本編: [trident.md](trident.md)
> 対象バージョン: TRIDENT 0.3.0（このマシンの `wsi_preprocess/.venv` にあるソースで確認）

## Part 1. 応用・上級機能

### 1.1 処理するスライドを細かく指定する（Processor の引数）

```python
from trident import Processor

proc = Processor(
    job_dir="results/trident",
    wsi_source="data/wsis",
    custom_list_of_wsis="slides.csv",     # 下の CSV
    custom_mpp_keys=["myscanner.MPP"],   # 追加で探すキー（値が float に変換できるもの）
    reader_type="openslide",              # 自動判定を上書き
    max_workers=8,                        # DataLoader のワーカー数の上限
    skip_errors=True,
)
```

`custom_list_of_wsis` の CSV:

```csv
wsi,mpp
case01.svs,0.2527
subdir/case02.ndpi,
case03.tif,0.5
```

| 列 | 必須 | 内容 |
|:---|:---|:---|
| `wsi` | 必須 | `wsi_source` からの相対パス（拡張子込み）。1 つでも存在しないとエラーで止まる |
| `mpp` | 任意 | 空欄はメタデータから取得する。値があればメタデータより優先される |

- `custom_mpp_keys` は既定のキー（`openslide.mpp-x`, `openslide.mirax.MPP`, `aperio.MPP`, `hamamatsu.XResolution`, `openslide.comment`）の**後に**探される。既定のキーで値が取れれば、そちらが使われる
- `selected_wsi_paths=[...]` でパスのリストを直接渡すこともできる（CSV も探索もしない）
- `search_nested=True` でサブフォルダも探す。出力のファイル名はファイル名だけになるので、別のフォルダに同名のスライドがあると衝突する

### 1.2 倍率と MPP の決まり方

TRIDENT は MPP から「倍率」を段階的に決めて、パッチ抽出ではその倍率を使う。

| MPP | 判定される倍率 |
|:---|:---|
| < 0.16 | 80 |
| < 0.2 | 60 |
| < 0.3 | 40 |
| < 0.6 | 20 |
| < 1.2 | 10 |
| < 2.4 | 5 |
| それ以上 | エラー（`Identified mpp is very low`） |

- MPP が取れない場合は `openslide.objective-power` を整数にして使う
- パッチ抽出（`extract_tissue_coords`）は、この倍率から pixel size を `10 / mag` として計算する。MPP 0.2527 も 0.23 も「40x」として扱われ、20x・256 px のパッチは level 0 で 512 px になる
- 組織検出（`segment_tissue`）は MPP の実測値を使う
- `patch_size_level0 = patch_size * level0_magnification // target_magnification`（整数の割り算）

MPP の違いを厳密に揃えたい場合は、1.3 節の `create_patcher` で `src_pixel_size` / `dst_pixel_size` を指定する。

### 1.3 WSIPatcher でパッチを直接取り出す

```python
from trident import load_wsi

wsi = load_wsi("slide.svs", tissue_seg_path="results/trident/contours_geojson/slide.geojson")

patcher = wsi.create_patcher(
    patch_size=224,
    src_pixel_size=wsi.mpp,        # 実際の MPP
    dst_pixel_size=0.5,            # 出力の MPP（dst_mag より優先）
    mask=wsi.gdf_contours,         # 組織マスク（GeoDataFrame）
    threshold=0.5,                 # パッチ内の組織割合の下限
    pil=True,                      # PIL で返す（False なら NumPy）
)
print(len(patcher))
for tile, x, y in patcher:         # (x, y) は level 0 の左上座標
    ...
img = patcher.visualize()          # パッチ位置を重ねたサムネイル
```

| 引数 | 既定値 | 説明 |
|:---|:---|:---|
| `src_pixel_size` / `src_mag` | どちらか必須 | 入力の MPP か倍率 |
| `dst_pixel_size` / `dst_mag` | 入力と同じ | 出力の MPP か倍率 |
| `overlap` | 0 | 出力解像度でのピクセル数 |
| `custom_coords` | None | (N, 2) の level 0 座標。グリッドを作らずこれを使う |
| `coords_only` | False | True なら `(x, y)` だけを返す |
| `threshold` | 0.15（`create_patcher`） | `WSIPatcher` を直接作るときの既定値は 0 |

- `create_patcher` は `scan_order`（`row-major` / `col-major`）と `interpolation` を受け取らない。`patcher.interpolation = cv2.INTER_AREA` のように後から設定するか、`WSIPatcher(...)` を直接作る
- `mask` を渡すと、ポリゴンを簡略化してから交差判定をする。境界付近のパッチは多少ずれる

### 1.4 自前のモデルで推論する（WSIPatcherDataset）

```python
from torch.utils.data import DataLoader
from trident import WSIPatcherDataset
from trident.IO import read_coords

attrs, coords = read_coords("results/trident/20x_256px_0px_overlap/patches/slide_patches.h5")
patcher = wsi.create_patcher(
    patch_size=attrs["patch_size"],
    src_mag=attrs["level0_magnification"], dst_mag=attrs["target_magnification"],
    custom_coords=coords, pil=True,
)
ds = WSIPatcherDataset(patcher, transform)          # __getitem__ は (tile, (x, y)) を返す
dl = DataLoader(ds, batch_size=128, num_workers=8)
for imgs, (xs, ys) in dl:
    ...
```

- TRIDENT 自身の特徴抽出と同じ読み方になる。座標は h5 の順番のまま
- `num_workers > 0` では Dataset が pickle される。TRIDENT 内部では、pickle に失敗したら `fork` に切り替える処理が入っているが、自分で DataLoader を作る場合は入っていない

### 1.5 自作・未登録のパッチエンコーダを使う

```python
import torch
from trident.patch_encoder_models import CustomInferenceEncoder

enc = CustomInferenceEncoder(
    enc_name="my_vit",              # 出力フォルダ名 features_my_vit/ に使われる
    model=my_model,                 # forward(x) -> (B, D)
    transforms=my_eval_transforms,  # PIL -> Tensor
    precision=torch.float16,        # autocast の dtype。float32 なら autocast しない
)
proc.run_patch_feature_extraction_job(coords_dir="20x_256px_0px_overlap", patch_encoder=enc, device="cuda:0")
```

登録済みのエンコーダに渡せる主な引数:

```python
from trident.patch_encoder_models import encoder_factory
enc = encoder_factory("uni_v1", weights_path="/path/to/pytorch_model.bin")   # ローカルの重み
enc = encoder_factory("uni_v1", target_img_size=448)                         # 入力解像度を変える
```

- `target_img_size` はパッチサイズ（ViT の patch size）の倍数にする。対応するのは `uni_v1`, `uni_v2`, `virchow`, `virchow2`, `gigapath`, `hoptimus0/1`, `h0-mini`, `gpfm`, `lunit-vits8`, `kaiko-*` など（`RESIZE_SUPPORTED_PATCH_ENCODERS`）
- 0.3.0 の登録名には基本編の表のほかに `midnight12k`, `genbio-pathfm`, `gemma4-e4b`, `gemma4-26b` がある

### 1.6 重みの置き場所と探す順番

| 種類 | 探す順番 |
|:---|:---|
| パッチエンコーダ | `weights_path` 引数 → `patch_encoder_models/local_ckpts.json` → Hugging Face から取得 |
| スライドエンコーダ | `slide_encoder_models/local_ckpts.json` → Hugging Face など |
| 組織検出 | `segmentation_models/local_ckpts.json` → `$TRIDENT_HOME`（未設定なら `$XDG_CACHE_HOME/trident`、さらに未設定なら `~/.cache/trident`）へ取得 |

- `local_ckpts.json` は site-packages の中にある。パッケージを入れ直すと消えるので、パッチエンコーダは `weights_path` 引数で渡す方が安全
- `local_ckpts.json` の相対パスは、そのフォルダの `model_zoo/` からの相対になる
- **パスが存在しない場合、エラーにならず空扱いになり、ダウンロードに進む**（2.2 節）

### 1.7 スライドエンコーダ

```python
from trident.slide_encoder_models import encoder_factory as slide_encoder_factory

slide_enc = slide_encoder_factory("titan")
proc.run_slide_feature_extraction_job(slide_encoder=slide_enc, coords_dir="20x_512px_0px_overlap", device="cuda:0")
# 必要なパッチ特徴（titan なら conch_v15）がなければ、先に自動で抽出する
```

| スライドエンコーダ | 必要なパッチエンコーダ | 出力次元 | 備考 |
|:---|:---|:---|:---|
| `titan` | `conch_v15` | 768 | gated |
| `feather` | `conch_v15` | | |
| `feather_uni_v2` | `uni_v2` | | |
| `care` | `conch_v15` | 512 | |
| `prism` | `virchow` | 1280 | Python 3.10 以上 |
| `chief` | `ctranspath` | 768 | CHIEF のリポジトリを clone してパスを設定 |
| `gigapath` | `gigapath` | 768 | `flash_attn==2.5.8` と `gigapath` パッケージが必要 |
| `madeleine` | `conch_v1` | 512 | 別パッケージが必要 |
| `mean-<patch_encoder>` | 名前の後半 | パッチ特徴と同じ | 平均をとるだけ。重みは不要 |
| `abmil` | — | | `pretrained=False` 専用（学習用の器） |

- `threads` は登録されているが、0.3.0 では "Coming Soon" の例外になる
- `titan` などは `patch_size_level0` を属性から読んで使う。学習時の設定（TITAN なら 20x・512 px）に合わせてパッチを作る
- スライド特徴は常に `.h5` で保存される（`saveas` は使われない）

### 1.8 組織検出の選択肢

```python
from trident.segmentation_models import segmentation_model_factory

seg = segmentation_model_factory("hest", confidence_thresh=0.5)       # 既定。10x で推論
seg = segmentation_model_factory("grandqc")                           # 1x で推論（速い）
seg = segmentation_model_factory("otsu")                              # 大津の二値化。重み不要
art = segmentation_model_factory("grandqc_artifact")                  # アーティファクト除去
art = segmentation_model_factory("grandqc_artifact", remove_penmarks_only=True)   # ペンの跡だけ除去

proc.run_segmentation_job(seg, seg_mag=10, artifact_remover_model=art, device="cuda:0")
```

| 項目 | `WSI.segment_tissue` | `Processor.run_segmentation_job` |
|:---|:---|:---|
| `holes_are_tissue` の既定値 | True | **False** |
| 結果 | `job_dir=None` なら GeoDataFrame を返す | 常にファイルに保存 |

- `confidence_thresh` を下げると、薄い組織も拾う（その分、背景も拾いやすい）
- `grandqc_artifact` はぼけた領域も除去する。スキャナによっては組織がすべて消えることがある（2.6 節）

#### 多クラスのセグメンテーション（segment_semantic）

```python
mask, scale, gdf = wsi.segment_semantic(my_seg_model, target_mag=10, return_contours=True, device="cuda:0")
# mask: 縮小された (H, W) のクラスラベル。scale: level 0 に対する縮小率
```

- モデルは `input_size`, `precision`, `eval_transforms` 属性を持ち、`(B, H, W)` の uint8 を返す必要がある（`SegmentationModel` を継承するのが簡単）
- 前処理や推論を変えたい場合は `collate_fn` と `inference_fn` を渡す

### 1.9 CLAM の座標ファイルを使う

TRIDENT は、自分の形式の属性（`patch_size`, `level0_magnification`, `target_magnification`）が見つからないと、CLAM 形式（`patch_size`, `patch_level`, `custom_downsample`）として読み直す。`extract_patch_features` などに CLAM の h5 をそのまま渡せる。

```python
from trident import WSIPatcher
patcher = WSIPatcher.from_legacy_coords_file(wsi, "clam/patches/slide.h5", coords_only=False, pil=True)
```

### 1.10 ヒートマップを描く

```python
from trident import visualize_heatmap

path = visualize_heatmap(
    wsi, scores=attn,                 # (N,) パッチごとのスコア
    coords=coords,                    # (N, 2) level 0 座標
    patch_size_level0=attrs["patch_size_level0"],
    vis_mag=2.5,                      # 表示倍率（vis_level より優先）
    normalize=True,                   # 順位に変換して 0〜1 にする
    num_top_patches_to_save=10,       # 上位パッチも保存
    output_dir="heatmaps", filename="slide.png",
)
```

- `normalize=True` は順位ベース。スコアの絶対値を比べたいときは False にして、自分で 0〜1 に揃える

### 1.11 パッチ画像を書き出して確認する

```python
proc.run_patching_job(target_magnification=20, patch_size=256,
                      dump_patches=True, dump_patches_max=200, dump_patches_format="jpg")
# <job_dir>/20x_256px_0px_overlap/patch_images/<slide>/000000_x..._y....jpg
```

1 枚だけなら `wsi.dump_patches(coords_path, save_patches_dir, max_patches=100)`。

### 1.12 普通の画像をピラミッド TIFF にする（Python API）

```python
from trident import AnyToTiffConverter

conv = AnyToTiffConverter(job_dir="converted", bigtiff=True)
conv.process_all(input_dir="imgs", mpp_csv="mpp.csv", downscale_by=1, num_workers=4)
```

- CSV は `wsi,mpp` の 2 列。pyvips で 256 px タイル・JPEG 圧縮のピラミッド TIFF を書く
- 変換後は必ず MPP を確認する（2.12 節）

### 1.13 実行記録を読む

| ファイル | 中身 |
|:---|:---|
| `_config_segmentation.json`, `<coords_dir>/_config_coords.json`, `_config_feats_<enc>.json` | 実行時の引数 |
| `_logs_segmentation.txt`, `<coords_dir>/_logs_coords.txt`, `_logs_feats_<enc>.txt` | スライドごとの最新の結果（`名前: メッセージ`） |
| `wsi_states/<slide>__<hash>.json` | スライドごとのタスクの状態（running / completed / skipped / error と理由） |

```bash
grep -h "ERROR" results/trident/_logs_*.txt results/trident/*/_logs_*.txt
```

## Part 2. トラブル対応

### 2.1 gated モデルで 401 / 403

- **症状**: `encoder_factory("uni_v1")` などで `Failed to download ... make sure that you were granted access` と出る
- **確認**:
```bash
trident-doctor --profile patch-encoders --check-gated     # トークンと各 gated リポジトリへのアクセスを確認
hf auth whoami
```
- **原因**: HF でアクセス申請が承認されていない、トークンが設定されていない、または別アカウントのトークンを使っている
- **対処**: モデルのページで申請して承認を待つ。`hf auth login` をやり直す。ジョブ内では `HF_TOKEN` が渡っているか確認する

### 2.2 オフラインの計算ノードで重みが読めない

- **症状**: `Internet connection does seem not available. Auto checkpoint download is disabled.` と出る。`local_ckpts.json` にパスを書いたのに同じエラーが出る
- **確認**:
```bash
python -c "from trident.IO import get_weights_path; print(repr(get_weights_path('patch', 'uni_v1')))"
python -c "from trident.IO import has_internet_connection; print(has_internet_connection())"
```
- **原因**:
  - 接続の確認は import 時に一度だけ行われる（`HF_ENDPOINT` の 443 番ポートへ接続を試す）
  - `local_ckpts.json` のパスが存在しないと、`get_weights_path` は空文字を返し、ダウンロードに進む。typo でもエラーにならない
- **対処**:
  - ログインノードで一度実行してダウンロードしておき、そのファイルのパスを `weights_path=` で渡す
  - 組織検出の重みは `$TRIDENT_HOME` に置かれる。ログインノードと計算ノードで同じ値にする
  - `get_weights_path` の結果が空でないことを先に確認する

### 2.3 ロックが残って処理が飛ばされる

- **症状**: ログや進捗表示に `is locked. Skipping...` と出て、何もされないスライドがある
- **確認**:
```bash
find results/trident -name "*.lock" | head
cat results/trident/20x_256px_0px_overlap/features_uni_v1/slide.h5.lock   # pid / hostname / created_at
```
- **原因**: 処理中に作られる `<出力>.lock` が、kill や OOM で消されずに残っている。`KeyboardInterrupt` 以外の終了では消えない
- **対処**:
```python
from trident.IO import clear_dead_locks
print(clear_dead_locks("results/trident"))      # 同じホストで PID が死んでいるもの、24 時間以上前のものを消す
```
  - 他のジョブがまだ動いていないことを確認してから消す
  - **パッチ座標（`patches/*.h5`）は、出力ファイルがあればロックを見ずに「生成済み」と判断する。** kill された場合は、`.lock` と一緒に出力ファイルも消してやり直す

### 2.4 MPP が取れないというエラー

- **症状**: `Unable to extract MPP from slide metadata` や `Unable to determine magnification` で止まる
- **確認**:
```python
import openslide
s = openslide.OpenSlide("slide.tif")
print({k: v for k, v in s.properties.items() if any(t in k.lower() for t in ("mpp", "resolution", "power"))})
```
- **原因**: メタデータに MPP も倍率もない。TIFF の解像度の単位が `centimeter` / `inch` 以外
- **対処**: 次のどれかで渡す
  - 1 枚なら `load_wsi(path, mpp=0.25)`
  - まとめて処理するなら CSV の `mpp` 列（1.1 節）
  - 独自のキーに入っているなら `custom_mpp_keys`

### 2.5 `Identified mpp is very low` で止まる

- **症状**: 低解像度の画像（サムネイル、縮小済み TIFF）で `ValueError: Identified mpp is very low` になる
- **確認**:
```python
import openslide
p = openslide.OpenSlide("img.tif").properties
print(p.get("openslide.mpp-x"), p.get("tiff.XResolution"), p.get("tiff.ResolutionUnit"))   # MPP が 2.4 以上になっていないか
```
- **原因**: MPP が 2.4 以上だと、対応する倍率がないのでエラーにしている（1.2 節の表）
- **対処**: この画像を TRIDENT で扱うのは想定外。MPP が間違っている（単位の取り違えで 10 倍になっている等）なら正しい値を渡す。本当に低解像度なら、TRIDENT を使わずに処理する

### 2.6 パッチが少なすぎる・0 枚になる

- **症状**: パッチ数が極端に少ない。`Empty GeoDataFrame` や `Saving empty features` の警告が出る
- **確認**:
```bash
ls results/trident/contours/ | head                        # 輪郭の画像を目で見る
grep -h "empty\|Artifact removal" results/trident/_logs_segmentation.txt
```
```python
from trident.IO import read_coords
attrs, coords = read_coords(".../patches/slide_patches.h5"); print(len(coords), attrs)
```
- **原因**:
  - 組織検出が組織を拾えていない（染色が薄い、ペンの跡、背景が暗い）
  - `grandqc_artifact` がぼけた領域をすべてアーティファクトとして消した
  - `min_tissue_proportion` が高すぎる
- **対処**:
  - `contours/*.jpg` を見て、検出がおかしいスライドを特定する
  - アーティファクト除去を外すか、`remove_penmarks_only=True` にする
  - `confidence_thresh` を下げる、または `grandqc` / `otsu` を試す
  - 組織検出をやり直すときは `contours/`, `contours_geojson/`, `thumbnails/` の該当ファイルを消す（下流のパッチ座標と特徴も消す）

### 2.7 GPU のメモリが足りない（CUDA OOM）

- **症状**: 特徴抽出や組織検出で `CUDA out of memory`
- **確認**:
```bash
nvidia-smi                                          # 他のプロセスが GPU を使っていないか
```
- **原因**: `batch_limit`（既定 512）はそのまま DataLoader のバッチサイズになる。大きなモデル（ViT-g 系など）や `target_img_size` を大きくした場合は収まらない
- **対処**: `batch_limit=128` などに下げる。組織検出は `batch_size` を下げる。1 つの GPU で複数のジョブを動かさない

### 2.8 CPU を使いすぎる・Slurm で遅い

- **症状**: 割り当てたコア数より多くのワーカーが立ち上がり、ノード全体が重くなる。メモリも多く使う
- **確認**:
```python
import os
from trident.IO import get_num_workers
print(os.cpu_count(), len(os.sched_getaffinity(0)), get_num_workers(512))
```
- **原因**: TRIDENT のワーカー数は `os.cpu_count()` の 75% で決まる。`os.cpu_count()` はジョブに割り当てられたコア数ではなく、ノードの全コア数を返す
- **対処**: `Processor(..., max_workers=<割り当てコア数>)` または `load_wsi(..., max_workers=...)` で上限を与える。`max_workers=0` なら単一プロセスで読む。ただし `Processor` で `custom_list_of_wsis` を使う場合、同じ値がファイル存在確認のスレッド数にも使われ、0 だとエラーになる。1 以上にする

### 2.9 特徴量と座標の対応がとれない

- **症状**: 特徴量の行数と座標の数が合わない。`.pt` で保存したら座標がない
- **確認**:
```python
import h5py
with h5py.File(".../features_uni_v1/slide.h5") as f:
    print(f["features"].shape, f["coords"].shape, dict(f["coords"].attrs))
```
- **原因**:
  - `saveas="pt"` は特徴量の配列だけを保存し、座標は保存しない
  - パッチ座標を作り直したのに、古い特徴量が残っていて飛ばされた
- **対処**: `h5` で保存し、特徴量ファイル内の `coords` を使う（行の順番は特徴量と同じ）。座標を作り直したら、その下の `features_*` も消す

### 2.10 設定を変えて再実行しても結果が変わらない

- **症状**: 組織検出のモデルやパラメータを変えたのに、同じ輪郭のまま
- **確認**:
```bash
cat results/trident/_config_segmentation.json       # 前回の設定
ls results/trident/contours/ | wc -l
```
- **原因**: 出力があれば飛ばす仕組み。パッチ座標のフォルダ名には倍率・サイズ・重なりが入るが、組織検出の出力（`contours/` など）はパラメータで分かれない。`Processor` は作成時に既存の geojson を読み込む
- **対処**: `job_dir` を分けるか、`contours/`, `contours_geojson/`, `thumbnails/` と下流の出力を消してから、`Processor` を作り直す

### 2.11 使う GPU を指定できない

- **症状**: `device="cuda:1"` を指定したのに GPU 0 が使われる。または `invalid device ordinal`
- **確認**:
```bash
echo $CUDA_VISIBLE_DEVICES
python -c "import torch; print(torch.cuda.device_count())"
```
- **原因**: Slurm などで `CUDA_VISIBLE_DEVICES` が設定されていると、見える GPU の番号が 0 から振り直される
- **対処**: ジョブ内では割り当てられた GPU を `cuda:0` として指定する。モデルもデータも `device` 引数の GPU に送られる（`segment_tissue`, `extract_patch_features` の `device`）

### 2.12 変換した TIFF の MPP がおかしい

- **症状**: `trident convert` や `AnyToTiffConverter` で作った TIFF を読むと、MPP が CSV に書いた値と違う
- **確認**:
```python
import openslide
s = openslide.OpenSlide("converted/img.tif")
print(s.properties.get("tiff.XResolution"), s.properties.get("tiff.ResolutionUnit"), s.properties.get("openslide.mpp-x"))
```
- **原因**: 0.3.0 のソースでは、書き込み時に `resunit=CM` と `xres=1/(mpp*1e-4)`（pixels/cm の値）を渡している。一方、libvips の `xres` は pixels/mm で解釈される。そのため 10 倍ずれる可能性がある（ソースを読んだうえでの推測。実ファイルでは未検証）
- **対処**: 変換後に上の確認をする。ずれていれば、パッチ抽出時に `load_wsi(..., mpp=<正しい値>)` や CSV の `mpp` 列で上書きする

### 2.13 一部のパッチが真っ白になっている

- **症状**: 特徴抽出は最後まで終わるが、`dump_patches` で見ると白一色のパッチが混じっている。`Corrupt region at ...` の警告が出ている
- **確認**:
```bash
python my_extract.py 2>&1 | grep -i "corrupt region\|fallback read failed"
```
- **原因**: `OpenSlideWSI.read_region` は、OpenSlide がエラーを出すとスライドを開き直し、level 0 から読み直す。それも失敗すると白い画像を返して処理を続ける
- **対処**: 警告をログに残しておき、該当スライドを [openslide_tiffslide_deep.md](openslide_tiffslide_deep.md) の 2.8 節の手順で確認する。多い場合は、そのスライドの特徴量を使うかどうかを判断する

### 2.14 スライドエンコーダが読み込めない

- **症状**: `Coming Soon!`、`Unknown encoder name`、`flash_attn` のエラー、`KeyError` などで止まる
- **確認**:
```python
from trident.slide_encoder_models import encoder_registry
from trident.slide_encoder_models.load import slide_to_patch_encoder_name
print(sorted(encoder_registry)); print(slide_to_patch_encoder_name)
```
```bash
trident-doctor --profile slide-encoders
```
- **原因**:
  - `threads` は 0.3.0 では未提供
  - `gigapath` は `flash_attn==2.5.8`、`chief` / `madeleine` は別リポジトリが必要
  - `mean-kaiko-vit8s` などの平均エンコーダの名前が、パッチエンコーダの名前（`kaiko-vits8`）と一致しない。`Processor` が対応するパッチ特徴を探すときに失敗する
- **対処**: 必要な依存を入れる。kaiko の平均特徴が欲しい場合は、パッチ特徴を読み込んで自分で平均をとる
