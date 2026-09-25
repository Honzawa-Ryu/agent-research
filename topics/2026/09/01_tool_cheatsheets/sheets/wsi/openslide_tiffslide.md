# OpenSlide / TiffSlide チートシート

> 対象: `openslide-python` 1.4 系 + `openslide-bin` 4.0、`tiffslide` 4.0 系（このマシンの `.venv` で確認）
> 公式: https://openslide.org/api/python/ ・ https://github.com/Bayer-Group/tiffslide
> 詳細編（応用・トラブル対応）: [openslide_tiffslide_deep.md](openslide_tiffslide_deep.md)

## 1. 基本概念

| 用語 | 意味 |
|:---|:---|
| WSI | ホールスライド画像。数万×数万ピクセルあり、解像度の違う画像を積み重ねたピラミッド構造になっている |
| level | ピラミッドの段。`level 0` が最高解像度で、数字が大きいほど粗い |
| downsample | level 0 に対する縮小率（例: 1, 4, 16, 32） |
| MPP | microns per pixel。1ピクセルが何 µm か。20x でおよそ 0.5、40x でおよそ 0.25 |
| objective power | スキャン時の対物倍率（20, 40 など） |
| associated images | サムネイルやラベルなど、本体とは別に入っている画像 |

**座標の決まり: `read_region` の `(x, y)` は、どの level を読むときも level 0 の座標で指定する。サイズ `(w, h)` は指定した level のピクセル数。**

## 2. OpenSlide と TiffSlide の違い

| | OpenSlide | TiffSlide |
|:---|:---|:---|
| 実装 | C ライブラリ（`openslide-bin` で pip から入る） | 純 Python（tifffile + zarr） |
| 対応形式 | svs, tif, ndpi, mrxs, scn, vms, bif, dcm など多い | TIFF 系が中心（svs, tif, ndpi など）。**mrxs には非対応** |
| 返り値 | PIL Image（RGBA） | PIL Image（RGB）、または `as_array=True` で NumPy |
| リモート読み込み | 不可 | fsspec 経由で S3 などから直接読める |
| API | — | OpenSlide とほぼ同じ（差し替えて使える） |

```python
import tiffslide as openslide      # この1行で既存のコードをほぼそのまま使える
```

## 3. 開く・情報を見る

```python
import openslide
# import tiffslide as openslide でも同じように書ける

slide = openslide.OpenSlide("slide.svs")      # with 文でも使える
print(slide.level_count)                      # 例: 4
print(slide.dimensions)                       # level 0 の (幅, 高さ)
print(slide.level_dimensions)                 # 全 level の (幅, 高さ)
print(slide.level_downsamples)                # 例: (1.0, 4.0, 16.0, 32.0)
print(dict(slide.properties))                 # メタデータ（文字列の辞書）
print(list(slide.associated_images))          # 例: ['thumbnail', 'label', 'macro']
slide.close()
```

### よく使うプロパティ

```python
mpp_x = float(slide.properties.get(openslide.PROPERTY_NAME_MPP_X, "nan"))   # 'openslide.mpp-x'
mag   = slide.properties.get(openslide.PROPERTY_NAME_OBJECTIVE_POWER)        # 'openslide.objective-power'
```

- TiffSlide にも同じ名前の定数があるが、キーは `tiffslide.mpp-x` のように違う。文字列で直接書かず、定数を使えば両方で動く
- MPP や倍率が入っていないスライドもある。`None` のときの処理を必ず書く
- ベンダー固有のキー: `aperio.MPP`, `aperio.AppMag`, `hamamatsu.SourceLens` など

## 4. 画像を読む

```python
# level 0 の座標 (x, y) から、level 1 で 512x512 を読む
region = slide.read_region((x, y), 1, (512, 512))   # PIL.Image (RGBA)
rgb = region.convert("RGB")

import numpy as np
arr = np.asarray(rgb)                                # (H, W, 3) uint8

# TiffSlide なら直接 NumPy で受け取れる
arr = slide.read_region((x, y), 1, (512, 512), as_array=True)
```

```python
thumb = slide.get_thumbnail((1024, 1024))            # アスペクト比を保ったまま縮小
label = slide.associated_images["label"]             # ラベル画像（個人情報に注意）
```

### 倍率・MPP から level を選ぶ

```python
target_mpp = 0.5                                     # 20x 相当
ds = target_mpp / mpp_x                              # 必要な縮小率
level = slide.get_best_level_for_downsample(ds)      # ds 以下で最も近い level
scale = ds / slide.level_downsamples[level]          # 残りはリサイズで合わせる

size_at_level = int(round(224 * scale))
patch = slide.read_region((x, y), level, (size_at_level, size_at_level)).convert("RGB")
patch = patch.resize((224, 224))
```

### level 間の座標変換

```python
ds = slide.level_downsamples[level]
x0, y0 = int(xl * ds), int(yl * ds)          # level の座標 → level 0 の座標
xl, yl = int(x0 / ds), int(y0 / ds)          # level 0 の座標 → level の座標
```

## 5. タイル単位で読む（DeepZoom）

```python
from openslide.deepzoom import DeepZoomGenerator

dz = DeepZoomGenerator(slide, tile_size=254, overlap=1, limit_bounds=True)
print(dz.level_count, dz.level_tiles[-1])    # 最後の level が最高解像度
tile = dz.get_tile(dz.level_count - 1, (col, row))
```

## 6. DataLoader で使うときの注意

```python
class PatchDataset(torch.utils.data.Dataset):
    def __init__(self, path, coords):
        self.path, self.coords = path, coords
        self.slide = None                      # ここでは開かない

    def __getitem__(self, i):
        if self.slide is None:                 # 各ワーカーで初めて使うときに開く
            self.slide = openslide.OpenSlide(self.path)
        x, y = self.coords[i]
        return np.asarray(self.slide.read_region((x, y), 0, (256, 256)).convert("RGB"))
```

- `OpenSlide` オブジェクトは pickle できない。`__init__` で開くと、`num_workers > 0` のときにエラーになったり、挙動がおかしくなったりする
- ワーカーごとに開けば、並列に読める

## 7. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| 背景が黒い・透明 | `read_region` は RGBA を返す。スライド外は透明 (0,0,0,0) になる。`.convert("RGB")` するか、白で合成する |
| 位置がずれる | `(x, y)` を level の座標で渡している。level 0 の座標で渡す |
| `OpenSlideUnsupportedFormatError` | 対応していない形式、またはファイルが壊れている。`openslide.OpenSlide.detect_format(path)` で確認する |
| `libopenslide` が見つからない | `openslide-bin` を入れる（`uv add openslide-bin`）。昔は apt で入れる必要があった |
| MPP が取れない | スキャナによっては入っていない。ベンダー固有のキーを見るか、既知の値を手で与える |
| mrxs を TiffSlide で開けない | TiffSlide は非対応。OpenSlide を使う |
| メモリが足りなくなる | level 0 全体を読もうとしている。タイルに分けて読む |
| 同じスライドでも倍率の見た目が違う | スキャナによって MPP が違う（40x で 0.25 と 0.23 など）。倍率ではなく MPP で揃える |
