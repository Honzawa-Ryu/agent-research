# h5py 詳細編（応用・トラブル対応）

> 基本編: [h5py.md](h5py.md)
> 対象バージョン: h5py 3.16.0 + HDF5 2.0.0（このマシンの `wsi_preprocess/.venv` で確認。MPI 版ではない）

```python
import h5py; print(h5py.version.info)     # h5py と HDF5 本体のバージョン
```

## Part 1. 応用・上級機能

### 1.1 チャンクの形を決める

HDF5 はチャンク単位で読み書き・圧縮する。1 要素だけ読む場合でも、それを含むチャンク全体を読んで展開する。

| 読み方 | 向いているチャンク形状（例: `features` が (N, 1024) float32） |
|:---|:---|
| 行をまとめて読む（`ds[i:i+256]`） | `(256, 1024)` |
| 1 行ずつランダムに読む | `(1, 1024)` 〜 `(16, 1024)` |
| 列を読む（`ds[:, j]`） | `(N_small, 1)` のように縦長 |
| 全部まとめて読むだけ | 大きめ（`(4096, 1024)` など）か、チャンクなし（連続配置） |

```python
ds = f.create_dataset("features", shape=(n, 1024), dtype="float32",
                      chunks=(64, 1024))           # 1 チャンク 256 KiB
print(ds.chunks)                                   # None ならチャンクなし（連続配置）
for sl in ds.iter_chunks():                        # チャンク単位のスライスを順に返す
    block = ds[sl]
```

- 1 チャンクは数十 KiB〜1 MiB 程度が目安。小さすぎるとメタデータが増え、大きすぎるとランダムアクセスが遅くなる
- `chunks=True` は h5py が自動で決める。読み方が決まっているなら明示した方がよい
- チャンクの形は作成後に変えられない。変えるには作り直すか `h5repack -l`（1.12 節）

### 1.2 チャンクキャッシュ

展開済みのチャンクはファイルごとのキャッシュに保持される。キャッシュに収まらないと、同じチャンクを何度も展開することになる。

```python
f = h5py.File("feats.h5", "r",
              rdcc_nbytes=256 * 1024**2,   # キャッシュの容量（バイト）
              rdcc_nslots=100003,          # ハッシュのスロット数。チャンク数の 10〜100 倍の素数が目安
              rdcc_w0=0.75)                # 読み切ったチャンクを優先して捨てる度合い
print(f.id.get_access_plist().get_cache()) # (_, nslots, nbytes, w0)
```

- このマシン（HDF5 2.0.0）の既定値は 8 MiB・8191 スロット。1 チャンクがこれより大きいとキャッシュされない
- キャッシュはデータセットごとに確保される。多数のデータセットを同時に開くと、その分メモリを使う

### 1.3 圧縮フィルタ

| フィルタ | 指定 | 特徴 |
|:---|:---|:---|
| gzip | `compression="gzip", compression_opts=4` | 圧縮率が高い。展開は遅め。どの環境でも読める |
| lzf | `compression="lzf"` | 速いが圧縮率は低い。h5py 以外（C や MATLAB）では読めないことがある |
| shuffle | `shuffle=True` | バイトの並べ替え。float の圧縮率が上がる。gzip / lzf と組み合わせる |
| scaleoffset | `scaleoffset=3` | 小数点以下 3 桁に丸めて整数化する（非可逆） |
| fletcher32 | `fletcher32=True` | チェックサム。破損を検出できる |

```python
f.create_dataset("features", data=feats, chunks=(256, feats.shape[1]),
                 compression="gzip", compression_opts=4, shuffle=True)
print(f["features"].compression, f["features"].filter_names)
```

- 特徴量（float32）は gzip でも縮みにくい。容量を減らしたいなら `float16` にする方が確実に効く（半分になる）
- zstd や blosc を使うには `hdf5plugin` が必要（この `.venv` には入っていない）。読む側にも同じプラグインが必要になる

### 1.4 文字列

```python
# 可変長 UTF-8（既定）。Python の str はこれになる
f.create_dataset("names", data=["slide_001", "slide_002"], dtype=h5py.string_dtype())
names = f["names"].asstr()[:]            # str の配列として読む（asstr を付けないと bytes）

# 固定長（他のツールとの互換性が高い）
f.create_dataset("ids", data=np.array([b"A01", b"B02"]), dtype="S3")
```

| 保存の仕方 | 読んだときの型 |
|:---|:---|
| 可変長文字列のデータセット | `bytes`（`asstr()` を付けると `str`） |
| 固定長（`S3` など） | `bytes`（`np.bytes_`） |
| `attrs["x"] = "abc"` | `str` |
| `attrs["x"] = np.bytes_("abc")` | `np.bytes_` |

### 1.5 可変長の配列（スライドごとにパッチ数が違う）

```python
vl = h5py.vlen_dtype(np.float32)
ds = f.create_dataset("attn", shape=(n_slides,), dtype=vl)
ds[0] = np.random.rand(1200).astype(np.float32)
ds[1] = np.random.rand(3400).astype(np.float32)
print(ds[1].shape)                        # (3400,)
```

- 1 次元の配列しか入らない。(N, D) の特徴量を可変長で持つより、スライドごとにデータセットを分けるか、全体を連結してオフセットを別に保存する方が扱いやすい
- 可変長データは圧縮が効きにくく、読み込みも遅い

### 1.6 属性の大きさの上限

- 既定のファイル形式では、1 つの属性はおよそ 64 KiB まで。超えると `OSError: ... object header message is too large` になる（このマシンで確認）
- `h5py.File(..., libver="latest")` で作ったファイルなら大きな属性も入る。ただし古い HDF5 では読めない
- 大きなもの（座標の一覧、設定の JSON 全体など）はデータセットにする方が確実

### 1.7 飛び飛びの行を読む（fancy indexing）

h5py の fancy indexing には NumPy にない制約がある。

| 書き方 | 可否 |
|:---|:---|
| `ds[[1, 3, 7]]`（昇順・重複なし） | OK |
| `ds[[3, 1]]`、`ds[[1, 1]]` | `TypeError: Indexing elements must be in increasing order` |
| `ds[mask]`（bool 配列） | OK |
| `ds[[0, 2], [1, 3]]`（2 軸同時） | `TypeError: Only one indexing vector ...` |

```python
idx = np.array([7, 3, 3, 10])
uniq, inv = np.unique(idx, return_inverse=True)   # 昇順・重複なしにする
rows = ds[uniq][inv]                              # 読んでから元の順番に戻す
```

- 要素が多い fancy indexing は遅い。範囲が狭いなら、まとめてスライスで読んでから NumPy で選ぶ方が速い

### 1.8 用意したバッファに直接読む

```python
out = np.empty((1000, 1024), dtype=np.float32)
ds.read_direct(out, source_sel=np.s_[5000:6000], dest_sel=np.s_[0:1000])   # コピーを1回減らせる
half = ds.astype("float16")[:1000]                                         # 読みながら型を変える
```

### 1.9 リンクと仮想データセット（複数ファイルをまとめる）

```python
with h5py.File("all.h5", "w") as f:
    f["slide_001"] = h5py.ExternalLink("feats/slide_001.h5", "/features")   # 別ファイルを参照
    f["latest"] = h5py.SoftLink("/slide_001")                               # 名前の別名
```

スライドごとの `features` を 1 つの配列として見せる（仮想データセット）:

```python
paths = sorted(glob.glob("feats/*.h5"))
shapes = [h5py.File(p, "r")["features"].shape for p in paths]
n, d = sum(s[0] for s in shapes), shapes[0][1]

layout = h5py.VirtualLayout(shape=(n, d), dtype="float32")
off = 0
for p, (m, _) in zip(paths, shapes):
    layout[off:off + m] = h5py.VirtualSource(p, "features", shape=(m, d))
    off += m
with h5py.File("all_virtual.h5", "w") as f:
    f.create_virtual_dataset("features", layout, fillvalue=0)
```

- 元のファイルは相対パスで参照される。移動するときはまとめて動かす
- 元ファイルが見つからないと、その部分は `fillvalue` になる（エラーにならない）

### 1.10 書きながら別のプロセスで読む（SWMR）

```python
# 書く側
f = h5py.File("log.h5", "w", libver="latest")
ds = f.create_dataset("loss", shape=(0,), maxshape=(None,), chunks=(1024,), dtype="f4")
f.swmr_mode = True                       # これ以降、新しいデータセットやグループは作れない
ds.resize((n,)); ds[-1] = v; ds.flush()

# 読む側
f = h5py.File("log.h5", "r", libver="latest", swmr=True)
ds = f["loss"]; ds.refresh(); print(ds.shape)
```

- 書き手は 1 つだけ。NFS などのネットワークファイルシステムでは正しく動作しない
- 学習ログのような用途なら、CSV や TensorBoard の方が簡単

### 1.11 複数プロセスで書き込む

この h5py は MPI 版ではない（`h5py.get_config().mpi` が `False`）。**1 つのファイルに複数のプロセスが同時に書き込むことはできない。**

```python
# 各ワーカーは自分のファイルにだけ書く
def worker(slide):
    feats = extract(slide)
    tmp = f"out/{slide}.h5.tmp"
    with h5py.File(tmp, "w") as f:
        f.create_dataset("features", data=feats)
    os.replace(tmp, f"out/{slide}.h5")        # 書き終わってから名前を変える（途中のファイルを見せない）
```

- まとめたい場合は、全部終わってから 1 プロセスで結合するか、1.9 節の仮想データセットを使う
- 別ファイルへのコピーは `src.copy(src["features"], dst, name="slide_001/features")` でできる

### 1.12 dask で大きな配列を扱う

```python
import dask.array as da

f = h5py.File("big.h5", "r")                         # 計算が終わるまで開いたままにする
x = da.from_array(f["features"], chunks=f["features"].chunks or (4096, -1))
mean = x.mean(axis=0).compute()
```

- dask のチャンクは HDF5 のチャンクの整数倍にすると効率がよい
- マルチプロセスのスケジューラでは、h5py のオブジェクトは渡せない。スレッドのスケジューラ（既定）で使う

### 1.13 ファイルロックの制御

```python
f = h5py.File("x.h5", "r", locking=False)   # このファイルだけロックを使わない
```

| 方法 | 効果 |
|:---|:---|
| `locking=True` / `False` / `"best-effort"` | ファイルごとに指定。`best-effort` はロックが使えないファイルシステムでは無視する |
| 環境変数 `HDF5_USE_FILE_LOCKING=FALSE` | プロセス全体で無効にする |

ロックを切ると、同時書き込みの保護もなくなる。読み取り専用の用途に限る。

### 1.14 コマンドラインツール

このマシンには HDF5 のコマンドラインツールが入っていない（`h5dump` などが見つからない）。使う場合は `apt install hdf5-tools` や conda の `hdf5` で入れる。

| コマンド | 用途 |
|:---|:---|
| `h5stat file.h5` | 容量の内訳、空き領域の量 |
| `h5dump -H -p file.h5` | ヘッダ。`-p` でチャンクや圧縮の設定も表示 |
| `h5repack in.h5 out.h5` | 作り直し（削除した分の容量を戻す） |
| `h5repack -f GZIP=4 in.h5 out.h5` | 圧縮を変えて作り直す |
| `h5repack -l features:CHUNK=256x1024 in.h5 out.h5` | チャンク形状を変えて作り直す |
| `h5clear -s file.h5` | 異常終了で残った「書き込み中」フラグを消す（`libver="latest"` 形式のファイル） |

ツールがない場合、Python で同じことができる:

```python
with h5py.File("in.h5", "r") as src, h5py.File("out.h5", "w") as dst:
    for k in src:
        src.copy(src[k], dst, name=k)     # 属性も含めてコピー（使っていない領域は持ち込まない）
```

## Part 2. トラブル対応

### 2.1 NFS 上でロックエラーになる

- **症状**: `OSError: Unable to synchronously open file (unable to lock file, errno = 11 / 37, ...)` が出る。ローカルでは出ない
- **確認**:
```bash
df -hT /path/to/file.h5                           # nfs などか
lsof /path/to/file.h5                             # 同じノードで開いているプロセス
echo $HDF5_USE_FILE_LOCKING
```
- **原因**: NFS ではロックが使えないか、不安定。または本当に別のプロセスが書き込みモードで開いている
- **対処**:
  - 読むだけなら `HDF5_USE_FILE_LOCKING=FALSE` を設定するか、`locking=False` で開く
  - 書き込み中のプロセスがあるなら、そちらが終わるのを待つ。ロックを切って同時に書くと壊れる

### 2.2 kill した後にファイルが開けない

- **症状**: 書き込み中に kill・OOM・ノード障害があった。開くと `OSError: Unable to open file (bad object header version number)`、`file signature not found`、`file is already open for write` などが出る
- **確認**:
```python
print(h5py.is_hdf5("out.h5"))                     # False ならヘッダから壊れている
```
```bash
ls -l out.h5                                      # サイズが途中で止まっていないか
h5clear -s out.h5                                 # "already open for write" の場合（ツールがあれば）
```
- **原因**: HDF5 はメタデータを書き終える前に落ちると、ファイル全体が読めなくなることがある。`libver="latest"` のファイルは、書き込み中のフラグが残って開けなくなることがある
- **対処**:
  - フラグだけの問題なら `h5clear -s` で開けるようになる
  - 壊れた場合は、基本的に作り直す。部分的な復旧は期待しない
  - 予防として、一時ファイルに書いて `os.replace` で名前を変える（1.11 節）。長時間の追記では、区切りごとに別ファイルにする

### 2.3 ランダムアクセスが遅い

- **症状**: DataLoader で 1 行ずつ読むと、全部読むより何十倍も遅い
- **確認**:
```python
ds = f["features"]
print(ds.chunks, ds.compression, ds.shape, ds.dtype)
import time; t = time.time(); [ds[i] for i in np.random.randint(0, len(ds), 1000)]; print(time.time() - t)
```
- **原因**: チャンクが大きく圧縮されていると、1 行読むたびにチャンク全体を展開する。チャンクキャッシュ（既定 8 MiB）に収まらないと、同じチャンクを何度も展開する
- **対処**:
  - ファイルが小さい（メモリに収まる）なら、最初に `ds[:]` で全部読む
  - チャンクを小さくするか圧縮を外して作り直す（1.1 節、1.14 節）
  - `rdcc_nbytes` を大きくする（1.2 節）
  - インデックスをソートして、近い行をまとめて読む

### 2.4 fancy indexing で TypeError

- **症状**: `TypeError: Indexing elements must be in increasing order` や `Only one indexing vector or array is currently allowed`
- **確認**:
```python
idx = np.asarray(idx)
print(np.all(np.diff(idx) > 0))                  # 昇順かつ重複なしか
```
- **原因**: h5py の fancy indexing は、昇順・重複なしの 1 軸だけに対応している（1.7 節）
- **対処**: `np.unique(..., return_inverse=True)` で並べ替えて読み、元の順に戻す。2 軸を選ぶなら、1 軸目で読んでから NumPy で 2 軸目を選ぶ

### 2.5 文字列が bytes で返ってくる

- **症状**: `b'slide_001'` のように返り、`str` との比較が常に False になる
- **確認**:
```python
ds = f["names"]
print(ds.dtype, h5py.check_string_dtype(ds.dtype))   # encoding と length（None なら可変長）
```
- **原因**: h5py 3 では、文字列のデータセットは `bytes` で返る。固定長の文字列や、他のツールで書いたファイルの属性も `bytes` になる
- **対処**: データセットは `ds.asstr()[:]`。属性や個別の値は `.decode()` する。混在する場合は次の関数でそろえる
```python
def to_str(v):
    return v.decode() if isinstance(v, (bytes, np.bytes_)) else str(v)
```

### 2.6 削除したのにファイルが小さくならない

- **症状**: `del f["old"]` や上書きを繰り返したのに、ファイルサイズが変わらない、または増え続ける
- **確認**:
```bash
h5stat -S file.h5                                  # 空き領域の量（ツールがあれば）
```
```python
sizes = []
f.visititems(lambda n, o: sizes.append(o.id.get_storage_size()) if isinstance(o, h5py.Dataset) else None)
print(sum(sizes) / 1e6, "MB used /", os.path.getsize("file.h5") / 1e6, "MB file")
```
- **原因**: HDF5 は削除した領域をファイルの末尾から切り詰めない。既定では空き領域の情報もファイルを閉じると失われ、再利用されない
- **対処**: `h5repack` か、1.14 節の Python のコピーで作り直す。何度も書き換えるデータは HDF5 に向かない。書き換えが前提なら、上書きではなく新しいファイルを作る

### 2.7 `Unable to open object` / `KeyError`

- **症状**: `KeyError: "Unable to synchronously open object (object 'features' doesn't exist)"`
- **確認**:
```python
f.visit(print)                                     # 実際の名前と階層
print("features" in f, list(f.keys()))
```
- **原因**: 名前の違い（`feats` と `features`、先頭の `/` の有無、グループの階層）。外部リンクや仮想データセットの参照先が見つからない。書き込み途中のファイルを読んでいる
- **対処**: 実際の構造を見て名前を合わせる。外部リンクなら `f.get("name", getlink=True)` でリンク先のパスを確認する

### 2.8 DataLoader で読むと壊れたデータやエラーになる

- **症状**: `num_workers > 0` にすると、ランダムに `OSError`、読み込みの値がおかしい、ワーカーが止まる
- **確認**:
```python
# Dataset の __init__ で h5py.File を開いたまま self に持っていないか
print(type(dataset.f) if hasattr(dataset, "f") else "no handle")
```
- **原因**: 親プロセスで開いたハンドルを fork した子で使うと、HDF5 内部の状態が共有されて壊れる。`spawn` では pickle できずにエラーになる
- **対処**: 基本編 6 節のように、ワーカー内で最初に使うときに開く。書き込みは DataLoader のワーカーでは行わず、メインプロセスか、ワーカーごとの別ファイルに書く（1.11 節）

### 2.9 複数ジョブで同じファイルに書いたら壊れた

- **症状**: Slurm の配列ジョブなどで同じ h5 に追記したら、データが欠ける、開けなくなる
- **確認**:
```bash
grep -n "h5py.File(.*['\"]a['\"]" *.py        # 共有ファイルを "a" で開いていないか
```
- **原因**: HDF5（非 MPI 版）は、同時に 1 つの書き手しか想定していない。ロックがない NFS では、同時に開けてしまい、メタデータが壊れる
- **対処**: ジョブごと（スライドごと）に別ファイルに書き、最後に 1 プロセスでまとめる（1.9 節、1.11 節）

### 2.10 `"w"` で開いてしまい中身が消えた

- **症状**: 追記のつもりが、以前のデータセットがなくなっている
- **確認**:
```bash
grep -n "h5py.File(" *.py | grep "\"w\"\|'w'"
```
- **原因**: `"w"` は既存のファイルを空にして作り直す
- **対処**: 追記は `"a"`、既存を上書きしたくない新規作成は `"x"`（あればエラー）を使う。消えたデータは復元できないので、重要なファイルは読み取り専用のコピーを残しておく

### 2.11 圧縮したファイルが他の環境で読めない

- **症状**: 別のマシンや他の言語で開くと `required filter ... is not registered` のようなエラーが出る
- **確認**:
```python
print(f["features"].filter_names)                 # 使われているフィルタ
```
- **原因**: `lzf` は h5py に組み込まれたフィルタで、素の HDF5（C、MATLAB、R など）にはない。`hdf5plugin` の zstd / blosc も同じで、読む側にプラグインが必要
- **対処**: 共有するファイルは `gzip`（+ `shuffle`）にする。すでにあるファイルは `h5repack -f GZIP=4` か Python のコピーで書き直す

### 2.12 float16 で保存したら値がおかしい

- **症状**: 読み戻した特徴量に `inf` が混じる、または小さい値が 0 になる
- **確認**:
```python
x = f["features"][:]
print(x.dtype, np.isinf(x).sum(), np.abs(x).max())
```
- **原因**: float16 の最大値は約 65504、精度は 3 桁程度。元のデータにこれを超える値があると `inf` になる
- **対処**: 保存前に値の範囲を確認する。範囲を超えるなら float32 のまま圧縮するか、正規化してから float16 にする。学習時には `astype("float32")` で読む
