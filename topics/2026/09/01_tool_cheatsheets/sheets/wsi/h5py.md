# h5py チートシート

> 対象: h5py 3.x（このマシンの `.venv` は 3.16）
> 公式: https://docs.h5py.org/
> 詳細編（応用・トラブル対応）: [h5py_deep.md](h5py_deep.md)

## 1. 基本概念

HDF5 は、1つのファイルの中にフォルダ構造を持てるバイナリ形式。

| 用語 | 意味 | Python での感覚 |
|:---|:---|:---|
| File | HDF5 ファイル本体 | ルートの Group |
| Group | フォルダ | `dict` |
| Dataset | N 次元配列 | `np.ndarray`（ただし必要な分だけディスクから読む） |
| Attribute | Group / Dataset に付けるメタデータ | 小さな `dict` |

## 2. 開くモード

| モード | 意味 |
|:---|:---|
| `"r"` | 読み取りのみ（既定） |
| `"r+"` | 読み書き。ファイルが存在している必要がある |
| `"w"` | 新規作成。**既存のファイルは消える** |
| `"w-"` / `"x"` | 新規作成。既存のファイルがあればエラー |
| `"a"` | あれば読み書き、なければ作成 |

## 3. 読む

```python
import h5py

with h5py.File("slide.h5", "r") as f:
    print(list(f.keys()))                  # 例: ['coords', 'features']
    f.visit(print)                         # 全体の構造を表示
    ds = f["features"]
    print(ds.shape, ds.dtype)              # 例: (12034, 1024) float32

    feats = ds[:]                          # 全部読んで np.ndarray にする
    first = ds[:100]                       # 一部だけ読む（ディスクから必要な分だけ）
    coords = f["coords"][()]               # [()] でも全部読める（スカラーの場合も含む）

    attrs = dict(f["coords"].attrs)        # 属性
    patch_size = f["coords"].attrs["patch_size"]
```

- `with` を抜けると `ds` は使えなくなる。中で `[:]` して NumPy にしておく
- 属性に文字列を入れると `bytes` で返ることがある。`.decode()` する

### 構造をすばやく確認する（CLI）

```bash
# h5ls / h5dump / h5repack は HDF5 の CLI ツール。このマシンには入っていない（apt の hdf5-tools などで入る）
h5ls -r slide.h5                 # 構造
h5dump -H slide.h5               # ヘッダ（形状・型・属性）だけ
python -c "import h5py; h5py.File('slide.h5').visititems(lambda n, o: print(n, o))"
```

## 4. 書く

```python
import numpy as np

with h5py.File("out.h5", "w") as f:
    f.create_dataset("features", data=feats)                       # 配列をそのまま保存
    f.create_dataset("coords", data=coords, compression="gzip")    # 圧縮する
    f["coords"].attrs["patch_size"] = 256
    f["coords"].attrs["level0_magnification"] = 40

    g = f.create_group("meta")                                     # グループ
    g.create_dataset("slide_id", data="slide_001")                 # 文字列
```

### 後から追記する（サイズが決まっていない場合）

```python
with h5py.File("out.h5", "a") as f:
    if "features" not in f:
        f.create_dataset("features", shape=(0, 1024), maxshape=(None, 1024),
                         dtype="float32", chunks=(256, 1024))
    ds = f["features"]
    n = ds.shape[0]
    ds.resize(n + len(batch), axis=0)
    ds[n:] = batch
```

### 主なオプション（`create_dataset`）

| オプション | 例 | 説明 |
|:---|:---|:---|
| `compression` | `"gzip"`, `"lzf"` | gzip は圧縮率が高く、lzf は速い |
| `compression_opts` | `4` | gzip のレベル（0〜9） |
| `chunks` | `True`, `(256, 1024)` | チャンクの形状。圧縮やリサイズには必須 |
| `maxshape` | `(None, 1024)` | `None` の軸は後から伸ばせる |
| `dtype` | `"float16"` | 容量を減らしたいときに |

## 5. 削除・名前変更

```python
with h5py.File("out.h5", "a") as f:
    del f["old"]                    # 削除（ファイルサイズは小さくならない）
    f.move("a", "b")                # 名前変更
```

削除した分の容量を戻すには、`h5repack in.h5 out.h5`（HDF5 の CLI ツール）で作り直すか、必要なデータだけを新しいファイルにコピーする。

## 6. DataLoader で使うときの注意

```python
class H5Dataset(torch.utils.data.Dataset):
    def __init__(self, path):
        self.path = path
        with h5py.File(path, "r") as f:
            self.n = f["features"].shape[0]
        self.f = None                          # ここでは開いたままにしない

    def __len__(self):
        return self.n

    def __getitem__(self, i):
        if self.f is None:                     # 各ワーカーで開く
            self.f = h5py.File(self.path, "r")
        return torch.from_numpy(self.f["features"][i])
```

- 開いたファイルハンドルは fork したワーカー間で共有できない。ワーカーごとに開く
- 小さいファイルなら、`__init__` で全部読んで NumPy で持つ方が簡単で速い

## 7. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| `unable to lock file` / `resource temporarily unavailable` | 別のプロセスが書き込み中、または NFS 上でロックがうまくいかない。`HDF5_USE_FILE_LOCKING=FALSE` を設定する |
| `"w"` で開いたらデータが消えた | `"w"` は上書きする。追記は `"a"` |
| 読み込みが遅い | ランダムアクセスとチャンク形状が合っていない、または gzip の展開が重い。ローカル SSD にコピーしてから読むのも有効 |
| 途中で kill したらファイルが壊れた | HDF5 は書き込み中に落ちると壊れやすい。一時ファイルに書いてから rename する |
| 文字列が `b'...'` になる | `bytes` で返っている。`.decode()` するか、`ds.asstr()[()]` を使う |
| `with` の外で Dataset を使ってエラー | ファイルが閉じている。中で `[:]` して取り出しておく |
