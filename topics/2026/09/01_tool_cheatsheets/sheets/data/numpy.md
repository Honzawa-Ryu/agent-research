# NumPy チートシート

> 対象: NumPy 2.x（このマシンの `.venv` は 2.3〜2.4）
> 公式: https://numpy.org/doc/stable/

## 1. 配列を作る

```python
import numpy as np

np.array([1, 2, 3]); np.array([[1, 2], [3, 4]], dtype=np.float32)
np.zeros((2, 3)); np.ones((2, 3)); np.full((2, 3), 7); np.empty((2, 3))
np.zeros_like(a); np.ones_like(a)
np.arange(0, 10, 2)            # [0 2 4 6 8]
np.linspace(0, 1, 5)           # [0. 0.25 0.5 0.75 1.]
np.eye(3)
```

### 乱数（新しい書き方）

```python
rng = np.random.default_rng(42)          # シードを固定した生成器
rng.random((2, 3)); rng.normal(0, 1, 100); rng.integers(0, 10, size=5)
rng.choice(n, size=100, replace=False)   # 重複なしで100個選ぶ
rng.permutation(n); rng.shuffle(a)
```

`np.random.seed()` + `np.random.rand()` の古い書き方より、`default_rng` のほうが扱いやすい。

## 2. 形状と型

```python
a.shape, a.ndim, a.size, a.dtype
a.reshape(2, -1)               # -1 は自動で計算される
a.ravel(); a.flatten()         # 1次元にする（flatten は必ずコピー）
a.T; a.transpose(2, 0, 1)      # 軸の入れ替え（HWC → CHW など）
a[None]; a[:, None]; np.expand_dims(a, 0)    # 次元を追加
a.squeeze()                    # 長さ1の次元を削除
a.astype(np.float32)           # 型の変換（コピーされる）
```

## 3. インデックス

```python
a[0]; a[-1]; a[1:4]; a[::2]; a[::-1]
m[0, :]; m[:, 1]; m[1:3, 2:4]
m[..., 0]                      # 最後の軸の0番目（画像の R チャンネルなど）

a[a > 0]                       # 条件で抽出（ブールインデックス）
a[[0, 2, 5]]                   # 指定した位置を抽出（ファンシーインデックス）
np.where(a > 0, a, 0)          # 条件で値を選ぶ
np.where(a > 0)                # 条件を満たす位置
np.argwhere(mask)              # 条件を満たす座標 (N, ndim)
```

**スライスはビュー（元の配列と同じメモリ）、ブール・ファンシーインデックスはコピー。** スライスを書き換えると元の配列も変わる。必要なら `.copy()`。

## 4. 演算と集約

```python
a + b; a * b; a @ b            # 要素ごとの和・積、行列積
np.sqrt(a); np.exp(a); np.log(a); np.abs(a); np.clip(a, 0, 1)

a.sum(); a.mean(axis=0); a.std(axis=1, keepdims=True)
a.min(); a.max(); a.argmin(); a.argmax(axis=1)
np.median(a); np.percentile(a, [5, 95]); np.quantile(a, 0.5)
np.nanmean(a)                  # NaN を無視
np.cumsum(a); np.diff(a)
np.unique(a, return_counts=True)
np.sort(a); np.argsort(a)[::-1]   # 降順の並び順
np.argpartition(a, -k)[-k:]       # 上位 k 個（順不同、sort より速い）
```

`axis=0` は「行方向につぶす」（列ごとの集計）、`axis=1` は「列方向につぶす」（行ごとの集計）。

## 5. ブロードキャスト

形状が違う配列どうしでも、後ろの軸から見て「同じ長さ」か「1」なら演算できる。

```python
X = rng.random((100, 3))
X_norm = (X - X.mean(axis=0)) / X.std(axis=0)        # (100,3) - (3,) → OK

# 全ペアの距離行列
D = np.linalg.norm(A[:, None, :] - B[None, :, :], axis=-1)   # (N,1,D) - (1,M,D) → (N,M)
```

## 6. 結合と分割

```python
np.concatenate([a, b], axis=0)    # 既存の軸でつなぐ
np.stack([a, b], axis=0)          # 新しい軸を作ってつなぐ
np.vstack([a, b]); np.hstack([a, b])
np.split(a, 3); np.array_split(a, 3)   # array_split は割り切れなくてもよい
```

## 7. 線形代数

```python
np.linalg.norm(x, axis=1)          # L2 ノルム
x / np.linalg.norm(x, axis=1, keepdims=True)    # 行ごとに正規化
A @ B; np.dot(a, b); np.einsum("nd,md->nm", A, B)
np.linalg.inv(A); np.linalg.solve(A, b)
U, S, Vt = np.linalg.svd(X, full_matrices=False)
w, V = np.linalg.eigh(C)           # 対称行列の固有値分解
```

### コサイン類似度

```python
An = A / np.linalg.norm(A, axis=1, keepdims=True)
Bn = B / np.linalg.norm(B, axis=1, keepdims=True)
sim = An @ Bn.T
```

## 8. 保存と読み込み

```python
np.save("a.npy", a); a = np.load("a.npy")
np.savez_compressed("data.npz", x=X, y=y); d = np.load("data.npz"); d["x"]
a = np.load("big.npy", mmap_mode="r")       # メモリに全部載せず、必要な部分だけ読む
np.savetxt("a.csv", a, delimiter=","); np.loadtxt("a.csv", delimiter=",")
```

## 9. dtype とメモリ

| dtype | バイト数 | 用途 |
|:---|---:|:---|
| `uint8` | 1 | 画像（0〜255） |
| `float16` | 2 | 特徴量の保存（容量を半分にする） |
| `float32` | 4 | 深層学習の標準 |
| `float64` | 8 | NumPy の既定。統計計算 |
| `int64` | 8 | インデックス、ラベル |

`a.nbytes / 1e9` でメモリ使用量 (GB) がわかる。

## 10. NumPy 2 での主な変更

| 変更 | 対処 |
|:---|:---|
| `np.float_`, `np.NaN`, `np.Inf` などの別名が削除 | `np.float64`, `np.nan`, `np.inf` を使う |
| スカラーの表示が `np.float64(1.0)` のようになる | 表示だけの変化。`float(x)` で普通の数値にできる |
| 型の昇格ルールの変更（NEP 50） | `np.float32(1) + 1.0` が float32 のままになるなど。精度が気になる箇所は明示的に `astype` |
| NumPy 1.x 向けにビルドされた拡張モジュールが動かない | 依存パッケージを NumPy 2 対応版に更新する |

## 11. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| 元の配列が勝手に変わった | スライス（ビュー）を書き換えた。`.copy()` する |
| `operands could not be broadcast together` | 形状が合わない。`shape` を確認し、`[:, None]` などで軸を足す |
| 画像の値がおかしい（オーバーフロー） | `uint8` どうしの演算で 255 を超えた。先に `astype(np.float32)` |
| 整数の割り算で結果が変わる | `//` は切り捨て、`/` は float になる |
| `==` での比較が合わない（float） | 誤差がある。`np.isclose` / `np.allclose` を使う |
| 平均が NaN になる | NaN が含まれている。`np.nanmean` を使うか、`np.isnan` で除く |
