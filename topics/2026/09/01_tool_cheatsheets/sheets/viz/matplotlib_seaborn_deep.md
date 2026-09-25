# Matplotlib / seaborn 詳細編（応用・トラブル対応）

> 基本編: [matplotlib_seaborn.md](matplotlib_seaborn.md)
> 対象バージョン: Matplotlib 3.10.9、seaborn 0.13.2（テンプレートの `.venv` のソースで確認）
> 公式: https://matplotlib.org/stable/ ・ https://seaborn.pydata.org/

## Part 1. 応用・上級機能

### 1.1 Figure / Axes / Artist の関係

| オブジェクト | 役割 | 主な取り出し方 |
|:---|:---|:---|
| `Figure` | 図全体。保存の単位 | `fig = plt.figure()`、`ax.figure` |
| `SubFigure` | Figure の中の小さな Figure。suptitle や凡例を独立に持てる | `fig.subfigures(1, 2)` |
| `Axes` | 1 枚のグラフ（座標系） | `fig.add_subplot()`、`fig.axes` |
| `Artist` | 線・点・文字・画像など、描かれるものすべて | `ax.lines`、`ax.collections`、`ax.images`、`ax.texts`、`ax.patches` |

```python
line, = ax.plot(x, y)             # plot は Line2D のリストを返す
sc = ax.scatter(x, y)             # PathCollection
im = ax.imshow(img)               # AxesImage
line.set(color="C3", lw=1)        # 描いたあとに見た目を変える
ax.get_legend_handles_labels()    # 凡例に載る Artist とラベル
```

seaborn の Axes 単位の関数（`sns.boxplot(..., ax=ax)` など）は Axes を返すので、そのあとは Matplotlib の API で調整できる。

### 1.2 レイアウトエンジン

| 指定 | 動作 |
|:---|:---|
| `layout="constrained"` | 軸ラベル・カラーバー・凡例のぶんの余白を自動で確保する。まずはこれ |
| `layout="compressed"` | constrained と同じ仕組み。`imshow` など縦横比が固定の Axes を並べたときに隙間を詰める |
| `layout="tight"` | `fig.tight_layout()` と同じ。カラーバーや `fig.legend` の扱いが constrained より弱い |
| `layout="none"` | 自動調整しない。`fig.subplots_adjust(...)` で手で決める |

```python
fig, axes = plt.subplots(2, 4, figsize=(10, 5), layout="compressed")   # 画像のグリッド
fig.get_layout_engine().set(w_pad=0.02, h_pad=0.02, wspace=0, hspace=0)   # constrained 系の余白
fig.align_labels()                                                      # 複数パネルの y ラベルの位置をそろえる
```

`layout="constrained"` の図で `fig.tight_layout()` を呼ばない。`The figure layout has changed to tight` の警告が出て、その後は constrained が無効になる。

### 1.3 不規則なグリッド（GridSpec・SubFigure）

```python
# 幅の比を指定
fig, axes = plt.subplots(1, 3, figsize=(10, 3), width_ratios=[3, 1, 1], layout="constrained")

# GridSpec: 行や列をまたぐ Axes
gs = fig.add_gridspec(3, 3)
ax_main = fig.add_subplot(gs[:2, :])
ax_a = fig.add_subplot(gs[2, 0]); ax_b = fig.add_subplot(gs[2, 1:])

# 入れ子: 1 マスをさらに分割
gs0 = fig.add_gridspec(1, 2)
gs_left = gs0[0].subgridspec(2, 2)

# SubFigure: 左右で別々に suptitle / colorbar を持たせる
fig = plt.figure(figsize=(10, 4), layout="constrained")
sf_l, sf_r = fig.subfigures(1, 2, width_ratios=[2, 1])
axl = sf_l.subplots(2, 2); sf_l.suptitle("(a) Tiles")
axr = sf_r.subplots(); sf_r.suptitle("(b) Score")
```

3.10 から、`subfigures` の並び順は行優先（row-major）になった。

### 1.4 2 本目の軸（twin / secondary axis）

```python
ax2 = ax.twinx()                         # 同じ x を共有し、右に別の y 軸（別の量を重ねる）
ax2.plot(x, lr, color="C1"); ax2.set_ylabel("lr")

# 同じ量の単位を変えた目盛り（変換関数とその逆関数を渡す）
sec = ax.secondary_xaxis("top", functions=(lambda um: um / 1000, lambda mm: mm * 1000))
sec.set_xlabel("mm")
```

`twinx` は凡例が別々になる。1 つにまとめるには、両方の `get_legend_handles_labels()` を連結して `ax.legend(h1 + h2, l1 + l2)`。

### 1.5 挿入図（inset）と拡大枠

```python
axins = ax.inset_axes([0.55, 0.55, 0.4, 0.4])     # 親 Axes 内の相対座標 [x, y, 幅, 高さ]
axins.imshow(thumb)
axins.set(xlim=(x0, x1), ylim=(y1, y0), xticks=[], yticks=[])   # imshow は y が下向きなので逆順
ax.indicate_inset_zoom(axins, edgecolor="k")      # 親に拡大範囲の枠と接続線を描く
```

### 1.6 正規化（norm）とカラーバー

| norm | 用途 |
|:---|:---|
| `Normalize(vmin, vmax)` | 線形（既定） |
| `LogNorm(vmin, vmax)` | 桁が広い正の値（カウント、p 値の逆数など）。0 以下は描けない |
| `SymLogNorm(linthresh, vmin=, vmax=)` | 0 付近は線形、外側は対数。正負を含む |
| `TwoSlopeNorm(vcenter=0, vmin=, vmax=)` | 中心を指定して左右で別の傾き（非対称な正負の値） |
| `CenteredNorm(vcenter=0)` | 中心から対称な範囲を自動で取る |
| `PowerNorm(gamma)` | べき乗で強調 |
| `BoundaryNorm(boundaries, ncolors)` | 区間ごとに離散色（グレード分類など） |

```python
from matplotlib import colors
im = ax.imshow(fc, cmap="RdBu_r", norm=colors.TwoSlopeNorm(vcenter=0, vmin=-2, vmax=5))
cb = fig.colorbar(im, ax=ax, extend="both", shrink=0.8, label="log2 FC")   # extend: 範囲外を三角で表す

# 離散カラーバー（例: 病理グレード 0〜3）
cmap = plt.get_cmap("viridis").resampled(4)
norm = colors.BoundaryNorm([-0.5, 0.5, 1.5, 2.5, 3.5], cmap.N)
im = ax.imshow(grade_map, cmap=cmap, norm=norm, interpolation="nearest")
cb = fig.colorbar(im, ax=ax, ticks=[0, 1, 2, 3])

# 複数パネルで 1 本のカラーバーを共有（vmin/vmax をそろえること）
fig.colorbar(im, ax=axes.ravel().tolist())

# Axes の高さにぴったり合わせる（layout を使わない場合）
from mpl_toolkits.axes_grid1 import make_axes_locatable
cax = make_axes_locatable(ax).append_axes("right", size="5%", pad=0.05)
fig.colorbar(im, cax=cax)
```

### 1.7 カラーマップを作る・登録する

```python
import matplotlib as mpl
from matplotlib.colors import ListedColormap, LinearSegmentedColormap

cmap = mpl.colormaps["magma"]                               # 取得（matplotlib.cm.get_cmap は非推奨で 3.11 で削除予定。plt.get_cmap は使える）
cmap8 = cmap.resampled(8)                                   # 8 段階に離散化
cmap_bad = cmap.with_extremes(bad="lightgray", under="k", over="w")   # NaN / 範囲外の色

tissue = ListedColormap(["#ffffff", "#e377c2", "#1f77b4", "#2ca02c"], name="tissue")   # カテゴリ
wr = LinearSegmentedColormap.from_list("white_red", ["white", "darkred"], N=256)       # 連続

mpl.colormaps.register(tissue)                              # 以後 cmap="tissue" で使える（同名の再登録は force=True）
```

`imshow` で NaN の画素を灰色にするには、`np.ma.masked_invalid(arr)` か NaN を含む配列をそのまま渡し、`with_extremes(bad=...)` を使う。

### 1.8 凡例の細かい制御

```python
# 手で凡例を作る（描いていない要素も載せられる）
from matplotlib.lines import Line2D
from matplotlib.patches import Patch
handles = [Patch(facecolor="C0", label="tumor"), Line2D([], [], color="k", ls="--", label="threshold")]
ax.legend(handles=handles, frameon=False)

# 図の外（constrained レイアウトなら領域を確保してくれる）
fig.legend(handles, labels, loc="outside right upper")
ax.legend(loc="upper left", bbox_to_anchor=(1.02, 1), borderaxespad=0)   # Axes の右外

# scatter の点の大きさ・色の凡例
sc = ax.scatter(x, y, s=sizes, c=vals)
ax.legend(*sc.legend_elements(prop="sizes", num=4), title="n cells")

# 重複ラベルを 1 つにまとめる
h, l = ax.get_legend_handles_labels()
uniq = dict(zip(l, h)); ax.legend(uniq.values(), uniq.keys())
```

### 1.9 目盛りと軸のスケール

```python
from matplotlib import ticker
ax.xaxis.set_major_locator(ticker.MaxNLocator(nbins=5, integer=True))   # 目盛りを最大 5 個、整数だけ
ax.xaxis.set_major_locator(ticker.MultipleLocator(10))                  # 10 刻み
ax.yaxis.set_major_formatter(ticker.PercentFormatter(xmax=1))           # 0.25 → 25%
ax.yaxis.set_major_formatter(ticker.FuncFormatter(lambda v, pos: f"{v/1e3:.0f}k"))
ax.yaxis.set_major_formatter(ticker.EngFormatter(unit="B"))             # 1.5 kB, 2 MB ...
ax.set_yscale("log"); ax.set_xscale("symlog", linthresh=1)             # 0 や負を含む場合は symlog
ax.tick_params(axis="x", labelrotation=45, labelsize=8)
ax.set_xticks(pos, names, rotation=45, ha="right")                      # 位置とラベルを同時に
```

日付軸:

```python
import matplotlib.dates as mdates
loc = mdates.AutoDateLocator()
ax.xaxis.set_major_locator(loc)
ax.xaxis.set_major_formatter(mdates.ConciseDateFormatter(loc))          # 年月日の重複表記を省く
```

### 1.10 注釈: 有意差のブラケット・棒の値

有意差の記号は、検定と多重比較補正を自分で行い、その結果を描くだけにする（描画側で検定しない）。

```python
def sig_bracket(ax, x1, x2, y, text, h=0.02):
    """x1〜x2 の間にブラケットと記号を描く。y はデータ座標。"""
    ax.plot([x1, x1, x2, x2], [y, y + h, y + h, y], lw=1, c="k")
    ax.text((x1 + x2) / 2, y + h, text, ha="center", va="bottom")

sig_bracket(ax, 0, 1, df.score.max() * 1.05, "**")

bars = ax.bar(names, vals)
ax.bar_label(bars, fmt="%.2f", padding=2)          # 棒の上に値
```

seaborn のカテゴリ軸では、カテゴリの位置が 0, 1, 2, ... になる（`order=` で並びを固定しておく）。

### 1.11 図形（patches）を重ねる

```python
from matplotlib.patches import Rectangle, Polygon, Circle
# WSI サムネイル上に、タイル（レベル 0 座標）を縮小率で変換して枠を描く
s = thumb.shape[1] / slide_w
ax.imshow(thumb)
for x0, y0 in coords:
    ax.add_patch(Rectangle((x0 * s, y0 * s), tile * s, tile * s, fill=False, ec="lime", lw=0.5))

ax.add_patch(Polygon(contour_xy * s, closed=True, fill=False, ec="red"))   # アノテーションの輪郭 (N, 2)
```

パッチが数千個以上なら `matplotlib.collections.PatchCollection` にまとめて 1 回で追加すると速い。

### 1.12 大量データを描く

| 方法 | 用途 |
|:---|:---|
| `rasterized=True`（Artist 単位） | PDF/SVG の中で、その要素だけ画像にする。点が多い散布図の PDF を軽くする |
| `ax.set_rasterization_zorder(z)` | zorder が z 未満の Artist をまとめて画像にする |
| `LineCollection` | 何千本もの線（学習曲線、軌跡）を 1 つの Artist で描く |
| `ax.hexbin(x, y, gridsize=100, bins="log")` | 点の密度を六角形のビンで描く。数十万点以上の UMAP など |
| `sns.histplot(x=, y=)` | 2 次元ヒストグラム |
| datashader | 数千万点以上。別パッケージ（`.venv` には入っていない） |

```python
from matplotlib.collections import LineCollection
segs = [np.column_stack([epochs, c]) for c in curves]      # 各線の (N, 2)
ax.add_collection(LineCollection(segs, colors="C0", alpha=0.1, lw=0.5))
ax.autoscale()                                             # add_collection は軸範囲を更新しない
```

PDF に `rasterized=True` の要素を含める場合、画像部分の解像度は `savefig(dpi=...)` で決まる。

### 1.13 スタイル・rcParams を管理する

```python
import matplotlib as mpl
mpl.matplotlib_fname()            # 読み込まれた matplotlibrc の場所
mpl.get_configdir()               # 設定ディレクトリ（既定 ~/.config/matplotlib、MPLCONFIGDIR で変更）
mpl.get_cachedir()                # キャッシュ（フォント一覧など。既定 ~/.cache/matplotlib）

plt.style.available                               # 同梱スタイル一覧
with plt.style.context(["seaborn-v0_8-whitegrid", "./paper.mplstyle"]):   # ブロック内だけ適用
    fig, ax = plt.subplots()
with mpl.rc_context({"font.size": 8, "lines.linewidth": 1}):
    ...
mpl.rcdefaults()                                  # 既定値に戻す
```

`paper.mplstyle` は `matplotlibrc` と同じ書式（`key: value`）。プロジェクトに置いておき、図のスクリプトから `plt.style.use("path/to/paper.mplstyle")` で読むと設定を共有できる。

`sns.set_theme()` は rcParams を書き換える。Matplotlib のスタイルと併用すると、後から適用したほうが勝つ。

### 1.14 保存: サイズ・解像度・フォント

| 設定 | 意味 |
|:---|:---|
| `figsize=(w, h)` | インチ単位。論文の 1 段幅はおよそ 3.3〜3.5 インチ、2 段幅はおよそ 7 インチ（投稿規定を確認） |
| `dpi` | 1 インチあたりの画素数。PNG の画素数 = figsize × dpi |
| `font.size` | ポイント単位。figsize を大きくして縮小すると文字が小さくなるので、最終サイズで作る |
| `pdf.fonttype: 42` | PDF に TrueType で埋め込む（既定 3 は Type 3 で、編集ソフトで文字として扱いにくい） |
| `svg.fonttype: "none"` | SVG の文字をパスにせず文字のまま出す（既定 `path`）。閲覧側にフォントが必要 |
| `savefig.transparent` / `transparent=True` | 背景を透明にする |

```python
fig.savefig("fig.png", dpi=300, facecolor="white")   # 背景を明示（透明な PNG が暗いスライドで見えなくなるのを防ぐ）
fig.savefig("fig.pdf", metadata={"CreationDate": None})   # 作成日時を入れない（再生成しても差分が出にくい）
```

### 1.15 フォントを追加する

```python
from matplotlib import font_manager as fm
fm.fontManager.addfont("/path/to/NotoSansCJKjp-Regular.otf")   # 実行中のプロセスに追加
prop = fm.FontProperties(fname="/path/to/NotoSansCJKjp-Regular.otf")
plt.rcParams["font.family"] = prop.get_name()

sorted({f.name for f in fm.fontManager.ttflist})                # Matplotlib が認識しているフォント名
fm.findfont("Noto Sans CJK JP")                                 # 実際に使われるファイル
```

`addfont` はそのプロセスだけに効く。コンテナにフォントが入っていない場合は、フォントファイルをプロジェクト内に置いて `addfont` するのが確実。

### 1.16 アニメーション

```python
from matplotlib.animation import FuncAnimation
fig, ax = plt.subplots()
im = ax.imshow(frames[0])
def update(i):
    im.set_data(frames[i]); return [im]
ani = FuncAnimation(fig, update, frames=len(frames), interval=100, blit=True)
ani.save("out.gif", writer="pillow", fps=10)       # Pillow だけで書ける
ani.save("out.mp4", writer="ffmpeg", fps=10)       # ffmpeg が PATH に必要
# ノートブック: from IPython.display import HTML; HTML(ani.to_jshtml())
```

### 1.17 seaborn: 推定量・誤差棒・順序・色

```python
sns.barplot(data=df, x="dose", y="score", estimator="median", errorbar=("pi", 50))   # 中央値 ± 四分位範囲
sns.pointplot(data=df, x="dose", y="score", hue="organ", errorbar="se", dodge=0.3)
sns.lineplot(data=log, x="epoch", y="auc", errorbar=("ci", 95), n_boot=1000, seed=0)
```

| `errorbar=` | 意味 |
|:---|:---|
| `"sd"` / `("sd", 2)` | 標準偏差（の倍数）。データのばらつき |
| `"se"` / `("se", 2)` | 標準誤差（の倍数）。平均の推定の不確かさ |
| `"ci"` / `("ci", 95)` | ブートストラップ信頼区間（既定）。`n_boot` と `seed` で再現性を固定 |
| `"pi"` / `("pi", 50)` | パーセンタイル区間 |
| `None` | 誤差棒なし（計算しないので速い） |

図の説明（キャプション）には、どの `errorbar` を使ったかを書く。

```python
order = ["ctrl", "low", "mid", "high"]
pal = {"liver": "C0", "kidney": "C1"}                  # 値→色の辞書で、図をまたいで色を固定
sns.boxplot(data=df, x="dose", y="score", hue="organ", order=order, hue_order=list(pal), palette=pal)
sns.stripplot(data=df, x="dose", y="score", hue="organ", order=order, hue_order=list(pal),
              palette=pal, dodge=True, legend=False, size=2)
```

0.13 のカテゴリ関数の変更点:

| 項目 | 内容 |
|:---|:---|
| `native_scale=True` | 数値や日付の x を、等間隔のカテゴリではなく実際の値の位置に置く |
| `hue` だけで色分け | `palette` を使うなら `hue` も指定する（指定しないと FutureWarning） |
| `legend=False` / `"brief"` / `"full"` | 凡例の制御 |
| `fill=False` | 箱・バイオリンを線だけで描く |
| `log_scale=True` | 軸を対数にする |

### 1.18 seaborn: グリッド系

```python
g = sns.FacetGrid(df, col="organ", row="dose", height=2.5, sharey=False)
g.map_dataframe(sns.histplot, x="score", hue="label")
g.refline(x=0.5)                                  # 全パネルに参照線
g.set_axis_labels("score", "count"); g.add_legend()
for ax in g.axes.flat: ax.set_xlim(0, 1)          # 個別の Axes は g.axes

g = sns.PairGrid(df[cols + ["label"]], hue="label", corner=True)
g.map_lower(sns.scatterplot, s=3); g.map_diag(sns.kdeplot)

g = sns.JointGrid(data=df, x="umap1", y="umap2", hue="cluster")
g.plot_joint(sns.scatterplot, s=3); g.plot_marginals(sns.kdeplot)
```

`map` は位置引数で列を受け取る関数用、`map_dataframe` は `data=` を受け取る seaborn の関数用。

### 1.19 seaborn objects インターフェース（`so.Plot`）

層を重ねて描く新しい書き方（0.12〜）。0.13 時点でも API は発展途上。

```python
import seaborn.objects as so
(
    so.Plot(log, x="epoch", y="auc", color="model")
    .add(so.Line(), so.Agg())                         # 平均の線
    .add(so.Band(), so.Est(errorbar="sd"))            # ± SD の帯
    .facet(col="organ", wrap=3)
    .label(x="epoch", y="AUC")
    .layout(size=(8, 3))
    .save("auc.png", dpi=200, bbox_inches="tight")
)
so.Plot(df, x="dose", y="score").add(so.Dots(), so.Jitter(0.3)).on(ax).plot()   # 既存の Axes に描く
```

## Part 2. トラブル対応

### 2.1 日本語フォントを入れたのに □ のまま

- **症状**: フォントを入れた、または `font.family` を設定したのに文字が □ になり、`findfont: Font family '...' not found` や `Glyph ... missing from font(s)` の警告が出る。
- **確認**:
```python
import matplotlib as mpl
from matplotlib import font_manager as fm
print(mpl.get_cachedir())                                       # フォント一覧キャッシュの場所
print([f.name for f in fm.fontManager.ttflist if "CJK" in f.name or "Gothic" in f.name])
print(fm.findfont(mpl.rcParams["font.family"][0]))
```
```bash
fc-list :lang=ja family      # OS（コンテナ内）にあるか
ls ~/.cache/matplotlib/      # fontlist-v390.json がキャッシュ
```
- **原因**: Matplotlib はフォント一覧を `fontlist-v390.json` にキャッシュする。キャッシュ作成後に入れたフォントは見えない。またはコンテナ内にフォントがない（ホストの `fc-list` とは別）。
- **対処**: キャッシュファイルを消して Python（カーネル）を再起動する。コンテナにない場合は、フォントファイルを置いて `fm.fontManager.addfont(path)`（1.15）。`font.family` には `ttflist` に出てくる名前をそのまま書く。

### 2.2 「temporary cache directory」の警告が毎回出る・起動が遅い

- **症状**: `Matplotlib created a temporary cache directory at /tmp/matplotlib-... because there was an issue with the default path` と出て、import のたびにフォント一覧の作成で数秒〜数十秒かかる。
- **確認**:
```bash
echo $HOME $MPLCONFIGDIR
ls -ld ~/.config ~/.cache
python -c "import matplotlib as mpl; print(mpl.get_configdir(), mpl.get_cachedir())"
```
- **原因**: 設定・キャッシュ用のディレクトリに書き込めない（コンテナ内で `$HOME` が読み取り専用、`--no-home` や `--containall` で起動した、など）。一時ディレクトリはプロセス終了時に消えるので、毎回作り直しになる。
- **対処**: 書き込めるディレクトリを `MPLCONFIGDIR` に指定する（例: `export APPTAINERENV_MPLCONFIGDIR=/path/under/honzawa/.mplconfig`）。

### 2.3 バックエンドのエラー（ヘッドレス・ノートブック）

- **症状**: `cannot connect to X server`、`TclError: no display name`、`%matplotlib widget` で `No module named 'ipympl'`、`plt.show()` で何も出ない（`FigureCanvasAgg is non-interactive` の警告）。
- **確認**:
```python
import matplotlib; print(matplotlib.get_backend())
import os; print(os.environ.get("MPLBACKEND"), os.environ.get("DISPLAY"))
import importlib.util; print(importlib.util.find_spec("ipympl"))
```
- **原因**: ディスプレイのない環境で GUI バックエンドを使おうとしている。`%matplotlib widget` には ipympl が必要（テンプレートの `.venv` には入っていない）。スクリプトでは Agg なので `plt.show()` は何もしない。
- **対処**: スクリプトは `MPLBACKEND=Agg` か `matplotlib.use("Agg")` にして `savefig` で保存する。ノートブックは `%matplotlib inline`（既定）を使う。対話的な拡大・移動が必要なら、ipympl を依存に追加するかプロジェクトで決めてから使う。

### 2.4 ループで図を作るとメモリが増え続ける

- **症状**: 何百枚も保存するスクリプトでメモリ使用量が増え続け、`More than 20 figures have been opened` の警告や OOM で落ちる。
- **確認**:
```python
import matplotlib.pyplot as plt
print(len(plt.get_fignums()))     # 開いたままの図の数
```
```bash
sacct -j <jobid> --format=JobID,State,MaxRSS,ReqMem
```
- **原因**: pyplot は作った図を閉じるまで保持する。`ax.clear()` だけでは Figure は残る。
- **対処**: 保存ごとに `plt.close(fig)`。大量に作るときは、1 つの Figure を使い回して `im.set_data()` / `line.set_data()` で中身だけ差し替えると速い。pyplot を使わずに `from matplotlib.figure import Figure; fig = Figure()` で作ると pyplot に登録されないので、閉じ忘れが起きない（`fig.savefig` は使える）。

### 2.5 PDF / SVG が巨大・開くと重い

- **症状**: 散布図やヒートマップの PDF が数十 MB になる。Illustrator やビューアで開くのが遅い。
- **確認**:
```python
print(len(ax.collections[0].get_offsets()))       # 点の数
for a in ax.get_children(): print(type(a).__name__, a.get_rasterized())
```
```bash
ls -lh fig.pdf
```
- **原因**: ベクター形式では点・セル 1 個ずつが図形として書かれる。`pcolormesh` や数十万点の `scatter` は重くなる。
- **対処**: 点が多い Artist に `rasterized=True` を付け、軸・文字はベクターのまま残す（1.12）。`savefig(dpi=300)` で画像部分の解像度を決める。ヒートマップは `imshow`（1 枚の画像になる）を使う。

### 2.6 カラーバーの高さが Axes と合わない・Axes が縮む

- **症状**: カラーバーが Axes より長い／短い。`imshow` のパネルにカラーバーを付けると、そのパネルだけ小さくなる。
- **確認**:
```python
print(fig.get_layout_engine())                 # None / Constrained / Tight
print(ax.get_aspect())                         # imshow は 'equal'
```
- **原因**: `fig.colorbar(im, ax=ax)` は Axes から場所を奪って作る。縦横比が固定の Axes では、高さがそろわない。
- **対処**: `layout="constrained"` か `"compressed"` を使い、`fig.colorbar(im, ax=ax, shrink=...)` で調整する。高さをぴったり合わせたいときは `make_axes_locatable` か `ax.inset_axes([1.02, 0, 0.05, 1])` を `cax=` に渡す（1.6）。複数パネル共通なら `ax=axes.ravel().tolist()`。

### 2.7 ラベルや目盛りが重なる

- **症状**: x 軸のカテゴリ名が重なる、サブプロットのタイトルと上の段の x ラベルがぶつかる、外に置いた凡例が保存時に切れる。
- **確認**:
```python
print(fig.get_size_inches(), fig.dpi, plt.rcParams["font.size"])
print(len(ax.get_xticklabels()))
```
- **原因**: figsize に対して文字が大きい、目盛りが多い、レイアウトエンジンを使っていない。`bbox_to_anchor` で外に出した凡例は、レイアウト計算に入らない場合がある。
- **対処**: `layout="constrained"` にする。回転は `ax.set_xticks(pos, names, rotation=45, ha="right")`。目盛りを減らすなら `MaxNLocator`。凡例は `fig.legend(loc="outside right upper")` か、`savefig(bbox_inches="tight")` で外側も含めて保存する。

### 2.8 日付軸の表示がおかしい

- **症状**: x 軸が `1.7e9` のような数値になる、日付ラベルが重なる、文字列の日付が等間隔のカテゴリとして並ぶ。
- **確認**:
```python
print(df["date"].dtype)                 # object なら文字列のまま
print(type(ax.xaxis.get_major_formatter()).__name__)
```
- **原因**: 日付が文字列のまま、または UNIX 時刻の数値のまま渡されている。
- **対処**: `pd.to_datetime(df["date"])` で変換してから描く。ラベルは `ConciseDateFormatter`（1.9）。seaborn のカテゴリ関数で日付の間隔を保つなら `native_scale=True`。

### 2.9 seaborn 0.13 で非推奨の警告が大量に出る

- **症状**: `Passing palette without assigning hue is deprecated`、`The errwidth parameter is deprecated`、`The scale parameter is deprecated`、`ci is deprecated` などの警告。
- **確認**:
```python
import warnings, seaborn as sns
print(sns.__version__)
warnings.simplefilter("error", FutureWarning)   # 一時的に例外にして、出どころの行を特定する
```
- **原因**: 0.12〜0.13 で引数が整理された。古い書き方は動くが警告が出て、今後の版で削除される。多くは FutureWarning だが、`pointplot` の `scale` / `join` は UserWarning で出る。
- **対処**: 下の表のとおり書き換える。警告を消すだけの `filterwarnings("ignore")` は、別の警告も隠すので避ける。

| 古い書き方 | 0.13 の書き方 |
|:---|:---|
| `palette=...`（`hue` なし） | `hue=<x と同じ列>, palette=..., legend=False` |
| `ci=95` / `ci="sd"` / `ci=None` | `errorbar=("ci", 95)` / `"sd"` / `None` |
| `errwidth=`, `errcolor=` | `err_kws={"linewidth": ..., "color": ...}` |
| `violinplot(scale=...)` | `density_norm=...` |
| `violinplot(scale_hue=...)` | `common_norm=...` |
| `violinplot(bw=...)` | `bw_method=` / `bw_adjust=` |
| `pointplot(join=False)` | `linestyle="none"` |
| `pointplot(scale=...)` | `markersize=` / `linewidth=` など |

### 2.10 ノートブックの図がぼやける・大きさが保存時と違う

- **症状**: ノートブック上の図がにじむ。表示と保存した PNG で余白や大きさが違う。
- **確認**:
```python
%config InlineBackend.figure_formats
%config InlineBackend.print_figure_kwargs
import matplotlib.pyplot as plt; print(plt.rcParams["figure.dpi"])
```
- **原因**: inline 表示は既定で PNG（`figure.dpi` の解像度）。高解像度ディスプレイでは粗く見える。inline は既定で `bbox_inches="tight"` で描画するので、`savefig` の既定（`standard`）と余白が異なる。
- **対処**: `%config InlineBackend.figure_formats = {"retina"}`（または `{"svg"}`）。見た目をそろえたいときは `savefig(..., bbox_inches="tight")` にするか、保存した画像を `IPython.display.Image` で表示して確認する。

### 2.11 図ごとに同じグループの色が変わる

- **症状**: 図 A では liver が青、図 B ではオレンジになる。サブセットで描くと色がずれる。
- **確認**:
```python
print(df["organ"].unique())       # 出現順。seaborn は hue_order を省くとこの順（数値ならソート順）
```
- **原因**: 色はカテゴリの出現順で割り当てられる。データの並びやサブセットで順序が変わる。
- **対処**: 値→色の辞書を 1 か所で定義して `palette=pal, hue_order=list(pal)` を毎回渡す（1.17）。Matplotlib だけで描く場合も同じ辞書から `color=pal[k]` で指定する。`pd.Categorical(..., categories=order)` にしておくと、seaborn はその順序を使う。

### 2.12 マスクやラベル画像の境界の色が混ざる

- **症状**: セグメンテーションのラベル画像や離散値のマップを縮小表示すると、境界に存在しないラベルの色が出る。拡大するとぼやける。
- **確認**:
```python
print(im.get_interpolation(), im.get_interpolation_stage())   # 既定 'auto'
print(label_map.dtype, np.unique(label_map)[:10])
```
- **原因**: `imshow` は表示サイズに合わせて補間する。データ空間（`interpolation_stage="data"`）で補間されると、ラベル値の中間の値が生まれ、別の色になる。
- **対処**: `ax.imshow(label_map, cmap=cmap, norm=norm, interpolation="nearest")`。縮小時のちらつきを抑えたいときは `interpolation_stage="rgba"`（色を付けてから補間）も試す。定量に使う図は、描画前に自分で縮小（`cv2.INTER_NEAREST` など）してから表示する。
