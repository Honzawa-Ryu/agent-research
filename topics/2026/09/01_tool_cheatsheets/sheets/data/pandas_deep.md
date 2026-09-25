# pandas 詳細編（応用・トラブル対応）

> 基本編: [pandas.md](pandas.md)
> 対象バージョン: pandas 2.3.3 / 3.0.3（両方で動作を確認。違いがある箇所は明記）

## Part 1. 応用・上級機能

### 1.1 Copy-on-Write（CoW）の詳細

CoW では「別のオブジェクトから取り出した DataFrame / Series は、常に独立したコピーとして振る舞う」。実際のコピーは書き込むときまで遅延される。

| バージョン | 状態 | 設定 |
|:---|:---|:---|
| 2.3 | 既定で無効 | `pd.options.mode.copy_on_write = True`（有効化）/ `"warn"`（3.0 で挙動が変わる箇所に警告） |
| 3.0 | 常に有効 | オプションは無効化できず、設定すると `Pandas4Warning` |

CoW 下での書き方:

```python
df.loc[df["a"] > 0, "b"] = 1              # 条件付き代入は1回の .loc で
df["a"] = df["a"].fillna(0)               # inplace=True より再代入
df = df.fillna({"a": 0})                  # 複数列ならまとめて
sub = df[df["organ"] == "liver"].copy()   # 3.0 では .copy() は不要だが、2.x と共用するなら付けておく
```

3.0 で挙動が変わるもの:
- `df["a"][0] = 1`、`df[mask]["b"] = 1`、`df["a"].fillna(0, inplace=True)` は元の df を変えない（`ChainedAssignmentError` 警告が出る）
- `Series.to_numpy()` / `.values` が読み取り専用の配列を返すことがある（2.6 参照）
- 2.x で出ていた `SettingWithCopyWarning` は無くなる

### 1.2 文字列の dtype（3.0）

| 指定 | 表示 | 欠損値 | 備考 |
|:---|:---|:---|:---|
| 3.0 の既定（推論） / `"str"` | `str` | `NaN` | `StringDtype(na_value=nan)`。pyarrow があれば pyarrow、なければ Python 実装 |
| `"string"` | `string` | `pd.NA` | 2.x からある nullable 文字列型 |
| `object` | `object` | `None` / `NaN` | 2.x の既定。任意の Python オブジェクトが入る |

```python
s = pd.Series(["a", None])        # 3.0: dtype=str, s[1] は nan / 2.3: dtype=object, s[1] は None
pd.api.types.is_string_dtype(s)   # 2.x・3.0 両方で True（dtype == object の判定は 3.0 で False）
s.str.len()                       # 欠損を含むと float64、含まなければ int64

pd.options.future.infer_string = True   # 2.3 で 3.0 の挙動を先に試す
```

- `str` 型の列に文字列以外（数値など）を代入すると `TypeError`。混在列が必要なら `astype(object)`。
- `df["s"].to_numpy()` は object 配列を返す。

### 1.3 nullable 型と `pd.NA`

整数列に欠損があると、既定では float64 に変わる。nullable 型なら整数のまま保てる。

```python
pd.Series([1, None]).dtype                    # float64
pd.Series([1, None], dtype="Int64")           # Int64（大文字）。欠損は <NA>
pd.Series([True, None], dtype="boolean")
df = df.convert_dtypes()                      # 各列を nullable 型に一括変換
df = df.convert_dtypes(dtype_backend="pyarrow")
```

`pd.NA` は3値論理で伝播する。

| 式 | 結果 |
|:---|:---|
| `pd.NA == 1` | `<NA>` |
| `pd.NA \| True` | `True` |
| `pd.NA & False` | `False` |
| `bool(pd.NA)` | `TypeError`（`if` に使えない） |

`if s[i] == x:` のように要素を `if` に使うコードは、nullable 型で `TypeError` になる。`pd.isna(v)` で先に分岐する。

### 1.4 Arrow バックエンド

```python
df = pd.read_parquet("a.parquet", dtype_backend="pyarrow")   # int64[pyarrow], string[pyarrow] など
df = pd.read_csv("a.csv", engine="pyarrow")                  # マルチスレッドで高速に読む
df = pd.read_csv("a.csv", engine="pyarrow", dtype_backend="pyarrow")
import pyarrow as pa
pd.ArrowDtype(pa.list_(pa.float32()))                        # リスト型の列も持てる
```

- `engine="pyarrow"` は `chunksize`、`skipfooter` など一部の引数に対応しない（エラーになる）。
- Arrow 型の列を NumPy / scikit-learn に渡す前は `df.to_numpy(dtype="float32")` のように型を明示する。

### 1.5 カテゴリ型の詳細

```python
df["dose"] = pd.Categorical(df["dose"], categories=["control", "low", "mid", "high"], ordered=True)
df.sort_values("dose")                         # カテゴリの順に並ぶ
df[df["dose"] >= "mid"]                        # ordered=True なら大小比較できる

s.cat.categories; s.cat.codes                  # カテゴリ一覧と整数コード（欠損は -1）
s.cat.remove_unused_categories()
s.cat.set_categories([...])                    # 無いカテゴリの値は NaN になる
s.cat.rename_categories({"low": "L"})
pd.api.types.union_categoricals([a, b])        # カテゴリの異なる列を連結
```

- `concat` / `merge` でカテゴリが一致しないと object（3.0 では str）に戻る。
- `groupby` の `observed` の既定が 2.x と 3.0 で違う（1.12 参照）。

### 1.6 MultiIndex の操作

```python
g = df.groupby(["study", "organ", "dose"])["score"].mean()   # 3段の MultiIndex

g.xs("liver", level="organ")                   # ある階層の値で切り出す
g.loc[("S01", "liver")]                        # 先頭からのタプル
idx = pd.IndexSlice
g.sort_index().loc[idx[:, "liver", ["low", "high"]]]   # 途中の階層で選ぶ（要ソート）

g.unstack("dose")                              # 階層を列へ
wide.stack()                                   # 列を階層へ（3.0 は新しい実装のみ）
g.swaplevel("organ", "dose").sort_index()
g.reset_index(level="dose")
g.index.get_level_values("organ")
df.columns = ["_".join(map(str, c)) for c in df.columns]   # 2段の列名を平坦化
```

`stack` は 2.1 で新実装（`future_stack=True`）が導入され、3.0 ではそれが唯一の実装になった。新実装では `dropna` / `sort` 引数を指定できない。

### 1.7 groupby の応用

```python
gb = df.groupby(["study", "organ"], as_index=False, observed=True, dropna=False)

gb.agg(n=("slide_id", "size"), auc=("score", lambda s: s.mean()))   # 名前付き集計
df.groupby("organ")["score"].agg(["mean", "std", "count"])

df["rank_in_group"] = df.groupby("organ")["score"].rank(ascending=False)
df["i_in_group"] = df.groupby("animal").cumcount()       # グループ内の通し番号
df["group_id"] = df.groupby(["study", "animal"]).ngroup()  # グループごとの整数 ID
df["prev"] = df.groupby("animal")["score"].shift(1)
df["z"] = (df["score"] - df.groupby("organ")["score"].transform("mean")) / \
          df.groupby("organ")["score"].transform("std")   # 文字列指定の transform は高速

df.groupby("organ").pipe(lambda g: g["a"].sum() / g["b"].sum())
df.groupby("organ").head(3)                               # グループごとに先頭 3 行
df.groupby("organ").sample(n=10, random_state=0)          # グループごとにサンプリング
```

`apply` の変化: 2.x ではグループ化に使った列も関数に渡っていた（`FutureWarning`）。3.0 では渡されない（`include_groups=False` が既定）。関数内でキー列が必要なら `g.name` を使う。

### 1.8 ウィンドウ関数

```python
s.rolling(7, min_periods=1, center=True).mean()
s.expanding().max()                               # 先頭からの累積
s.ewm(span=10, adjust=False).mean()               # 指数移動平均

ts = df.set_index("date").sort_index()
ts["value"].rolling("7D").mean()                  # 時間幅の窓（DatetimeIndex が必要、単調増加であること）

df.groupby("animal")["weight"].rolling(3).mean()  # グループごと。結果は (animal, 元の index) の MultiIndex
df["w_roll"] = (df.groupby("animal")["weight"]
                  .rolling(3).mean().reset_index(level=0, drop=True))   # 元の行に戻す
df["w_roll"] = df.groupby("animal")["weight"].transform(lambda s: s.rolling(3).mean())   # 同じ結果
```

### 1.9 結合の応用

```python
m = left.merge(right, on="slide_id", how="outer", indicator=True, suffixes=("", "_r"))
m["_merge"].value_counts()                  # left_only / right_only / both
m.query("_merge != 'both'")                 # 対応が取れなかった行

# 近い時刻どうしで結合（両方ともキーでソート済みであること）
pd.merge_asof(
    obs.sort_values("time"), dose.sort_values("time"),
    on="time", by="animal", direction="backward", tolerance=pd.Timedelta("1D"),
)

left.join(right, how="left")                # index どうしの結合
```

### 1.10 メソッドチェーン

```python
out = (
    pd.read_parquet("scores.parquet")
      .query("organ in @organs and score.notna()", engine="python")   # @ で Python 変数を参照
      .assign(log_s=lambda d: np.log1p(d["score"]),                   # lambda で途中の df を参照
              pos=lambda d: d["log_s"] > thr)
      .pipe(add_meta, meta=meta_df)                                   # 自作関数を挟む
      .groupby("organ", observed=True).agg(rate=("pos", "mean"))
      .sort_values("rate", ascending=False)
)
df.eval("ratio = a / b")                    # 列式の評価（新しい df を返す）
```

`assign` の中で同じ呼び出し内の前の列を参照するには lambda を使う（上の `pos`）。

### 1.11 IO の高速化

```python
df.to_parquet("out/", partition_cols=["study"])                       # study=S01/ ... のディレクトリに分割
pd.read_parquet("out/", filters=[("study", "in", ["S01", "S02"])])    # 必要なパーティションだけ読む
pd.read_parquet("a.parquet", columns=["slide_id", "score"])           # 列指定で読む量を減らす

import pyarrow.parquet as pq
pq.read_metadata("a.parquet")                # 行数・行グループ数・スキーマを中身を読まずに確認
pq.read_schema("a.parquet")
```

| 形式 | 速さ | 型の保存 | 備考 |
|:---|:---|:---|:---|
| CSV | 遅い | されない | 人が見る・他ツールに渡す用 |
| Parquet | 速い | される | 基本はこれ。列単位で読める |
| Feather | 最速 | される | 一時ファイル向け |
| pickle | 速い | される | pandas のバージョンが違うと読めないことがある。長期保存には使わない |

### 1.12 メモリの削減

```python
df.memory_usage(deep=True).sort_values(ascending=False).head(10)   # 重い列
df["n"] = pd.to_numeric(df["n"], downcast="integer")               # 入る最小の整数型へ
df["x"] = pd.to_numeric(df["x"], downcast="float")                 # float32 へ
low_card = [c for c in df.select_dtypes(["object", "string"]).columns
            if df[c].nunique() < 0.05 * len(df)]
df[low_card] = df[low_card].astype("category")
```

### 1.13 テスト用の比較

```python
pd.testing.assert_frame_equal(a, b, check_dtype=False, check_like=True, rtol=1e-5)   # check_like: 列・行の順を無視
pd.testing.assert_series_equal(s1, s2, check_names=False)
pd.testing.assert_index_equal(a.index, b.index)
```

`a.equals(b)` は dtype も含めて完全一致かどうかだけを返す（差分の場所は教えてくれない）。

### 1.14 2.3 と 3.0 の違い（確認済みの挙動）

| 項目 | 2.3.3 | 3.0.3 |
|:---|:---|:---|
| Copy-on-Write | 無効（opt-in） | 常に有効 |
| 連鎖代入 `df["a"][0] = 1` | 効く（`FutureWarning`） | 効かない（`ChainedAssignmentError` 警告） |
| 文字列列の dtype | `object` | `str` |
| 文字列からの `to_datetime` | `datetime64[ns]` | 入力の精度から推論（`"2024-01-01"` は `datetime64[us]`） |
| `groupby` の `observed` 既定 | `False`（`FutureWarning`） | `True` |
| `groupby.apply` にキー列を渡す | 渡す（`FutureWarning`） | 渡さない |
| 型の合わない値の代入 `s.iloc[0] = "a"`（int 列） | object に変わる（`FutureWarning`） | `TypeError` |
| 頻度の別名 `"H"`, `"M"` | 使える（`FutureWarning`） | `ValueError`。`"h"`, `"ME"` を使う |
| `DataFrame.applymap` | 非推奨 | 削除。`DataFrame.map` |
| `stack` | 旧実装が既定 | 新実装のみ |

## Part 2. トラブル対応

### 2.1 代入したのに値が変わらない（3.0 の連鎖代入）

- **症状**: 2.x で動いていたコードが 3.0 では元の df を更新しない。`ChainedAssignmentError: A value is being set on a copy of a DataFrame or Series through chained assignment.` という警告が出る。
- **確認**:
```python
import warnings
warnings.simplefilter("error", pd.errors.ChainedAssignmentError)   # 該当行で止める
print(pd.__version__)
```
```bash
grep -nE '\]\[[^]]+\] *=|\]\.(fillna|replace|clip|where|mask)\(.*inplace=True' src/*.py   # 典型パターンの洗い出し
```
- **原因**: CoW では `df["a"]` や `df[mask]` は独立したオブジェクトなので、それへの代入・`inplace` 操作は元に伝わらない。
- **対処**: `df.loc[mask, "b"] = v`、`df.loc[0, "a"] = v`、`df["a"] = df["a"].fillna(0)` に書き換える。2.3 で `pd.options.mode.copy_on_write = "warn"` にすると移行前に該当箇所を洗い出せる。

### 2.2 3.0 で文字列列の判定・欠損の扱いが変わった

- **症状**: `dtype == object` の分岐に入らない、`x is None` の判定が効かない、`select_dtypes("object")` で `Pandas4Warning`（3.0.3 では互換のため `str` 列も含まれるが、将来含まれなくなる）。
- **確認**:
```python
print(df.dtypes.value_counts())
print(repr(df["s"].dtype), repr(df["s"].iloc[df["s"].isna().argmax()]))   # 欠損の中身（nan か None か NA か）
```
- **原因**: 3.0 では文字列列が `str` 型（欠損は `NaN`）で読み込まれる。
- **対処**:
  - 判定は `pd.api.types.is_string_dtype(s)`、列選択は `df.select_dtypes(include=["object", "string"])`（2.3・3.0 の両方で文字列列を選べ、警告も出ない）
  - 欠損判定は `pd.isna(v)` / `s.isna()` に統一する
  - どうしても旧挙動が必要な箇所は `astype(object)` で明示する

### 2.3 merge の結果が空・キー型の不一致

- **症状**: `ValueError: You are trying to merge on int64 and str columns for key 'k'`。もしくはエラーなしで一致 0 件になる。
- **確認**:
```python
print(left["k"].dtype, right["k"].dtype)
print(left["k"].head(3).map(repr).tolist(), right["k"].head(3).map(repr).tolist())   # 空白・先頭 0・".0" の有無
print(len(set(left["k"]) & set(right["k"])), left["k"].nunique(), right["k"].nunique())
```
- **原因**: 片方が数値、片方が文字列。文字列同士でも前後の空白、大文字小文字、先頭の 0 の欠落（`"007"` と `"7"`）、欠損を含む整数列が float になった結果の `"7.0"` など。
- **対処**: 読み込み時に `dtype={"k": str}` で揃え、`str.strip()` / `str.zfill(n)` で正規化してから結合。`indicator=True` で `left_only` の行を確認する。

### 2.4 merge 後に行数が爆発・メモリ不足

- **症状**: merge 後の行数が元より桁違いに多い、または merge 中にプロセスが落ちる。
- **確認**:
```python
print(left["k"].duplicated().sum(), right["k"].duplicated().sum())
est = (left["k"].value_counts() * right["k"].value_counts()).sum()   # 結果の行数の見積もり
print(f"推定 {est:,.0f} 行")
```
- **原因**: 両側でキーが重複していて多対多の結合になった（重複数の積だけ行が増える）。キーの欠損同士も一致として扱われる。
- **対処**: `validate="one_to_one"` / `"many_to_one"` を付けて事前に検出する。右側を `drop_duplicates("k")` するか、集計してから結合する。欠損キーは結合前に `dropna(subset=["k"])`。

### 2.5 index の重複でずれる・NaN が入る

- **症状**: 列の代入や `concat(axis=1)` で値が NaN になる、行数が増える、`cannot reindex on an axis with duplicate labels`。
- **確認**:
```python
print(df.index.is_unique, df.index.duplicated().sum())
print(s.index.equals(df.index))
```
- **原因**: pandas は演算・代入を index のラベルで揃える。filter や concat の後で index が重複したり、別の df と並びが違ったりすると、位置ではなくラベルで対応付けられる。
- **対処**: `reset_index(drop=True)` してから結合する。位置で代入したいなら `df["x"] = s.to_numpy()`（長さが同じことを確認してから）。`concat` では `ignore_index=True`。

### 2.6 `assignment destination is read-only`（3.0）

- **症状**: `arr = df["x"].to_numpy(); arr[0] = 0` で `ValueError: assignment destination is read-only`。
- **確認**:
```python
arr = df["x"].to_numpy()
print(arr.flags["WRITEABLE"], np.shares_memory(arr, df["x"].to_numpy()))
```
- **原因**: 3.0 の CoW では、Series が持つデータのビューを返すとき、書き換えで df が壊れないよう読み取り専用にする。
- **対処**: `arr = df["x"].to_numpy(copy=True)`。df 自体を更新したいなら pandas 側で代入する。

### 2.7 groupby で行・グループが消える／増える

- **症状**: 集計結果のグループ数が想定と違う。キーが欠損の行が消える、カテゴリ型で観測されていない組み合わせが大量に出る（2.x）、または出ない（3.0）。
- **確認**:
```python
print(df[keys].isna().sum())
print({k: df[k].dtype for k in keys})
print(df.groupby(keys, dropna=False, observed=True).ngroups)
```
- **原因**: `dropna=True`（既定）で欠損キーの行は除外される。カテゴリ型のキーは `observed=False`（2.x の既定）だと全カテゴリの直積が出る。3.0 は既定が `observed=True` に変わった。
- **対処**: `dropna` と `observed` を常に明示する。全組み合わせが必要な場合（件数 0 も表に出したい）は `observed=False` を指定するか、`reindex` で補う。

### 2.8 タイムゾーン付き日時の比較・結合エラー

- **症状**: `TypeError: Invalid comparison between dtype=datetime64[us, UTC] and Timestamp`、`Cannot compare tz-naive and tz-aware`、merge で一致しない。
- **確認**:
```python
print(df["t"].dtype, getattr(df["t"].dt, "tz", None))
print(other["t"].dtype)
```
- **原因**: タイムゾーン付き（aware）と無し（naive）が混在している。ISO 形式の `+09:00` などを含む文字列を読むと aware になる。
- **対処**: どちらかに揃える。
```python
df["t"] = pd.to_datetime(df["t"], utc=True)                       # 全部 UTC で aware に
df["t"] = df["t"].dt.tz_convert("Asia/Tokyo").dt.tz_localize(None)  # 日本時間の naive に
ts = pd.Timestamp("2024-01-02", tz="UTC")                           # 比較相手も aware に
```

### 2.9 3.0 で日時の整数表現・差分の単位が変わった

- **症状**: `df["t"].astype("int64")` の値が 1000 分の 1 になった、エポック秒への変換結果がずれる、2.x と 3.0 で保存したファイルの値が一致しない。
- **確認**:
```python
print(df["t"].dtype)            # datetime64[us] か [ns] か
```
- **原因**: 3.0 では文字列から `to_datetime` した結果の精度が入力から推論され、秒までの文字列だと `datetime64[us]`（2.x は常に ns）。整数化すると単位がマイクロ秒になる。
- **対処**: 整数表現に依存しない書き方にする: `(df["t"] - pd.Timestamp("1970-01-01")) // pd.Timedelta("1s")`。ns に揃えたいなら `df["t"].astype("datetime64[ns]")`。

### 2.10 parquet が読めない

- **症状**: `ImportError: Unable to find a usable engine; tried using: 'pyarrow', 'fastparquet'`、`ArrowInvalid`、`ArrowTypeError`、別プロジェクトで書いた parquet が読めない。
- **確認**:
```python
import pyarrow as pa, pyarrow.parquet as pq
print(pd.__version__, pa.__version__)
print(pq.read_schema("a.parquet"))                  # 列の型
print(pq.read_metadata("a.parquet").created_by)     # 書いたライブラリとバージョン
```
- **原因**: pyarrow が入っていない／古い（pandas 3.0 の要件は pyarrow 13 以上）。object 列に型の違う値（数値と文字列）が混ざっていて書き込めない。書いた側が新しい型（拡張型、ネスト）を使っている。
- **対処**: `uv add pyarrow` で入れる・上げる。書き込み前に混在列を `astype(str)` などで揃える。プロジェクト間で共有するファイルは基本的な型（数値・文字列・日時）に限定する。

### 2.11 2.x の FutureWarning が 3.0 でエラーになる

- **症状**: 3.0 に上げたら `TypeError`、`ValueError: Invalid frequency`、`AttributeError: 'DataFrame' object has no attribute 'applymap'` などで止まる。
- **確認**: 2.3 の環境で警告をエラーにして実行し、該当箇所を洗い出す。
```python
import warnings
warnings.filterwarnings("error", category=FutureWarning, module="pandas")
warnings.filterwarnings("error", category=DeprecationWarning, module="pandas")
pd.options.mode.copy_on_write = "warn"
pd.options.future.infer_string = True
```
- **原因**: 3.0 で非推奨機能が削除され、既定値が変わった（1.14 の表）。
- **対処**: 2.3 で警告が出なくなるまで直してから 3.0 に上げる。依存ライブラリ側から出る警告は `module=` で区別する。

### 2.12 apply / iterrows が遅い

- **症状**: 数十万行で `df.apply(f, axis=1)` や `for _, r in df.iterrows()` が数分以上かかる。
- **確認**:
```python
%timeit df.head(10_000).apply(f, axis=1)        # 小さく測って全体の時間を見積もる
```
- **原因**: 行ごとに Python 関数を呼び、各行で Series を作っている。`iterrows` は行ごとに型を変換するので dtype も崩れる。
- **対処**（上から順に検討）:
  - 列演算・`np.where` / `np.select`・`Series.case_when`・`.str` / `.dt` アクセサで書き直す
  - グループ単位の処理は `groupby(...).transform("mean")` のような組み込み集計にする
  - どうしてもループするなら `itertuples(index=False)` か、`zip(df["a"].to_numpy(), df["b"].to_numpy())`
  - 重い処理が行ごとに独立しているなら、配列にしてから NumPy でまとめて処理する
