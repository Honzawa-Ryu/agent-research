# pandas チートシート

> 対象: pandas 2.x / 3.x（このマシンの `.venv` はプロジェクトにより 2.3.3 と 3.0.3 が混在）
> 公式: https://pandas.pydata.org/docs/
> 詳細編（応用・トラブル対応）: [pandas_deep.md](pandas_deep.md)

## 1. 読む・書く

```python
import pandas as pd

df = pd.read_csv("a.csv")
df = pd.read_csv("a.tsv", sep="\t", usecols=["id", "label"], dtype={"id": str})
df = pd.read_parquet("a.parquet")                  # 速くて型も保存される。大きいデータにおすすめ
df = pd.read_excel("a.xlsx", sheet_name="Sheet1")  # openpyxl が必要
df = pd.read_json("a.jsonl", lines=True)

df.to_csv("out.csv", index=False)                  # index=False を忘れると余計な列ができる
df.to_parquet("out.parquet", index=False)
```

## 2. 中身を見る

```python
df.head(); df.tail(3); df.sample(5)
df.shape; df.columns; df.dtypes
df.info()                      # 型と欠損の数
df.describe()                  # 数値列の要約統計
df["label"].value_counts(dropna=False)
df["label"].nunique(); df["label"].unique()
df.isna().sum()                # 列ごとの欠損数
```

## 3. 列と行を選ぶ

```python
df["a"]                        # Series
df[["a", "b"]]                 # DataFrame
df.loc[行ラベル, 列ラベル]       # ラベルで選ぶ
df.iloc[0:5, 0:2]              # 位置で選ぶ

df[df["score"] > 0.5]
df[(df["dose"] == "high") & (df["organ"] == "liver")]      # 複数条件は & | と括弧
df[df["organ"].isin(["liver", "kidney"])]
df[df["slide_id"].str.startswith("TG")]
df.query("score > 0.5 and organ == 'liver'")
```

## 4. 列を作る・変える

```python
df["ratio"] = df["a"] / df["b"]
df = df.assign(log_a=np.log1p(df["a"]))
df["group"] = np.where(df["dose"] > 0, "treated", "control")
df["label_id"] = df["label"].map({"normal": 0, "lesion": 1})
df["bin"] = pd.cut(df["age"], bins=[0, 10, 20, 100])
df = df.rename(columns={"old": "new"})
df = df.drop(columns=["tmp"])
df["a"] = df["a"].astype("float32")
df.loc[df["score"] < 0, "score"] = 0          # 条件に合う行の値を書き換える
```

## 5. 欠損・重複・並べ替え

```python
df.dropna(subset=["label"]); df.fillna({"score": 0})
df.drop_duplicates(subset=["slide_id"], keep="first")
df.sort_values(["organ", "score"], ascending=[True, False])
df.reset_index(drop=True)                     # 行番号を振り直す
df.set_index("slide_id")
```

## 6. グループ集計

```python
df.groupby("organ")["score"].mean()
df.groupby(["dose", "organ"]).agg(
    n=("slide_id", "count"),
    mean_score=("score", "mean"),
    max_score=("score", "max"),
).reset_index()

df["score_z"] = df.groupby("organ")["score"].transform(lambda s: (s - s.mean()) / s.std())   # 元の行数のまま
df.groupby("organ").filter(lambda g: len(g) >= 10)                                            # 条件に合うグループだけ残す
```

## 7. 結合

```python
pd.merge(left, right, on="slide_id", how="left")              # how: inner / left / right / outer
pd.merge(left, right, left_on="id", right_on="slide_id", how="inner", validate="one_to_one")
pd.concat([df1, df2], ignore_index=True)                      # 縦に連結
pd.concat([df1, df2], axis=1)                                 # 横に連結（index で揃う）
```

`validate="one_to_one"` / `"many_to_one"` を付けると、キーが重複していたときにエラーにしてくれる（行が勝手に増えるのを防げる）。

## 8. 形を変える

```python
wide = df.pivot_table(index="animal", columns="organ", values="score", aggfunc="mean")
long = wide.reset_index().melt(id_vars="animal", var_name="organ", value_name="score")
pd.crosstab(df["dose"], df["finding"], margins=True)
df.explode("tags")                           # リストの列を行に展開する
```

## 9. 文字列・日付

```python
df["name"].str.lower(); df["name"].str.strip(); df["name"].str.replace("-", "_")
df["name"].str.contains("liver", case=False, na=False)
df[["study", "animal"]] = df["slide_id"].str.split("_", n=1, expand=True)
df["name"].str.extract(r"(\d+)")

df["date"] = pd.to_datetime(df["date"])
df["date"].dt.year; df["date"].dt.strftime("%Y-%m-%d")
```

## 10. pandas 3.0 での主な変更

このマシンには 2.x と 3.x のプロジェクトが混在しているので、コードを移すときに注意する。

| 変更 | 影響と対処 |
|:---|:---|
| Copy-on-Write が常に有効 | `df[df.a > 0]["b"] = 1` のような連鎖代入は**効かなくなる**（元の df は変わらない）。`df.loc[df.a > 0, "b"] = 1` と書く |
| 文字列の既定の型が `str`（PyArrow ベース）に | `dtype == object` で文字列を判定しているコードが動かなくなる。`pd.api.types.is_string_dtype` を使う |
| 非推奨だった機能の削除 | 2.x で `FutureWarning` が出ていた書き方はエラーになる。2.x で警告を消してから移行する |

## 11. 大きいデータ

```python
df = pd.read_csv("big.csv", usecols=["a", "b"], dtype={"a": "float32"})    # 必要な列だけ、小さい型で
for chunk in pd.read_csv("big.csv", chunksize=100_000):                  # 少しずつ読む
    process(chunk)
df.memory_usage(deep=True).sum() / 1e9                                    # メモリ使用量 (GB)
df["organ"] = df["organ"].astype("category")                              # 種類の少ない文字列はカテゴリ型に
```

## 12. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| `SettingWithCopyWarning`（2.x） | フィルタした結果に代入している。`.loc` を使うか、`.copy()` してから代入する |
| merge したら行が増えた | キーが重複している。`validate=` を付ける、先に `drop_duplicates` |
| ID の先頭の 0 が消えた | 数値として読まれた。`dtype={"id": str}` |
| `and` / `or` でエラー | Series の条件は `&` / `|` と括弧を使う |
| CSV に余計な `Unnamed: 0` 列 | `to_csv` で `index=False` を忘れた |
| `groupby` で NaN のグループが消える | 既定で除外される。`dropna=False` |
| `apply` が遅い | できるだけベクトル演算（列どうしの演算、`np.where`、`.str`、`.map`）で書く |
