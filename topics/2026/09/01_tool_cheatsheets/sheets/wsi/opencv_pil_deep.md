# OpenCV / Pillow 詳細編（応用・トラブル対応）

> 基本編: [opencv_pil.md](opencv_pil.md)
> 対象バージョン: OpenCV 5.0.0（`opencv-python` / `opencv-python-headless` 5.0.0.93）、Pillow 12.3.0、scikit-image 0.26（このマシンの `wsi_preprocess/.venv` で確認）

```python
import cv2, PIL; print(cv2.__version__, PIL.__version__)
```

## Part 1. 応用・上級機能

### 1.1 染色の分離（HED）

```python
from skimage.color import rgb2hed, hed2rgb

hed = rgb2hed(patch_rgb)                    # (H, W, 3) float。チャンネルは H, E, DAB
h = hed[..., 0]                             # ヘマトキシリン（核）の濃さ
h_only = hed2rgb(np.stack([h, np.zeros_like(h), np.zeros_like(h)], axis=-1))   # H だけの画像に戻す
```

- 染色ベクトルは固定値（標準的な H&E）。スキャナや施設によって実際の色は違うので、定量には使いにくい。核の大まかな抽出やマスク作りに使う

### 1.2 色の正規化

#### Reinhard（LAB の平均と標準偏差を合わせる）

```python
def reinhard(src_rgb, ref_rgb):
    src = cv2.cvtColor(src_rgb, cv2.COLOR_RGB2LAB).astype(np.float32)
    ref = cv2.cvtColor(ref_rgb, cv2.COLOR_RGB2LAB).astype(np.float32)
    s_mu, s_sd = src.reshape(-1, 3).mean(0), src.reshape(-1, 3).std(0) + 1e-6
    r_mu, r_sd = ref.reshape(-1, 3).mean(0), ref.reshape(-1, 3).std(0)
    out = (src - s_mu) / s_sd * r_sd + r_mu
    return cv2.cvtColor(np.clip(out, 0, 255).astype(np.uint8), cv2.COLOR_LAB2RGB)
```

- 背景（白）の割合で平均が大きく変わる。組織マスクの内側だけで統計をとる方がよい
- uint8 の LAB は OpenCV 独自の範囲（L: 0〜255）。上のように uint8 → float → uint8 で往復するなら問題ない

#### Macenko（染色ベクトルを推定する）

```python
def macenko(img, Io=240, alpha=1, beta=0.15,
            HERef=np.array([[0.5626, 0.2159], [0.7201, 0.8012], [0.4062, 0.5581]]),
            maxCRef=np.array([1.9705, 1.0308])):
    h, w, _ = img.shape
    od = -np.log((img.reshape(-1, 3).astype(np.float64) + 1) / Io)       # 光学濃度
    od_hat = od[~np.any(od < beta, axis=1)]                               # 背景（薄いピクセル）を除く
    _, eigvecs = np.linalg.eigh(np.cov(od_hat.T))
    v = eigvecs[:, 1:3]                                                   # 上位 2 つの固有ベクトル
    that = od_hat @ v
    phi = np.arctan2(that[:, 1], that[:, 0])
    lo, hi = np.percentile(phi, alpha), np.percentile(phi, 100 - alpha)
    v1 = v @ np.array([np.cos(lo), np.sin(lo)])
    v2 = v @ np.array([np.cos(hi), np.sin(hi)])
    HE = np.array([v1, v2]).T if v1[0] > v2[0] else np.array([v2, v1]).T  # H を 1 列目に
    C = np.linalg.lstsq(HE, od.T, rcond=None)[0]                          # 各ピクセルの染色濃度
    C *= (maxCRef / np.percentile(C, 99, axis=1))[:, None]
    out = Io * np.exp(-HERef @ C)
    return np.clip(out.T.reshape(h, w, 3), 0, 255).astype(np.uint8)
```

- 背景がほとんどのパッチや、組織が少ないパッチでは推定が不安定になる。例外処理を入れ、失敗したら元の画像を返す
- パッチごとに推定すると、隣り合うパッチで色が変わる。スライド単位（サムネイルや代表パッチ）で推定して使い回す方が安定する
- 実運用では `torchstain` などの既存実装も選択肢になる（この `.venv` には入っていない）

### 1.3 コントラストの調整（CLAHE）

```python
lab = cv2.cvtColor(rgb, cv2.COLOR_RGB2LAB)
clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
lab[..., 0] = clahe.apply(lab[..., 0])              # 明るさのチャンネルだけに適用
out = cv2.cvtColor(lab, cv2.COLOR_LAB2RGB)
```

RGB の各チャンネルに別々に適用すると色が変わる。L チャンネルか、グレースケールに適用する。

### 1.4 形態学的処理の応用

| 操作 | 用途 |
|:---|:---|
| `MORPH_GRADIENT` | 輪郭線を取り出す（膨張 − 収縮） |
| `MORPH_TOPHAT` | 背景より明るい小さな構造を強調 |
| `MORPH_BLACKHAT` | 背景より暗い小さな構造（核など）を強調 |
| `iterations=n` | 同じ処理を n 回繰り返す。大きなカーネルより速い |

小さな領域を消す・穴を埋める:

```python
def remove_small(mask, min_area):
    n, lab, stats, _ = cv2.connectedComponentsWithStats((mask > 0).astype(np.uint8), connectivity=8)
    keep = np.zeros(n, bool); keep[1:] = stats[1:, cv2.CC_STAT_AREA] >= min_area
    return (keep[lab] * 255).astype(np.uint8)

def fill_holes(mask):
    inv = mask.copy()
    ff = np.zeros((mask.shape[0] + 2, mask.shape[1] + 2), np.uint8)
    cv2.floodFill(inv, ff, (0, 0), 255)          # 外側の背景を塗る（(0,0) が背景である前提）
    return mask | cv2.bitwise_not(inv)           # 塗られなかった背景 = 穴
```

### 1.5 接している核を分ける（距離変換 + watershed）

```python
dist = cv2.distanceTransform(mask, cv2.DIST_L2, 5)                  # 前景の各点から背景までの距離
_, sure_fg = cv2.threshold(dist, 0.5 * dist.max(), 255, cv2.THRESH_BINARY)
sure_fg = sure_fg.astype(np.uint8)
sure_bg = cv2.dilate(mask, np.ones((3, 3), np.uint8), iterations=3)
unknown = cv2.subtract(sure_bg, sure_fg)
_, markers = cv2.connectedComponents(sure_fg)
markers = markers + 1; markers[unknown == 255] = 0
markers = cv2.watershed(bgr, markers)                               # 境界は -1 になる
```

- `watershed` の入力は 3 チャンネルの uint8 画像と int32 のマーカー
- 本格的な核セグメンテーションには、学習済みモデル（HoVer-Net など）の方が精度が高い

### 1.6 輪郭の階層（穴のある組織）

```python
contours, hier = cv2.findContours(mask, cv2.RETR_CCOMP, cv2.CHAIN_APPROX_SIMPLE)
hier = hier[0]                           # (N, 4): [次, 前, 最初の子, 親]
outer = [i for i in range(len(contours)) if hier[i][3] == -1]      # 外側の輪郭
holes = {i: [j for j in range(len(contours)) if hier[j][3] == i] for i in outer}
```

| モード | 返す輪郭 |
|:---|:---|
| `RETR_EXTERNAL` | 一番外側だけ（穴は無視） |
| `RETR_CCOMP` | 外側と穴の 2 階層 |
| `RETR_TREE` | 入れ子をすべて |
| `RETR_LIST` | 階層なしですべて |

### 1.7 マスクとポリゴンの相互変換

```python
# マスク → ポリゴン（shapely）
from shapely.geometry import Polygon
polys = []
for i in outer:
    shell = contours[i][:, 0, :]
    if len(shell) < 3:
        continue
    hole_pts = [contours[j][:, 0, :] for j in holes[i] if len(contours[j]) >= 3]
    polys.append(Polygon(shell, hole_pts).buffer(0))      # buffer(0) で自己交差を直す

# 頂点を減らす
approx = cv2.approxPolyDP(contours[0], epsilon=2.0, closed=True)   # epsilon はピクセル単位の許容誤差

# ポリゴン → マスク
mask = np.zeros((H, W), np.uint8)
cv2.fillPoly(mask, [pts.astype(np.int32)], 255)          # 穴は後から 0 で fillPoly する
```

サムネイル上の輪郭を level 0 座標にする:

```python
scale = slide.dimensions[0] / thumb_w            # 横と縦で別々に計算すると誤差が減る
contour_l0 = (contours[i].astype(np.float64) * scale).round().astype(np.int64)
```

### 1.8 幾何変換とテンプレートマッチング

```python
small = cv2.pyrDown(img)                                           # ガウシアンでぼかして 1/2 に
M = cv2.getRotationMatrix2D((w / 2, h / 2), angle=15, scale=1.0)
rot = cv2.warpAffine(img, M, (w, h), borderMode=cv2.BORDER_REFLECT)   # 端を反射で埋める

res = cv2.matchTemplate(big_gray, tmpl_gray, cv2.TM_CCOEFF_NORMED)  # 小さな画像の位置を探す
_, score, _, (x, y) = cv2.minMaxLoc(res)                           # 最大値の位置 (x, y)
```

`warpAffine` の既定の境界は黒（0）。病理画像では `BORDER_REFLECT` か `borderValue=(255, 255, 255)` にする。

### 1.9 Pillow のモード

| モード | 中身 | NumPy の dtype |
|:---|:---|:---|
| `1` | 1 ビット（白黒） | bool |
| `L` | 8 ビットグレー | uint8 |
| `P` | パレット（インデックス + 色表） | uint8 |
| `RGB` / `RGBA` | 8 ビット × 3 / 4 | uint8 |
| `I;16` | 16 ビット整数 | uint16 |
| `I` | 32 ビット整数 | int32 |
| `F` | 32 ビット浮動小数 | float32 |

```python
Image.fromarray(np.uint16_array).mode        # 'I;16'
Image.fromarray(np.float32_2d).mode          # 'F'（3 チャンネルの float はエラー）

# ラベルマップを色付きで保存（値はそのまま残る）
lbl = Image.fromarray(label_uint8, mode="P")
lbl.putpalette([0,0,0, 255,0,0, 0,255,0] + [0] * (256 - 3) * 3)
lbl.save("label.png")                        # 読み戻すと np.asarray でラベル値が得られる
```

### 1.10 ImageOps

| 関数 | 動作 |
|:---|:---|
| `ImageOps.exif_transpose(img)` | EXIF の回転情報を画像に反映する（スマホ写真など） |
| `ImageOps.pad(img, (224, 224), color="white")` | 比率を保って縮小し、余白を塗る |
| `ImageOps.fit(img, (224, 224))` | 比率を保って中央を切り抜く |
| `ImageOps.contain(img, (224, 224))` | 比率を保って枠に収める（余白なし） |
| `ImageOps.autocontrast(img, cutoff=1)` | 上下 1% を切ってコントラストを伸ばす |
| `ImageOps.equalize(img)` | ヒストグラム平坦化 |

### 1.11 描画と重ね合わせ（Pillow）

```python
from PIL import ImageDraw

mask = Image.new("L", (W, H), 0)
ImageDraw.Draw(mask).polygon([tuple(p) for p in pts], fill=255, outline=255)

overlay = Image.new("RGBA", base.size, (255, 0, 0, 0))
overlay.putalpha(mask.point(lambda v: 90 if v else 0))   # マスク部分だけ半透明の赤
vis = Image.alpha_composite(base.convert("RGBA"), overlay).convert("RGB")
```

### 1.12 マルチページ TIFF

```python
from PIL import ImageSequence
with Image.open("stack.tif") as im:
    pages = [p.copy() for p in ImageSequence.Iterator(im)]
pages[0].save("out.tif", save_all=True, append_images=pages[1:], compression="tiff_deflate")

ok, mats = cv2.imreadmulti("stack.tif", flags=cv2.IMREAD_UNCHANGED)   # OpenCV で全ページ
```

- WSI の TIFF をこの方法で開かない。ページ全体を展開するのでメモリが足りなくなる（[openslide_tiffslide.md](openslide_tiffslide.md) を使う）

### 1.13 保存オプション

| 形式 | Pillow | OpenCV |
|:---|:---|:---|
| JPEG | `quality=95, subsampling=0, optimize=True` | `[cv2.IMWRITE_JPEG_QUALITY, 95]` |
| PNG | `compress_level=1`（速い）〜`9`（小さい） | `[cv2.IMWRITE_PNG_COMPRESSION, 1]` |
| WebP | `quality=90` / `lossless=True` | `[cv2.IMWRITE_WEBP_QUALITY, 90]` |
| TIFF | `compression="tiff_deflate"` / `"tiff_lzw"` | `[cv2.IMWRITE_TIFF_COMPRESSION, ...]` |

- `subsampling=0` は色の間引きをしない（4:4:4）。染色の色を保ちたいパッチで有効
- PNG は可逆なので `compress_level` は速度とサイズだけに影響する。大量に書き出すなら 1〜3 で十分

### 1.14 速度のための工夫

```python
# バイト列（zip やネットワークから）を直接デコード
arr = cv2.imdecode(np.frombuffer(data, np.uint8), cv2.IMREAD_COLOR)

# JPEG を縮小しながら読む（デコード自体が速くなる）
img = Image.open("big.jpg"); img.draft("RGB", (img.width // 4, img.height // 4)); img = img.convert("RGB")
bgr_half = cv2.imread("big.jpg", cv2.IMREAD_REDUCED_COLOR_2)

# 整数倍の縮小
small = img.reduce(4)

# コピーせずに NumPy にする（読み取り専用になる）
view = np.asarray(img)
```

| 設定 | 効果 |
|:---|:---|
| `cv2.setNumThreads(n)` | OpenCV 内部のスレッド数。DataLoader のワーカー内では 0〜1 |
| `cv2.useOptimized()` | SIMD 最適化が有効か（既定で True） |
| `cv2.ocl.setUseOpenCL(False)` | OpenCL を使わない。このビルドでは OpenCL が有効 |

- このビルドの SIMD: 基本 SSE3、実行時に AVX2 / AVX512 を選択（`cv2.getBuildInformation()` で確認）
- `pillow-simd` は Pillow を置き換える別パッケージ。Pillow 本体より版が古いことが多く、他のパッケージとの依存関係に注意

## Part 2. トラブル対応

### 2.1 `libGL.so.1: cannot open shared object file`

- **症状**: `import cv2` で `ImportError: libGL.so.1 ...`。コンテナや計算ノードで起きる
- **確認**:
```bash
python -c "import cv2" 2>&1 | tail -1
ldconfig -p | grep -E "libGL.so.1|libgthread"
pip list 2>/dev/null | grep -i opencv            # uv なら: uv pip list | grep -i opencv
```
- **原因**: GUI 付きの `opencv-python` は libGL などの GUI 用ライブラリに依存する。サーバの最小イメージには入っていない
- **対処**: `opencv-python-headless` だけにする（2.2 節）。GUI 版が必要な場合は `apt install libgl1 libglib2.0-0` でライブラリを入れる（コンテナのイメージに入れる）

### 2.2 `opencv-python` と `opencv-python-headless` が両方入っている

- **症状**: 片方を消したら `import cv2` が壊れた。挙動がインストール順で変わる
- **確認**:
```bash
ls .venv/lib/python3.12/site-packages | grep -i opencv
python -c "import cv2; print([l for l in cv2.getBuildInformation().splitlines() if 'GUI' in l])"
uv tree --invert --package opencv-python        # 何が GUI 版を要求しているか
```
- **原因**: 2 つは同じ `cv2/` ディレクトリに展開される。後から入れた方のファイルで上書きされ、片方を消すと共有のファイルも消える。**このマシンの `wsi_preprocess/.venv` は両方 5.0.0.93 が入っており、現在の `cv2` は GUI 版（`GUI: QT5`）になっている**
- **対処**: 両方をアンインストールしてから 1 つだけ入れ直す。依存パッケージが GUI 版を要求している場合があるので、`uv tree --invert` で確認してから決める（どちらに揃えるかは環境全体に影響するので、変更前に確認をとる）

### 2.3 16 ビットの TIFF が正しく読めない

- **症状**: 16 ビットの画像を読むと値が 0〜255 に潰れる、真っ黒・真っ白に見える
- **確認**:
```python
a = cv2.imread("x.tif", cv2.IMREAD_UNCHANGED); print(a.dtype, a.min(), a.max())
im = Image.open("x.tif"); print(im.mode)           # 'I;16' など
```
- **原因**: `cv2.imread` の既定（`IMREAD_COLOR`）は 8 ビットに変換する。16 ビットをそのまま表示すると、値の範囲が 0〜65535 のため暗く見える
- **対処**:
```python
a = cv2.imread("x.tif", cv2.IMREAD_UNCHANGED)       # 元の dtype とチャンネル数のまま
lo, hi = np.percentile(a, (1, 99))
vis = np.clip((a - lo) / (hi - lo) * 255, 0, 255).astype(np.uint8)   # 表示用に 8 ビットへ
```

### 2.4 色が合わない

- **症状**: 保存・表示した画像の色が、元の画像やビューアと違う
- **確認**:
```python
print(img.mode, img.info.get("icc_profile") is not None)   # Pillow
print(arr.shape, arr.dtype, arr[..., :3].mean(axis=(0, 1)))  # 赤と青の平均が入れ替わっていないか
```
- **原因**:
  - RGB と BGR の取り違え（基本編 7 節）
  - RGBA や `P` モードを `np.asarray` して、アルファやパレットのインデックスを色として扱っている
  - ICC プロファイルが無視されている
- **対処**: 読んだ直後に `.convert("RGB")` で揃える。RGBA は白背景と合成してから RGB にする。ICC は [openslide_tiffslide_deep.md](openslide_tiffslide_deep.md) の 1.3 節の方法で変換する

### 2.5 dtype や形状のエラー

- **症状**: `cv2.error: ... (-210:Unsupported format or combination of formats)`、`TypeError: Cannot handle this data type: (1, 1, 3), <f4`
- **確認**:
```python
print(x.dtype, x.shape, x.flags["C_CONTIGUOUS"])
```
- **原因**:
  - `findContours` に bool や int64 のマスクを渡している（uint8 の 1 チャンネルが必要。OpenCV 5.0 でも同じ）
  - `cvtColor` に float64 を渡している（uint8 / uint16 / float32 のみ）
  - `Image.fromarray` に float の 3 チャンネルを渡している
  - スライスや転置で非連続になった配列を、書き込み先として OpenCV に渡している
- **対処**:
```python
mask_u8 = (mask > 0).astype(np.uint8) * 255
img_f32 = img.astype(np.float32)
pil = Image.fromarray(np.clip(x * 255, 0, 255).astype(np.uint8))
arr = np.ascontiguousarray(arr)
```

### 2.6 `DecompressionBombError` / `DecompressionBombWarning`

- **症状**: 大きな画像を `Image.open` すると警告やエラーが出る
- **確認**:
```python
from PIL import Image
print(Image.MAX_IMAGE_PIXELS)                      # 既定 89478485（約 8,900 万画素）
```
- **原因**: 画素数が上限を超えると警告、上限の 2 倍を超えるとエラーになる。巨大な画像でメモリを使い果たす攻撃への対策
- **対処**: 信頼できる画像だけ `Image.MAX_IMAGE_PIXELS = None` にする。1 億画素を超えるような画像は、そもそも全体をメモリに展開しない方法（OpenSlide、pyvips）で扱う

### 2.7 日本語などを含むパスで読み書きできない

- **症状**: `cv2.imread` が `None` を返す、`cv2.imwrite` が `False` を返す。パスは正しい
- **確認**:
```python
import os; print(os.path.exists(path), path.isascii())
```
- **原因**: OpenCV のファイル入出力は、環境によっては ASCII 以外のパスを扱えない。失敗しても例外を出さない
- **対処**:
```python
img = cv2.imdecode(np.fromfile(path, np.uint8), cv2.IMREAD_COLOR)
ok, buf = cv2.imencode(".png", img); buf.tofile(out_path)
```
  - `cv2.imwrite` の返り値は必ず確認する

### 2.8 `findContours` の返り値の数が合わない

- **症状**: `ValueError: not enough values to unpack (expected 3, got 2)`
- **確認**:
```python
print(cv2.__version__, len(cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)))
```
- **原因**: OpenCV 3 系は `(image, contours, hierarchy)` の 3 つを返していた。4 系以降（このマシンの 5.0.0 も）は `(contours, hierarchy)` の 2 つ
- **対処**: 古いコードを直すか、どちらでも動くように書く
```python
res = cv2.findContours(mask, cv2.RETR_CCOMP, cv2.CHAIN_APPROX_SIMPLE)
contours, hier = res[-2], res[-1]
```

### 2.9 OpenCV 5 に上がって動かなくなった

- **症状**: 4.x で動いていたコードが、5.0 で `AttributeError` やエラーになる
- **確認**:
```bash
python -c "import cv2; print(cv2.__version__)"
python -c "import cv2; print(hasattr(cv2, 'NAME_TO_CHECK'))"
uv pip list | grep -i opencv
```
- **原因**: メジャーバージョンが上がると、関数や定数の削除・移動が起きることがある。このシートで使っている関数（`findContours`, `watershed`, `distanceTransform`, `createCLAHE`, `imreadmulti` など）は 5.0.0 にあることを確認した。それ以外の変更点は個別に確認が必要
- **対処**: エラーが出た関数を `hasattr` で確認し、公式の移行情報を調べる。切り分けのために 4.x に固定する（`opencv-python-headless<5`）のも手。どちらにするかは環境全体に影響するので確認をとってから変更する

### 2.10 DataLoader で CPU を使い切る・固まる

- **症状**: ワーカーを増やしても速くならない。CPU 使用率が 100% に張り付く。まれにワーカーが止まる
- **確認**:
```python
print(cv2.getNumThreads(), cv2.ocl.haveOpenCL(), cv2.ocl.useOpenCL())
```
```bash
top -H -p <PID>                                   # 1 プロセスが多数のスレッドを持っていないか
```
- **原因**: 各ワーカーの OpenCV がコア数分のスレッドを立て、ワーカー数 × スレッド数に膨らむ。fork 後の OpenCL の初期化で止まることもある
- **対処**:
```python
def worker_init_fn(_):
    cv2.setNumThreads(0)
    cv2.ocl.setUseOpenCL(False)
DataLoader(ds, num_workers=8, worker_init_fn=worker_init_fn)
```
  - NumPy / PyTorch のスレッドも同様に制限する（`OMP_NUM_THREADS=1` など）

### 2.11 サムネイルで作ったマスクが WSI とずれる

- **症状**: サムネイル上のマスクや輪郭を level 0 に戻すと、端の方ほどずれる
- **確認**:
```python
print(slide.dimensions, thumb.size)
print(slide.dimensions[0] / thumb.size[0], slide.dimensions[1] / thumb.size[1])   # 縦横で違わないか
```
- **原因**: `get_thumbnail` は比率を保って整数サイズに丸めるので、縦と横の縮小率がわずかに違う。1 つの縮小率で両方を戻すと、誤差がサムネイルの画素数倍に拡大される
- **対処**: 縮小率を縦横別々に計算する（1.7 節）。マスクをリサイズするときは `INTER_NEAREST` を使う

### 2.12 マスクを JPEG で保存したら値が崩れた

- **症状**: 0/255 のマスクやラベル画像を読み戻すと、境界付近に中間の値が出る
- **確認**:
```python
print(np.unique(np.asarray(Image.open("mask.jpg"))))
```
- **原因**: JPEG は非可逆圧縮。境界がぼけ、ラベル値も変わる
- **対処**: マスク・ラベルは PNG（または `P` モードの PNG、1.9 節）で保存する。JPEG で保存済みのものは、閾値処理（`> 127`）で二値に戻す

### 2.13 `Too many open files`

- **症状**: 大量の画像を読むループで `OSError: [Errno 24] Too many open files`
- **確認**:
```bash
ulimit -n
ls /proc/<PID>/fd | wc -l
```
- **原因**: `Image.open` は遅延読み込みで、画素を読むまでファイルを開いたままにする。`Image.open(p)` をリストにため込むと、ファイルが閉じられない
- **対処**:
```python
with Image.open(p) as im:
    img = im.convert("RGB")          # convert や load で読み込んでから with を抜ける
```
