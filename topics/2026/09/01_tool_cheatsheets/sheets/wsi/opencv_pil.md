# OpenCV / Pillow チートシート

> 対象: `opencv-python`（`cv2`）、Pillow（`PIL`）12 系
> 公式: https://docs.opencv.org/ ・ https://pillow.readthedocs.io/

## 1. まず押さえる違い

| | Pillow | OpenCV |
|:---|:---|:---|
| 画像の型 | `PIL.Image.Image` | `np.ndarray` |
| 色の並び | **RGB** | **BGR** |
| サイズの表し方 | `img.size` = (幅, 高さ) | `arr.shape` = (高さ, 幅, チャンネル) |
| 座標 | (x, y) | 配列は `[y, x]`、関数の引数は (x, y) |
| 得意なこと | 読み書き、簡単な変換、torchvision との連携 | フィルタ、形態学的処理、輪郭、色空間の変換 |

```python
import numpy as np, cv2
from PIL import Image

arr = np.asarray(pil_img)                               # PIL → NumPy (RGB)
pil_img = Image.fromarray(arr)                          # NumPy → PIL（uint8 の RGB を想定）
bgr = cv2.cvtColor(arr, cv2.COLOR_RGB2BGR)              # RGB → BGR
rgb = cv2.cvtColor(bgr, cv2.COLOR_BGR2RGB)              # BGR → RGB
```

## 2. 読む・書く

```python
# Pillow
img = Image.open("a.png").convert("RGB")      # RGBA やパレット画像を RGB に揃える
img.save("b.jpg", quality=95)
img.save("b.png")

# OpenCV
bgr = cv2.imread("a.png")                     # BGR の uint8。失敗すると None（例外は出ない）
gray = cv2.imread("a.png", cv2.IMREAD_GRAYSCALE)
cv2.imwrite("b.jpg", bgr, [cv2.IMWRITE_JPEG_QUALITY, 95])
```

## 3. リサイズ・切り出し・回転

```python
# Pillow
img.resize((224, 224), Image.Resampling.BILINEAR)
img.thumbnail((512, 512))                     # アスペクト比を保って縮小（その場で変更される）
img.crop((left, top, right, bottom))
img.rotate(90, expand=True)
img.transpose(Image.Transpose.FLIP_LEFT_RIGHT)

# OpenCV
cv2.resize(arr, (224, 224), interpolation=cv2.INTER_AREA)   # 引数は (幅, 高さ)
arr[y0:y1, x0:x1]                                           # 切り出し（NumPy のスライス）
cv2.rotate(arr, cv2.ROTATE_90_CLOCKWISE)
cv2.flip(arr, 1)                                            # 1: 左右, 0: 上下
```

| 補間 | 向いている用途 |
|:---|:---|
| `INTER_AREA` / `Resampling.BOX` | 縮小（エイリアシングが少ない） |
| `INTER_LINEAR` / `BILINEAR` | 一般的な拡大・縮小 |
| `INTER_CUBIC` / `BICUBIC` | 拡大（きれいだが遅い） |
| `INTER_NEAREST` / `NEAREST` | マスク・ラベル画像（値を混ぜない） |

## 4. 色空間

```python
hsv  = cv2.cvtColor(rgb, cv2.COLOR_RGB2HSV)      # H: 0〜179, S/V: 0〜255
gray = cv2.cvtColor(rgb, cv2.COLOR_RGB2GRAY)
lab  = cv2.cvtColor(rgb, cv2.COLOR_RGB2LAB)

img.convert("L")      # Pillow のグレースケール
img.convert("HSV")
```

OpenCV の H は 0〜179（360°の半分）で、一般的な 0〜360 とは違う。

## 5. 病理画像でよく使う処理

### 組織マスク（大津の二値化）

```python
hsv = cv2.cvtColor(thumb_rgb, cv2.COLOR_RGB2HSV)
sat = cv2.GaussianBlur(hsv[..., 1], (5, 5), 0)
_, mask = cv2.threshold(sat, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)   # 彩度の高いところを組織とみなす

kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (7, 7))
mask = cv2.morphologyEx(mask, cv2.MORPH_CLOSE, kernel)    # 小さな穴を埋める
mask = cv2.morphologyEx(mask, cv2.MORPH_OPEN, kernel)     # 小さなノイズを消す
tissue_ratio = (mask > 0).mean()
```

### 輪郭と連結成分

```python
contours, hierarchy = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
areas = [cv2.contourArea(c) for c in contours]
x, y, w, h = cv2.boundingRect(contours[0])
vis = cv2.drawContours(bgr.copy(), contours, -1, (0, 255, 0), 2)

n, labels, stats, centroids = cv2.connectedComponentsWithStats(mask)
```

### ぼけの検出（ラプラシアンの分散）

```python
gray = cv2.cvtColor(patch_rgb, cv2.COLOR_RGB2GRAY)
blur_score = cv2.Laplacian(gray, cv2.CV_64F).var()   # 小さいほどぼけている。閾値はデータで決める
```

### 描画

```python
cv2.rectangle(bgr, (x, y), (x + w, y + h), (0, 0, 255), 2)
cv2.putText(bgr, "label", (x, y - 5), cv2.FONT_HERSHEY_SIMPLEX, 0.6, (0, 0, 255), 1)

overlay = cv2.addWeighted(bgr, 0.7, heatmap_bgr, 0.3, 0)       # 半透明で重ねる
heatmap_bgr = cv2.applyColorMap((score * 255).astype(np.uint8), cv2.COLORMAP_JET)
```

## 6. 大きい画像を扱うとき

```python
Image.MAX_IMAGE_PIXELS = None     # 巨大な画像で DecompressionBombError が出るのを止める（信頼できる画像だけ）
```

- WSI 本体は Pillow / OpenCV で直接開かない。OpenSlide / TiffSlide で必要な部分だけ読む（[openslide_tiffslide.md](openslide_tiffslide.md)）

## 7. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| 色が青っぽい / 赤と青が逆 | RGB と BGR の取り違え。`cv2.imread` の結果を matplotlib で表示するときは `cvtColor` で RGB にする |
| `cv2.imread` が `None` を返す | パスが間違っている、または日本語のパス。`cv2.imdecode(np.fromfile(path, np.uint8), cv2.IMREAD_COLOR)` を使う |
| `resize` の結果が縦横逆 | `cv2.resize` の引数は (幅, 高さ)、`shape` は (高さ, 幅) |
| マスクの値が 0/255 以外になる | マスクを線形補間でリサイズしている。`INTER_NEAREST` を使う |
| `Image.fromarray` でエラー | dtype が float か、形状がおかしい。`uint8` の (H, W, 3) にする |
| `libGL.so.1` が見つからない | GUI 版の `opencv-python` が入っている。サーバでは `opencv-python-headless` を使うか、`libgl1` を apt で入れる |
| `cv2` の挙動がおかしい・import エラー | `opencv-python` と `opencv-python-headless` が両方入っていて衝突している。片方だけにする |
| DataLoader で遅い / CPU を使い切る | OpenCV の内部スレッドとワーカーが競合している。`cv2.setNumThreads(0)` を設定する |
