# NumPy 詳細編（応用・トラブル対応）

> 基本編: [numpy.md](numpy.md)
> 対象バージョン: NumPy 2.x（動作確認は 2.3.5。2.4.1 の環境もある）

## Part 1. 応用・上級機能

### 1.1 einsum のパターン

添字で「どの軸を残し、どの軸で和をとるか」を書く。`->` の右に出てこない添字は和をとられる。

| やりたいこと | einsum | 同じ意味の書き方 |
|:---|:---|:---|
| 行列積 | `"ik,kj->ij"` | `A @ B` |
| バッチ行列積 | `"bik,bkj->bij"` | `A @ B`（3次元） |
| 行ごとの内積 | `"ij,ij->i"` | `(A * B).sum(1)` |
| 全ペアの内積 | `"nd,md->nm"` | `A @ B.T` |
| 対角成分 | `"ii->i"` | `np.diag(A)` |
| トレース | `"ii->"` | `np.trace(A)` |
| 外積 | `"i,j->ij"` | `np.outer(a, b)` |
| 軸の入れ替え | `"hwc->chw"` | `a.transpose(2, 0, 1)` |
| 二次形式 x^T S x（行ごと） | `"ni,ij,nj->n"` | `((X @ S) * X).sum(1)` |

```python
np.einsum("ni,ij,nj->n", X, S, X, optimize=True)   # 3つ以上の配列は optimize=True で計算順を最適化
```

### 1.2 高度なインデックスの規則

整数配列（ファンシー）とスライスを混ぜると、結果の軸の並びが直感と違うことがある。

```python
a = np.zeros((5, 6, 7))
a[:, [0, 1], [0, 1]].shape     # (5, 2)   配列インデックスが隣り合う → その位置に入る
a[[0, 1], :, [0, 1]].shape     # (2, 6)   間にスライスがある → 配列由来の軸が先頭に来る
a[np.ix_([0, 1], [0, 1, 2], [0])].shape   # (2, 3, 1)  各軸で独立に選ぶ（直積）
```

| 書き方 | 意味 |
|:---|:---|
| `m[[0, 2], [1, 3]]` | 要素 (0,1) と (2,3) の2個（ペアで選ぶ） |
| `m[np.ix_([0, 2], [1, 3])]` | 2×2 の部分行列（行と列を独立に選ぶ） |
| `m[[0, 2]][:, [1, 3]]` | 同じく 2×2（ただし2回コピーされる） |

`argsort` などの結果で各行を並べ替えるときは `take_along_axis` を使う。

```python
idx = np.argsort(scores, axis=1)[:, ::-1][:, :k]    # 行ごとの上位 k 個の位置
topk = np.take_along_axis(scores, idx, axis=1)       # (N, k)
np.put_along_axis(out, idx, 1, axis=1)               # 同じ位置に書き込む
```

### 1.3 パッチ・タイルの切り出し（stride tricks）

```python
from numpy.lib.stride_tricks import sliding_window_view

img = np.zeros((H, W, 3), dtype=np.uint8)

# 重なりありのパッチ（ビューなのでメモリは増えない）
win = sliding_window_view(img, (256, 256), axis=(0, 1))   # (H-255, W-255, 3, 256, 256)
patches = win[::128, ::128]                                # stride 128 で間引く
# 窓の軸は末尾に付く。HWC に戻すなら:
patches = patches.transpose(0, 1, 3, 4, 2)                 # (nh, nw, 256, 256, 3)

# 重なりなしのタイル（H, W が p で割り切れる場合）
p = 256
tiles = img.reshape(H // p, p, W // p, p, 3).transpose(0, 2, 1, 3, 4)   # (nh, nw, p, p, 3)
tiles = tiles.reshape(-1, p, p, 3)                                     # (N, p, p, 3) ここでコピーが起きる
```

- `sliding_window_view` の結果は既定で書き込み不可（`writeable=False`）。
- `np.lib.stride_tricks.as_strided` は形状と strides を自由に指定できるが、範囲外のメモリを読んでもエラーにならない。`sliding_window_view` で済むならそちらを使う。

### 1.4 メモリレイアウト（C 順・F 順・連続性）

```python
x = np.arange(12).reshape(3, 4)
x.strides                       # (32, 8)  1つ隣の行へは 32 バイト、列へは 8 バイト
x.flags["C_CONTIGUOUS"]         # True
x.T.flags["C_CONTIGUOUS"]       # False（転置はビューで、F 順になる）
x[:, ::2].flags["C_CONTIGUOUS"] # False（間引きスライス）

y = np.ascontiguousarray(x.T)   # C 連続なコピーを作る（すでに連続ならコピーしない）
np.asfortranarray(x)            # F 順
```

C 連続が必要になる場面:
- `torch.from_numpy` 後に `.view()` する、C 拡張や Cython に渡す、`np.frombuffer` / `tobytes` でバイト列にする
- 非連続な配列に対する大きな集約は遅くなることがある。何度も使うなら一度 `ascontiguousarray` しておく

### 1.5 ビューとコピーの見分け方

```python
np.shares_memory(a, b)          # 正確に判定（遅いことがある）
np.may_share_memory(a, b)       # 範囲が重なるかだけ見る（速い、偽陽性あり）
b.base is a                     # b が a のビューか（ビューのビューだと base は大元になる）
```

| 操作 | 結果 |
|:---|:---|
| 基本スライス `a[1:5]`, `a[::2]`, `a[..., 0]` | ビュー |
| `reshape`, `T`, `transpose`, `squeeze`, `a[None]` | 基本はビュー（無理ならコピー） |
| `ravel` | できればビュー、`flatten` は常にコピー |
| ファンシー・ブールインデックス | コピー |
| `astype`（型が同じでも） | 既定でコピー（`copy=False` で不要なら省略） |
| `np.asarray(a)` | 既に ndarray ならコピーしない |
| `np.array(a)` | 既定でコピー |

### 1.6 ループを置き換えるベクトル化

```python
# 同じ位置への加算（a[idx] += 1 は重複した idx を1回しか数えない）
np.add.at(counts, idx, 1)
np.maximum.at(best, idx, vals)          # 位置ごとの最大値

# ラベルごとの合計・平均（groupby 相当）
sums = np.bincount(labels, weights=x, minlength=K)
cnts = np.bincount(labels, minlength=K)
means = sums / np.maximum(cnts, 1)

# 任意の値をラベル番号に変換（逆引き）
uniq, inv = np.unique(keys, return_inverse=True)   # keys == uniq[inv]
res = np.unique_inverse(keys)                      # NumPy 2.0+: res.values, res.inverse_indices
res = np.unique_counts(keys)                       # res.values, res.counts

# 区間への振り分け
np.digitize(x, bins)                    # bins[i-1] <= x < bins[i] なら i
np.searchsorted(sorted_arr, x)          # sorted_arr に x を挿入する位置（二分探索）

# 所属判定
np.isin(slide_ids, test_ids)            # in1d は非推奨
```

`np.add.at` は `bincount` より遅い。1次元で和をとるだけなら `bincount` を使う。

### 1.7 型の昇格（NEP 50）

NumPy 2 では Python の `int` / `float` は「弱い型」として扱われ、配列の dtype が優先される。

| 式 | 結果の dtype | 補足 |
|:---|:---|:---|
| `np.float32(3) + 3.0` | float32 | 1.x では float64 だった |
| `np.array([1], np.int32) + np.int64(1)` | int64 | NumPy スカラーは強い型 |
| `np.array([1], np.float32) * np.float64(2)` | float64 | 同上 |
| `np.array([1], np.int8) + 1.5` | float64 | 整数 + Python float は既定の float |
| `np.array([200], np.uint8) + 300` | OverflowError | Python int が dtype の範囲外 |
| `np.array([200], np.uint8) + np.array([100], np.uint8)` | uint8（値は 44） | 警告なしで桁あふれ |
| `np.result_type(np.int64, np.uint64)` | float64 | 符号あり・なしの 64bit は float になる |

```python
np.result_type(a, b)            # 演算結果の dtype を事前に確認する
```

### 1.8 浮動小数点の精度

```python
np.finfo(np.float16)            # max=65504, 有効桁は約3桁
np.finfo(np.float32).eps        # 約 1.19e-07
np.float32(16777216) + np.float32(1)   # 16777216.0（2^24 を超える整数は表せない）

with np.errstate(over="raise", invalid="raise"):   # 範囲内だけ警告をエラーにする
    y = x.astype(np.float16)
```

| 注意点 | 対処 |
|:---|:---|
| float16 は 65504 を超えると inf | 保存前に `np.abs(x).max()` を確認。特徴量の保存は float32 が無難 |
| float32 の大量の和は誤差がたまる | `x.sum(dtype=np.float64)` |
| float32 でのインデックス・ID | 1677万を超えると丸められる。ID は int64 で持つ |
| `np.log(0)`, `0/0` | `-inf`, `nan`。`np.log1p`, `np.clip(x, eps, None)` |

### 1.9 乱数の並列化（SeedSequence / spawn）

ワーカーごとに `default_rng(seed + i)` とするより、`spawn` で独立な系列を作るほうが安全。

```python
ss = np.random.SeedSequence(42)
rngs = [np.random.default_rng(s) for s in ss.spawn(n_workers)]

rng = np.random.default_rng(42)
child_rngs = rng.spawn(n_workers)             # Generator から直接作る

# Slurm array job で task ごとに独立な乱数
task = int(os.environ["SLURM_ARRAY_TASK_ID"])
rng = np.random.default_rng([task, 42])       # シードに整数のリストを渡せる
```

`ss.entropy` を記録しておけば、シード未指定で作った系列も後で再現できる。

### 1.10 大きな配列をディスク上で扱う（memmap）

```python
from numpy.lib.format import open_memmap

# 先に .npy ファイルを確保し、少しずつ書き込む（全体をメモリに載せない）
out = open_memmap("feats.npy", mode="w+", dtype=np.float32, shape=(n_patches, 1536))
for i, batch in enumerate(loader):
    out[i * bs:(i + 1) * bs] = batch
out.flush(); del out

feats = np.load("feats.npy", mmap_mode="r")   # 読み込み側（読み取り専用）
sub = np.asarray(feats[idx])                   # 必要な部分だけメモリに載る
```

| 方法 | ヘッダ | 備考 |
|:---|:---|:---|
| `open_memmap` | .npy 形式 | `np.load(mmap_mode=)` でそのまま読める。こちらを推奨 |
| `np.memmap` | なし（生バイト列） | dtype と shape を別途覚えておく必要がある |

- ネットワーク FS 上の memmap をランダムアクセスすると遅い。可能ならローカル SSD（`$TMPDIR` など）にコピーしてから使う。
- `mmap_mode="r"` の配列に書き込むと `ValueError: assignment destination is read-only`。

### 1.11 線形代数の数値安定性

| やりたいこと | 推奨 | 避けたい書き方 |
|:---|:---|:---|
| Ax = b を解く | `np.linalg.solve(A, b)` | `np.linalg.inv(A) @ b` |
| 最小二乗 | `np.linalg.lstsq(X, y, rcond=None)` | `inv(X.T @ X) @ X.T @ y` |
| 特異に近い行列の逆 | `np.linalg.pinv(A, rcond=...)` | `inv` |
| 対称行列の固有値 | `np.linalg.eigh`（実数・昇順） | `eig`（複素数が混ざることがある） |
| 共分散の Cholesky | `np.linalg.cholesky(C + 1e-6 * np.eye(d))` | ジッタなしで失敗 |

```python
np.linalg.cond(A)                             # 条件数。1e12 を超えたら要注意（float64 の場合）
coef, res, rank, sv = np.linalg.lstsq(X, y, rcond=None)
rank < X.shape[1]                              # True なら列が線形従属
```

### 1.12 BLAS のスレッド数

`@`, `linalg.*`, `einsum(optimize=True)` の一部は BLAS（OpenBLAS / MKL）を使い、既定でノードの全コアを使おうとする。

```python
np.show_config()                  # どの BLAS とリンクしているか

from threadpoolctl import threadpool_info, threadpool_limits
threadpool_info()                 # 現在のスレッド数（num_threads）
with threadpool_limits(limits=4, user_api="blas"):
    Z = X @ W
```

環境変数で制限する場合は **`import numpy` より前に** 設定する（sbatch スクリプトで export するのが確実）。

```bash
export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=$OMP_NUM_THREADS
export MKL_NUM_THREADS=$OMP_NUM_THREADS
```

## Part 2. トラブル対応

### 2.1 MemoryError / ジョブが OOM で落ちる

- **症状**: `numpy._core._exceptions._ArrayMemoryError: Unable to allocate 74.5 GiB ...`、または Slurm で `OUT_OF_MEMORY` / `oom-kill`。
- **確認**:
```python
print(A.shape, B.shape, A.dtype)
n, m, d = len(A), len(B), A.shape[1]
print("中間配列 GB:", n * m * d * A.itemsize / 1e9)      # A[:, None] - B[None] の大きさ
print("結果 GB:", n * m * 8 / 1e9)
```
```bash
sacct -j <jobid> --format=JobID,MaxRSS,ReqMem,State
```
- **原因**: ブロードキャストで (N, M, D) の中間配列ができる、float64 への暗黙の変換、コピーの連鎖（`astype`、ファンシーインデックス）。
- **対処**:
  - 距離は展開式で計算して中間配列を作らない: `d2 = (A**2).sum(1)[:, None] + (B**2).sum(1)[None] - 2 * A @ B.T`
  - 行をチャンクに分けて処理し、結果だけ `open_memmap` に書く
  - `float32` のまま計算する（`astype(np.float64)` しない）
  - 大きな入力は `np.load(..., mmap_mode="r")`

### 2.2 整数の桁あふれが警告なしで起きる

- **症状**: 画像の差分や積の結果が小さすぎる・負になる。エラーも警告も出ない。
- **確認**:
```python
print(a.dtype, b.dtype, np.result_type(a, b))
print(np.iinfo(a.dtype))                      # 表せる範囲
r = a.astype(np.int64) - b.astype(np.int64)
print((r < np.iinfo(a.dtype).min).any(), (r > np.iinfo(a.dtype).max).any())
```
- **原因**: 配列どうしの整数演算は桁あふれしても警告を出さない（`uint8` 同士で 200 + 100 = 44）。NumPy スカラー同士の演算では `RuntimeWarning: overflow` が出るので、配列でだけ気づかないことが多い。
- **対処**: 演算前に `astype(np.int32)` / `astype(np.float32)`。`np.clip(r, 0, 255).astype(np.uint8)` で戻す。`a + 300` のように範囲外の Python int を足すと NumPy 2 では `OverflowError` になるので、これも同様に型を広げる。

### 2.3 float16 で保存した特徴量に inf が出る

- **症状**: 読み込んだ特徴量で学習すると loss が nan、`StandardScaler` が `Input contains infinity` で止まる。
- **確認**:
```python
x = np.load("feats.npy", mmap_mode="r")
print(x.dtype, np.isinf(x).sum(), np.isnan(x).sum())
print(np.abs(np.asarray(orig, dtype=np.float32)).max())   # 元の最大値が 65504 を超えていないか
```
- **原因**: float16 の上限は 65504。それを超える値は `astype(np.float16)` で inf になる（`RuntimeWarning: overflow encountered in cast` が出るが見落としやすい）。二乗や分散の計算を float16 のまま行った場合も同様。
- **対処**: 保存は float32 にする。容量を減らしたい場合は、値域を確認してから float16 にし、計算時は `astype(np.float32)` してから使う。`np.errstate(over="raise")` で変換時にエラーにできる。

### 2.4 意図しないブロードキャスト

- **症状**: 結果の形が (N, N) になる、メモリが急に増える、平均値がおかしい。エラーは出ない。
- **確認**:
```python
print(y_true.shape, y_pred.shape)             # (N,) と (N, 1) の組み合わせが典型
print(np.broadcast_shapes(y_true.shape, y_pred.shape))
```
- **原因**: (N,) と (N, 1) を演算すると (N, N) に広がる。モデル出力が (N, 1) のまま残っていることが多い。
- **対処**: `y_pred = y_pred.reshape(-1)` か `squeeze(-1)`。関数の入口で `assert a.shape == b.shape` を書く。

### 2.5 元の配列が書き換わる / 書き換えたはずが反映されない

- **症状**: (a) 関数内で配列を変更したら呼び出し元の配列も変わった。(b) `a[idx][mask] = 0` としたのに `a` が変わらない。
- **確認**:
```python
print(np.shares_memory(a, b), b.base is a)
print(b.flags["OWNDATA"])                     # False ならビュー
```
- **原因**: (a) スライスや `reshape` の結果はビュー。(b) ファンシーインデックス `a[idx]` がコピーを返し、そのコピーに代入している。
- **対処**: (a) 変更する前に `.copy()`。関数の入口で `x = np.array(x, copy=True)` にしておく。(b) 1回のインデックスにまとめる: `sel = idx[mask[...]]; a[sel] = 0`、または `np.where` で新しい配列を作る。

### 2.6 `assignment destination is read-only`

- **症状**: `ValueError: assignment destination is read-only`。
- **確認**:
```python
print(a.flags["WRITEABLE"], type(a), a.base is not None)
```
- **原因**: `np.load(mmap_mode="r")` の配列、`sliding_window_view` / `broadcast_to` の結果、pandas 3.0 の `Series.to_numpy()`（Copy-on-Write）、`torch` から共有した配列など。
- **対処**: 書き換える必要があるなら `a = a.copy()`。memmap を更新したい場合は `mmap_mode="r+"`。

### 2.7 クラスタで計算が遅い・CPU 使用率が異常（BLAS の過剰スレッド）

- **症状**: `--cpus-per-task=4` なのに `top` で CPU 使用率が数千 %、もしくは並列化したら逆に遅くなった。
- **確認**:
```python
import os
from threadpoolctl import threadpool_info
print(os.environ.get("SLURM_CPUS_PER_TASK"), os.cpu_count(), len(os.sched_getaffinity(0)))
print([(i["internal_api"], i["num_threads"]) for i in threadpool_info()])
```
- **原因**: BLAS がノード全体のコア数でスレッドを立てる。さらに `multiprocessing` / `joblib` / DataLoader のワーカーごとに BLAS スレッドが立つと、ワーカー数 × スレッド数に膨らむ。
- **対処**: sbatch で `OMP_NUM_THREADS` などを `SLURM_CPUS_PER_TASK` に合わせて export（1.12 参照）。プロセス並列するときは各ワーカーを 1 スレッドにする（`threadpool_limits(1)`）。

### 2.8 NumPy 2 との ABI 不一致

- **症状**: import 時に `A module that was compiled using NumPy 1.x cannot be run in NumPy 2.x as it may crash.`、`ImportError: numpy.core.multiarray failed to import`、`_ARRAY_API not found` など。
- **確認**:
```bash
uv pip list | grep -iE "numpy|pandas|scipy|scikit|torch|opencv|numba"
python -c "import numpy; print(numpy.__version__)"
python -X importtime -c "import 問題のパッケージ" 2>&1 | tail   # どこで止まるか
```
- **原因**: NumPy 1.x 向けにビルドされた C 拡張（古い opencv、numba、pyarrow、自前の拡張など）を NumPy 2 環境で読み込んでいる。
- **対処**: 該当パッケージを NumPy 2 対応版に上げる。上げられない場合は `numpy<2` に固定する（`uv add "numpy<2"`）。どちらにするかはプロジェクト全体の依存に影響するので、決める前に相談する。

### 2.9 `np.load` で `Object arrays cannot be loaded when allow_pickle=False`

- **症状**: `ValueError: Object arrays cannot be loaded when allow_pickle=False`。
- **確認**:
```python
d = np.load("x.npz")
print(d.files)
# どのキーが object 配列か（読み込みに失敗するものを探す）
for k in d.files:
    try: print(k, d[k].dtype, d[k].shape)
    except ValueError as e: print(k, "object 配列")
```
- **原因**: 保存時に長さの違うリストや文字列・dict を含む配列が `dtype=object` になり、pickle で書かれた。
- **対処**: 自分で作った信頼できるファイルに限り `np.load(path, allow_pickle=True)`。今後の保存では、可変長データは連結した配列 + オフセット配列に分ける、メタデータは JSON や parquet に分ける。

### 2.10 `setting an array element with a sequence` / `inhomogeneous shape`

- **症状**: `ValueError: setting an array element with a sequence. The requested array has an inhomogeneous shape after 1 dimensions.`
- **確認**:
```python
from collections import Counter
print(Counter(np.shape(x) for x in items))    # 要素ごとの形の分布
```
- **原因**: 長さや形の異なる配列のリストを `np.array` に渡した（スライドごとにパッチ数が違う、など）。NumPy 1.24 以降、暗黙の object 配列化はエラーになった。
- **対処**: パディングして揃える、`np.concatenate` で連結し別途 `lengths` を持つ、本当に object 配列にしたいなら `np.array(items, dtype=object)`。

### 2.11 合計や行列積の結果が実行ごとにわずかに違う

- **症状**: 同じ入力なのに `X @ W` や `sum` の下位桁が実行ごと・マシンごとに違う。`==` での比較テストが落ちる。
- **確認**:
```python
r1, r2 = f(X), f(X)
print(np.abs(r1 - r2).max(), np.allclose(r1, r2, rtol=1e-5, atol=1e-7))
print(X.dtype)
```
- **原因**: 浮動小数点の加算は順序で結果が変わる。BLAS のスレッド数や CPU 命令セット（AVX2 / AVX512）、配列の連続性によって加算順序が変わる。float32 だと差が目立つ。
- **対処**: 比較は `np.allclose` / `np.testing.assert_allclose` で行う。完全一致が必要なら float64 で計算し BLAS を 1 スレッドに固定する。集計値は `sum(dtype=np.float64)` で誤差を減らす。

### 2.12 NaN が混ざったときの誤動作

- **症状**: `argmax` が NaN の位置を返す、`sort` の結果の末尾に NaN が集まる、`x == np.nan` が常に False。
- **確認**:
```python
print(np.isnan(x).sum(), np.isnan(x).any(axis=0).nonzero())
```
- **原因**: NaN は比較演算で常に False になり、`max` / `argmax` は NaN を最大として扱う（伝播する）。
- **対処**: `np.isnan(x)` で判定する。`np.nanargmax`, `np.nanmax`, `np.nanpercentile` を使う。上流で NaN が生まれる箇所（`0/0`、`log(0)`、空配列の `mean`）を `np.errstate(invalid="raise")` で特定する。
