# Matplotlib / seaborn チートシート

> 対象: Matplotlib 3.10、seaborn 0.13（このマシンの `.venv` で確認）
> 公式: https://matplotlib.org/stable/ ・ https://seaborn.pydata.org/

## 1. 基本（オブジェクト指向の書き方）

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(6, 4))
ax.plot(x, y, label="train", color="C0", lw=2)
ax.plot(x, y2, label="val", color="C1", ls="--")
ax.set(xlabel="epoch", ylabel="loss", title="Loss", ylim=(0, 1))
ax.legend()
ax.grid(alpha=0.3)
fig.tight_layout()
fig.savefig("loss.png", dpi=200, bbox_inches="tight")
plt.close(fig)                           # ループで大量に作るときは必ず閉じる
```

`plt.plot(...)` のような状態を持つ書き方より、`fig, ax` を使うほうが複数の図を扱いやすい。

### サーバ（ディスプレイなし）で使う

```python
import matplotlib
matplotlib.use("Agg")                    # pyplot を import する前に
```

## 2. 複数のパネル

```python
fig, axes = plt.subplots(2, 3, figsize=(12, 7), sharex=True, sharey=True)
for ax, img, title in zip(axes.flat, imgs, titles):
    ax.imshow(img); ax.set_title(title); ax.axis("off")
fig.suptitle("Samples")

fig, axd = plt.subplot_mosaic([["a", "a", "b"], ["c", "d", "b"]], figsize=(10, 6))   # 不規則な配置
axd["a"].plot(x, y)
```

## 3. よく使うプロット

```python
ax.scatter(x, y, c=labels, s=5, alpha=0.5, cmap="tab10")
ax.hist(v, bins=50, alpha=0.6, density=True)
ax.bar(names, values, yerr=err, capsize=3)
ax.boxplot([a, b], tick_labels=["A", "B"])
ax.errorbar(x, m, yerr=s, fmt="o-")
ax.fill_between(x, m - s, m + s, alpha=0.2)       # 平均 ± 標準偏差の帯
ax.axhline(0.5, color="gray", ls=":"); ax.axvline(10)
ax.text(x, y, "note", fontsize=9)
ax.annotate("peak", xy=(x0, y0), xytext=(x0 + 1, y0 + 0.1), arrowprops=dict(arrowstyle="->"))
```

## 4. 画像とヒートマップ

```python
ax.imshow(rgb)                                          # (H, W, 3) の uint8、または 0〜1 の float
im = ax.imshow(score_map, cmap="magma", vmin=0, vmax=1)
fig.colorbar(im, ax=ax, fraction=0.046, pad=0.04)

ax.imshow(thumb)
ax.imshow(heatmap, cmap="jet", alpha=0.4, extent=(0, thumb.shape[1], thumb.shape[0], 0))   # サムネイルに重ねる
```

| カラーマップ | 用途 |
|:---|:---|
| `viridis`, `magma`, `cividis` | 連続値（知覚的に均一、色覚多様性に配慮） |
| `RdBu_r`, `coolwarm` | 0 を中心にした正負の値（`vmin=-v, vmax=v` で対称に） |
| `tab10`, `tab20` | カテゴリ |
| `jet` | 見栄えはよいが明るさが均一でない。定量的な比較には向かない |

## 5. seaborn（DataFrame から描く）

```python
import seaborn as sns
sns.set_theme(style="whitegrid", context="paper")

sns.boxplot(data=df, x="dose", y="score", hue="organ", ax=ax)
sns.violinplot(data=df, x="dose", y="score", inner="quart")
sns.stripplot(data=df, x="dose", y="score", color="k", size=2, alpha=0.5)   # 箱ひげの上に点を重ねる
sns.histplot(data=df, x="score", hue="label", bins=50, stat="density", common_norm=False)
sns.kdeplot(data=df, x="score", hue="label", fill=True)
sns.scatterplot(data=emb_df, x="umap1", y="umap2", hue="cluster", s=5, linewidth=0)
sns.lineplot(data=log_df, x="epoch", y="auc", hue="model", errorbar="sd")   # seed 間の平均 ± SD
sns.heatmap(cm, annot=True, fmt=".2f", cmap="Blues", square=True)
sns.clustermap(mat, cmap="vlag", z_score=0)
```

### Figure 単位の関数（ファセット）

```python
g = sns.relplot(data=df, x="epoch", y="auc", hue="model", col="organ", kind="line", col_wrap=3)
g = sns.catplot(data=df, x="dose", y="score", col="organ", kind="box")
g = sns.displot(data=df, x="score", hue="label", col="organ", kind="kde")
g.set_titles("{col_name}"); g.savefig("facet.png", dpi=200)
```

`relplot` / `catplot` / `displot` は Figure ごと作るので `ax=` は渡せない。

## 6. 論文・スライド用の設定

```python
plt.rcParams.update({
    "font.size": 10,
    "axes.spines.top": False, "axes.spines.right": False,
    "savefig.dpi": 300,
    "pdf.fonttype": 42,            # PDF のフォントを埋め込む（Illustrator で編集できる）
})
fig.savefig("fig.pdf", bbox_inches="tight")     # ベクター形式（拡大しても劣化しない）
fig.savefig("fig.svg")
```

### 日本語を表示する

```python
# japanize-matplotlib を入れる方法（簡単）
import japanize_matplotlib

# フォントを直接指定する方法
plt.rcParams["font.family"] = "Noto Sans CJK JP"   # fc-list :lang=ja でインストール済みのフォントを確認
```

## 7. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| 日本語が □ になる | 日本語フォントがない。第6節を参照 |
| ラベルが見切れる | `fig.tight_layout()` か `savefig(..., bbox_inches="tight")`、または `plt.subplots(layout="constrained")` |
| メモリが増え続ける / 警告が出る | 図を閉じていない。`plt.close(fig)` |
| `cannot connect to X server` | ディスプレイがない。`matplotlib.use("Agg")` |
| 画像の色がおかしい | BGR のまま表示している（OpenCV）。RGB に変換する |
| 画像が真っ白 / 真っ黒 | float の画像が 0〜1 の範囲にない。`uint8` にするか正規化する |
| seaborn の凡例が邪魔 | `sns.move_legend(ax, "upper left", bbox_to_anchor=(1, 1))` |
| 点が多すぎて描画が遅い / 潰れる | `s` と `alpha` を小さくする、`rasterized=True`、ランダムに間引く |
