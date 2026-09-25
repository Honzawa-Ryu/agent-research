# OpenSlide / TiffSlide 詳細編（応用・トラブル対応）

> 基本編: [openslide_tiffslide.md](openslide_tiffslide.md)
> 対象バージョン: `openslide-python` 1.4.6 + `openslide-bin` 4.0.1（C ライブラリ 4.0.1）、`tiffslide` 4.0.0（このマシンの `wsi_preprocess/.venv` のソースで確認）

## Part 1. 応用・上級機能

### 1.1 プロパティを一通り調べる

```python
import openslide

print(openslide.OpenSlide.detect_format("slide.svs"))   # 'aperio' / 'hamamatsu' / 'mirax' など。未対応なら None
with openslide.OpenSlide("slide.svs") as s:
    for k, v in sorted(s.properties.items()):
        print(k, "=", v[:80])
```

OpenSlide は全形式に共通のキー（`openslide.*`）と、ベンダー固有のキーの両方を持つ。

| 共通キー（定数名） | 中身 |
|:---|:---|
| `PROPERTY_NAME_VENDOR` (`openslide.vendor`) | 形式名 |
| `PROPERTY_NAME_MPP_X` / `_Y` | µm/px |
| `PROPERTY_NAME_OBJECTIVE_POWER` | 対物倍率 |
| `PROPERTY_NAME_BOUNDS_X` / `_Y` / `_WIDTH` / `_HEIGHT` | 画像が実際にある領域（1.2 節） |
| `PROPERTY_NAME_BACKGROUND_COLOR` | 背景色（`ffffff` のような16進）。`get_thumbnail` が透明部分をこの色で塗る |
| `PROPERTY_NAME_QUICKHASH1` | スライドの簡易ハッシュ。同じスライドかどうかの判定に使える |
| `PROPERTY_NAME_BARCODE` | バーコード（個人情報を含むことがある） |
| `PROPERTY_NAME_COMMENT` | 自由記述。Aperio では ImageDescription がそのまま入る |

| ベンダー | よく見るキーの例 |
|:---|:---|
| Aperio (svs) | `aperio.MPP`, `aperio.AppMag`, `aperio.Date` |
| Hamamatsu (ndpi) | `hamamatsu.SourceLens`, `hamamatsu.XResolution` |
| TIFF 共通 | `tiff.XResolution`, `tiff.ResolutionUnit`, `tiff.ImageDescription` |

- 形式ごとにキーの有無は違う。上のキーを前提にせず、手元のスライドで一度ダンプして確認する
- TiffSlide は `tiffslide.level[0].width`、`tiffslide.series-axes` など独自のキーを持つ

### 1.2 bounds（非空領域）だけを扱う

MIRAX などでは、画像の大部分が空で、実際の組織は一部の矩形にしかない。その矩形が `openslide.bounds-*` に入っている。

```python
p = slide.properties
bx = int(p.get(openslide.PROPERTY_NAME_BOUNDS_X, 0))
by = int(p.get(openslide.PROPERTY_NAME_BOUNDS_Y, 0))
bw = int(p.get(openslide.PROPERTY_NAME_BOUNDS_WIDTH, slide.dimensions[0]))
bh = int(p.get(openslide.PROPERTY_NAME_BOUNDS_HEIGHT, slide.dimensions[1]))
# パッチのグリッドを (bx, by)〜(bx+bw, by+bh) の範囲だけに作る
```

- `DeepZoomGenerator(..., limit_bounds=True)` はこの値を使ってタイルの原点をずらす。bounds がないスライドでは全体を使う
- `limit_bounds=True` のとき、DeepZoom のタイル座標は bounds の左上を原点とする。`get_tile_coordinates()` で level 0 の座標に戻せる（1.5 節）

### 1.3 ICC カラープロファイル

OpenSlide 4.0 以降では、スライドに埋め込まれた ICC プロファイルを読める。`read_region` などの返り値にも `img.info["icc_profile"]` として付く。**ただし色変換はされない。**

```python
from PIL import ImageCms
import io

prof = slide.color_profile                     # ImageCmsProfile。埋め込みがなければ None
if prof is not None:
    srgb = ImageCms.createProfile("sRGB")
    tf = ImageCms.buildTransform(prof, srgb, "RGB", "RGB")   # 1回だけ作って使い回す
    patch = slide.read_region((x, y), 0, (512, 512)).convert("RGB")
    patch_srgb = ImageCms.applyTransform(patch, tf)
```

- TiffSlide にも `slide.color_profile` がある（TIFF の ICC タグを読む）
- 学習データ全体でプロファイル変換をするかどうかは揃える。一部だけ変換すると色の分布がずれる
- `buildTransform` は重いので、パッチごとに作らない

### 1.4 タイルキャッシュ（OpenSlide）

OpenSlide は、デコード済みタイルを各スライドのキャッシュに保持している。大きさを変えたり、複数スライドで共有したりできる（OpenSlide 4.0 以降）。

```python
from openslide import OpenSlide, OpenSlideCache

cache = OpenSlideCache(256 * 1024**2)          # 256 MiB（単位はバイト）
for path in paths:
    s = OpenSlide(path)
    s.set_cache(cache)                         # 同じキャッシュを複数スライドで共有
```

| 状況 | 設定の目安 |
|:---|:---|
| 同じ領域を何度も読む（重なりのあるパッチ、可視化） | キャッシュを大きく |
| 多数のワーカーで別々のスライドを読む | ワーカー数 × キャッシュ量がメモリに収まるか確認 |

### 1.5 DeepZoom の詳細

```python
from openslide.deepzoom import DeepZoomGenerator

dz = DeepZoomGenerator(slide, tile_size=254, overlap=1, limit_bounds=True)
lvl = dz.level_count - 1                        # DeepZoom の最後の level = スライドの level 0 相当
args = dz.get_tile_coordinates(lvl, (col, row)) # read_region に渡す ((x0, y0), level, (w, h))
dims = dz.get_tile_dimensions(lvl, (col, row))  # 端のタイルは小さい
xml  = dz.get_dzi("jpeg")                       # .dzi のメタデータ（ビューア用）
```

- DeepZoom の level は、1×1 ピクセルから始まり 2 倍ずつ大きくなる。スライドの level とは番号の向きも段数も違う
- 性能のため `tile_size + 2*overlap` を 2 の累乗にする（254 + 2 = 256）
- `get_tile` は透明部分を背景色で塗った RGB を返す。自前で `read_region` するときとは違う

### 1.6 普通の画像を同じ API で扱う（ImageSlide / open_slide）

```python
from openslide import open_slide, ImageSlide

s = open_slide("path/to/file")      # WSI なら OpenSlide、それ以外（png, jpg）なら ImageSlide
s = ImageSlide(pil_image)           # PIL Image から作ることもできる
```

- `ImageSlide` は level が 1 つだけ（`level_count == 1`）。`properties` は空で MPP はない
- `tiffslide.open_slide` は TiffSlide で開けなければ `NotTiffSlide` にフォールバックする

### 1.7 TiffSlide 固有の機能

```python
import tiffslide

s = tiffslide.TiffSlide("slide.svs")

# 範囲外を詰めずに返す（既定の padding=True はゼロ埋めでサイズを揃える）
arr = s.read_region((x, y), 0, (512, 512), as_array=True, padding=False)

# 埋め込みサムネイルがあれば使う（ピラミッドを読むより速いことがある）
thumb = s.get_thumbnail((1024, 1024), use_embedded=True)

# tifffile の TiffFile に直接アクセス
tf = s.ts_tifffile
print([ser.name for ser in tf.series])
```

#### zarr として扱う

```python
grp = s.zarr_group                 # zarr.Group。キー "0", "1", ... が各 level の配列
lv0 = grp["0"]
print(lv0.shape, lv0.chunks)       # 例: (H, W, 3) と TIFF タイルに対応したチャンク
tile = lv0[y:y+512, x:x+512]       # NumPy 配列で読む

import dask.array as da
d = da.from_array(lv0, chunks=(4096, 4096, 3))   # 遅延評価の配列として扱う
```

- `zarr_group` はスレッドごとにキャッシュされる。スレッドをまたいで同じ `TiffSlide` を使える
- 複数のシリーズを合成する形式では、キーの構造が `"<series>/<level>"` になる。まず `list(grp.keys())` で確認する

#### リモートから読む（fsspec）

```python
s = tiffslide.TiffSlide("s3://bucket/slide.svs", storage_options={"anon": True})
```

- `s3fs` などのプロトコル実装が別途必要（このマシンの `.venv` には入っていない）
- 読むたびに HTTP リクエストが発生する。パッチを大量に読むなら、ローカルにコピーした方が速い

### 1.8 複数の倍率で読むときの定石

```python
def read_at_mpp(slide, x0, y0, size, target_mpp, mpp0):
    """level 0 の (x0, y0) から、target_mpp で size x size のパッチを読む"""
    ds = target_mpp / mpp0
    lvl = slide.get_best_level_for_downsample(ds)
    lds = slide.level_downsamples[lvl]
    read_size = int(round(size * ds / lds))                 # その level で読むピクセル数
    img = slide.read_region((x0, y0), lvl, (read_size, read_size)).convert("RGB")
    return img.resize((size, size), Image.Resampling.BOX) if read_size != size else img
```

- level 0 を読んで縮小するより、近い level を読んで縮小する方がずっと速い
- 縮小には `BOX` か `INTER_AREA` を使う（エイリアシングが少ない）
- level 0 の広い範囲を読むときは、2048 px 程度のブロックに分けてループする

### 1.9 並列に読む

```python
from concurrent.futures import ProcessPoolExecutor

_slide = None
def _init(path):
    global _slide
    _slide = openslide.OpenSlide(path)          # ワーカーごとに開く

def _read(xy):
    return np.asarray(_slide.read_region(xy, 0, (256, 256)).convert("RGB"))

with ProcessPoolExecutor(8, initializer=_init, initargs=(path,)) as ex:
    patches = list(ex.map(_read, coords, chunksize=64))
```

| パターン | 向いている場面 |
|:---|:---|
| スライド単位でプロセスを分ける | 多数のスライドを処理する（一番単純で速い） |
| 1 枚のスライドをワーカーごとに開き直す | 1 枚が巨大で、パッチ数が多い |
| スレッド | OpenSlide の読み込みは C 側で行われ GIL を離すので効く。まず試す価値がある |

### 1.10 MPP を揃える

```python
def get_mpp(slide):
    p = slide.properties
    for key in (openslide.PROPERTY_NAME_MPP_X, "aperio.MPP"):
        if key in p:
            return float(p[key])
    unit, xres = p.get("tiff.ResolutionUnit"), p.get("tiff.XResolution")
    if unit and xres:
        scale = {"centimeter": 1e4, "inch": 25400.0}.get(unit.lower())
        if scale:
            return scale / float(xres)
    return None                                  # 取れない場合は呼び出し側で既知の値を使う
```

- スキャナによって「40x」の MPP は 0.23〜0.27 程度の幅がある。倍率ではなく MPP から縮小率を決める
- データセット内の MPP の分布を最初に集計しておくと、外れ値のスライドに気づける

### 1.11 ピラミッド TIFF に変換する（pyvips）

ピラミッドのない大きな TIFF や、OpenSlide が読めない形式を扱いやすくする。`pyvips` はこの `.venv` に入っている。

```python
import pyvips

mpp = 0.25
img = pyvips.Image.new_from_file("in.tif", access="sequential")
img.tiffsave(
    "out.tif", tile=True, tile_width=256, tile_height=256, pyramid=True,
    compression="jpeg", Q=90, bigtiff=True,
    xres=1000 / mpp, yres=1000 / mpp,            # libvips の xres の単位は pixels/mm
)
```

- 変換後は OpenSlide で開き直して、`level_dimensions` と MPP を確認する
- JPEG 圧縮は非可逆。解析用のマスクやラベル画像は `compression="deflate"` などにする

### 1.12 ラベル・マクロ画像の扱い

- `associated_images["label"]` や `["macro"]` には、患者 ID や施設名が写っていることがある。`openslide.barcode` などのプロパティも同じ
- 図や共有用のデータにはこれらを含めない。サムネイルは `get_thumbnail()`（本体から作る）を使う
- OpenSlide / TiffSlide は読み込み専用で、ラベルを消す機能はない。消す場合は専用ツールを使い、必ずコピーに対して行う

## Part 2. トラブル対応

### 2.1 NFS 上のスライドの読み込みが遅い

- **症状**: 同じコードでもローカルより数倍〜数十倍遅い。ワーカーを増やしても速くならない
- **確認**:
```bash
df -hT /path/to/slides                   # nfs / lustre などか確認
time python -c "import openslide; s=openslide.OpenSlide('slide.svs'); s.read_region((0,0),0,(4096,4096))"
cp slide.svs /tmp/ && time python -c "...同じ処理を /tmp/slide.svs で..."
```
- **原因**: 小さなランダム読み込みがネットワーク越しに大量に発生している。多数のワーカーが同時に読むと、ストレージ側が詰まる
- **対処**:
  - ジョブの最初にローカル SSD（`$TMPDIR` など）へコピーしてから処理する
  - パッチを座標順（行ごと）に並べて読む。ランダム順より局所性が上がる
  - ワーカー数を減らして、スループットが最大になる数を実測する

### 2.2 メモリ使用量が増え続ける

- **症状**: 処理が進むにつれてメモリが増え、最後に OOM で落ちる
- **確認**:
```python
import psutil, os
proc = psutil.Process(os.getpid())
print(proc.memory_info().rss / 1024**3, "GiB")   # ループの前後で比較
print(slide.level_dimensions[level])              # 読もうとしているサイズ
```
- **原因**:
  - level 0 全体など、巨大な範囲を一度に `read_region` している（RGBA なので幅×高さ×4 バイト）
  - スライドを開いたまま次々に新しく開いている（各スライドがキャッシュを持つ）
  - 読んだ PIL Image をリストにため込んでいる
- **対処**:
  - ブロックに分けて読む（1.8 節）
  - 使い終わったスライドは `close()` するか `with` で開く
  - スライドが多いときは `OpenSlideCache` を共有して総量を抑える（1.4 節）

### 2.3 他のビューアと色が違う

- **症状**: QuPath やベンダーのビューアで見た色と、Python で読んだ色が違う（くすんでいる、色相がずれている）
- **確認**:
```python
print(slide.color_profile)                        # None でなければ ICC が埋め込まれている
img = slide.read_region((x, y), 0, (256, 256))
print("icc_profile" in img.info)
```
- **原因**: OpenSlide / TiffSlide は ICC プロファイルを付けて返すだけで、変換はしない。ビューアの方は変換して表示している
- **対処**: 1.3 節の方法で sRGB に変換する。学習と推論で扱いを揃える。PNG で保存する場合、`img.save(path, icc_profile=img.info.get("icc_profile"))` とすれば、対応するビューアでは正しい色で表示される

### 2.4 タイルが黒い・透明になる

- **症状**: スライドの端や空の領域を読むと、黒一色や透明の画像が返る
- **確認**:
```python
img = slide.read_region((x, y), level, (w, h))
a = np.asarray(img)
print(img.mode, a[..., 3].min() if img.mode == "RGBA" else "no alpha")   # 0 なら透明
print(slide.properties.get(openslide.PROPERTY_NAME_BACKGROUND_COLOR))
```
- **原因**:
  - OpenSlide はスライド外や空の領域を `(0, 0, 0, 0)` で返す。そのまま `.convert("RGB")` すると黒になる
  - TiffSlide の `padding=True`（既定）は範囲外を 0 で埋める。こちらも黒になる
  - MIRAX などでは bounds の外側がすべて空
- **対処**:
```python
rgba = slide.read_region((x, y), level, (w, h))
bg = Image.new("RGB", rgba.size, (255, 255, 255))
bg.paste(rgba, mask=rgba.split()[3])              # アルファで白と合成
```
  - TiffSlide では `padding=False` で範囲外を切り詰めるか、座標を画像内に収める
  - グリッドを bounds の範囲に限定する（1.2 節）

### 2.5 level を変えると座標がずれる

- **症状**: level 0 で見つけた位置を level 2 で読むと、数ピクセル〜数十ピクセルずれる
- **確認**:
```python
print(slide.level_downsamples)          # 4.0 ではなく 4.0003 のような値になっていないか
print(slide.level_dimensions)
```
- **原因**:
  - `read_region` の `(x, y)` に level の座標を渡している（基本編 7 節）
  - downsample が整数でない。level の寸法が切り捨てで決まるため、`level_dimensions[0][0] / level_dimensions[k][0]` が 4 ちょうどにならない
  - 自前の変換で `int(x / ds)` と `round(x / ds)` が混在している
- **対処**: 座標は常に level 0 で保存し、変換には `slide.level_downsamples[k]` を使う。丸め方をコード全体で統一する。サイズも `level_dimensions` を基準にする

### 2.6 MPP が取れない・値がおかしい

- **症状**: `openslide.mpp-x` がない、または 0.0001 や 25400 のようなありえない値
- **確認**:
```python
p = slide.properties
for k in p:
    if any(s in k.lower() for s in ("mpp", "resolution", "power", "mag", "lens")):
        print(k, p[k])
```
- **原因**: スキャナや変換ツールが MPP を書き込んでいない。TIFF の `ResolutionUnit` が `none` や `inch` で、単位の変換を間違えている。スキャン後に別ツールで変換したファイルでよく起きる
- **対処**: 1.10 節の順でフォールバックする。それでも取れなければ、スキャナの仕様値を CSV で管理して与える。取れた値がおおむね 0.1〜2.0 の範囲にあるか、チェックを入れておく

### 2.7 一度エラーが出たら以降の読み込みがすべて失敗する

- **症状**: ある座標で `OpenSlideError` が出た後、正常な座標を読んでも同じエラーになる
- **確認**:
```python
try:
    slide.read_region((x, y), 0, (256, 256))
except openslide.OpenSlideError as e:
    print("error:", e)
slide.read_region((0, 0), slide.level_count - 1, (16, 16))   # 別の場所も失敗するか
```
- **原因**: OpenSlide はエラーを「ラッチ」する。一度 `OpenSlideError` が出たオブジェクトは、`close()` 以外のすべての操作が失敗する（公式のドキュメント文字列に明記されている）
- **対処**: エラーを捕まえたらオブジェクトを開き直す。どのパッチが壊れているかを記録しておき、後でまとめて確認する

```python
def safe_read(path, slide, *args):
    try:
        return slide, slide.read_region(*args)
    except openslide.OpenSlideError:
        slide.close()
        slide = openslide.OpenSlide(path)        # 開き直す
        return slide, None                       # このパッチは欠損として扱う
```

### 2.8 ファイルが壊れている・一部だけ読めない

- **症状**: 開くときに `OpenSlideUnsupportedFormatError` が出る。または特定の領域だけ JPEG のデコードエラーになる
- **確認**:
```bash
ls -l slide.svs                                        # サイズが 0 や極端に小さくないか
python -c "import openslide as o; s=o.OpenSlide('slide.svs'); print(s.level_dimensions)"   # 開けるか
python -c "import tifffile; tf=tifffile.TiffFile('slide.svs'); print(tf.series)"
```
- **原因**: 転送の途中で切れた、ダウンロードが不完全、スキャン中に書き込みが中断された
- **対処**:
  - コピー元とハッシュ（`sha256sum`）を比較して、転送を確認する
  - 開けるが一部だけ壊れている場合は、低解像度の level ではその領域を読めることが多い。前処理ではそのパッチを除外して記録する
  - 再取得できないなら、そのスライドをデータセットから外すかどうかを判断する

### 2.9 MIRAX (.mrxs) が開けない

- **症状**: `.mrxs` を開くと `OpenSlideError` や `OpenSlideUnsupportedFormatError` になる
- **確認**:
```bash
ls slide.mrxs slide/                    # 同じ名前のディレクトリがあるか
ls slide/ | head                        # Slidedat.ini と Data*.dat があるか
```
- **原因**: MIRAX は、`.mrxs` ファイルと、同名のディレクトリ（`Slidedat.ini` と多数の `Data*.dat`）の組で1枚になる。`.mrxs` だけをコピーした、ディレクトリ名を変えた、一部の `.dat` が欠けている
- **対処**: ディレクトリごとコピーし、名前を `.mrxs` と揃える。TiffSlide は MIRAX 非対応なので OpenSlide を使う

### 2.10 DICOM が開けない

- **症状**: `.dcm` を開くと非対応形式のエラーになる、または level が 1 つしかない
- **確認**:
```python
print(openslide.__library_version__)          # 4.0 以上か（DICOM 対応は 4.0 から）
print(openslide.OpenSlide.detect_format("x.dcm"))
```
```bash
ls series_dir/ | wc -l                        # 1 シリーズのファイルがそろっているか
```
- **原因**: WSI の DICOM は、level ごと（あるいは付随画像ごと）に別々のファイルになっている。1 ファイルだけを取り出すと、ピラミッドがそろわない。古い `libopenslide` が読み込まれている場合もある
- **対処**: 同じシリーズのファイルをすべて同じディレクトリに置き、そのうちの 1 つを開く。`openslide.__library_version__` が `openslide-bin` の版になっているか確認する

### 2.11 DataLoader やマルチプロセスで固まる・エラーになる

- **症状**: `num_workers > 0` にすると pickle のエラー、ワーカーが止まる、画像が壊れる
- **確認**:
```python
import multiprocessing as mp, torch
print(mp.get_start_method())                  # 'fork' / 'spawn'
# Dataset の __init__ で OpenSlide を開いていないか確認する
```
- **原因**:
  - `OpenSlide` は ctypes のポインタを持つので pickle できない（`spawn` で失敗する）
  - `fork` で親プロセスのハンドルを子に引き継ぐと、ファイル位置やキャッシュの状態が共有され、挙動が不定になる
- **対処**: 基本編 6 節のように、`__getitem__` で初めて使うときに開く。`worker_init_fn` で開いてもよい。親プロセスではメタデータだけ読んで閉じる

### 2.12 TiffSlide で ImageDescription のデコードエラー

- **症状**: TiffSlide で開くと `UnicodeDecodeError` などが出る。OpenSlide では開ける
- **確認**:
```bash
python -m tiffslide.repair fix-description-tag-encoding slide.svs     # 既定は dry-run。問題の有無を表示するだけ
```
- **原因**: ImageDescription タグに ASCII 以外のバイト（日本語のコメントなど）が入っている
- **対処**:
  - ファイルを変更せずに読むなら、開く前に `tiffslide.repair.monkey_patch_description_tag_encoding()` を呼ぶ
  - ファイルを直すなら、バックアップを取ってから `--no-dry-run` を付けて実行する
