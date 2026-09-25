# scikit-learn チートシート

> 対象: scikit-learn 1.x（このマシンの `.venv` は 1.8）
> 公式: https://scikit-learn.org/stable/

## 1. 共通の API

すべての推定器が同じメソッドを持つ。

| メソッド | 意味 |
|:---|:---|
| `fit(X, y)` | 学習する |
| `predict(X)` | 予測する（クラスラベルや値） |
| `predict_proba(X)` | クラスごとの確率 |
| `decision_function(X)` | スコア（SVM などで確率の代わりに使う） |
| `transform(X)` / `fit_transform(X)` | 前処理・次元削減 |
| `score(X, y)` | 既定の指標（分類は accuracy、回帰は R²） |

`X` は (サンプル数, 特徴量数) の 2次元配列、`y` は (サンプル数,) の 1次元配列。

## 2. データの分割

```python
from sklearn.model_selection import train_test_split

X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, stratify=y, random_state=0)
```

### グループを考慮した分割（病理ではほぼ必須）

同じ動物・同じスライドのパッチが train と test に分かれると、性能が過大に評価される（リーク）。

```python
from sklearn.model_selection import GroupShuffleSplit, GroupKFold, StratifiedGroupKFold

gss = GroupShuffleSplit(n_splits=1, test_size=0.2, random_state=0)
tr_idx, te_idx = next(gss.split(X, y, groups=animal_ids))

cv = StratifiedGroupKFold(n_splits=5, shuffle=True, random_state=0)   # クラス比も保つ
for tr_idx, te_idx in cv.split(X, y, groups=animal_ids):
    ...
```

| 分割 | 使いどころ |
|:---|:---|
| `KFold` | 普通の k 分割 |
| `StratifiedKFold` | クラス比を保つ（分類の既定） |
| `GroupKFold` | 同じグループを同じ fold に |
| `StratifiedGroupKFold` | グループ + クラス比 |
| `LeaveOneGroupOut` | 1グループずつ test にする（試験単位・施設単位の検証） |
| `TimeSeriesSplit` | 時系列 |

## 3. 前処理

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler, LabelEncoder, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.decomposition import PCA

scaler = StandardScaler().fit(X_tr)          # train だけで fit する
X_tr_s, X_te_s = scaler.transform(X_tr), scaler.transform(X_te)

y_id = LabelEncoder().fit_transform(y_str)   # 文字列ラベル → 0, 1, 2, ...
Z = PCA(n_components=50, random_state=0).fit_transform(X)
```

**test を含めて fit しない。** `fit` は train だけ、test には `transform` だけを使う。Pipeline を使えば自然に守れる。

## 4. Pipeline

```python
from sklearn.pipeline import make_pipeline
from sklearn.linear_model import LogisticRegression

clf = make_pipeline(StandardScaler(), LogisticRegression(max_iter=2000, C=1.0))
clf.fit(X_tr, y_tr)
prob = clf.predict_proba(X_te)[:, 1]
```

列ごとに別の前処理をする場合:

```python
from sklearn.compose import ColumnTransformer
pre = ColumnTransformer([
    ("num", StandardScaler(), ["age", "weight"]),
    ("cat", OneHotEncoder(handle_unknown="ignore"), ["sex", "dose"]),
])
clf = make_pipeline(pre, LogisticRegression(max_iter=2000))
```

## 5. よく使うモデル

| 用途 | モデル | 備考 |
|:---|:---|:---|
| 線形分類 / リニアプローブ | `LogisticRegression` | 基盤モデルの特徴量の評価の定番。`C` で正則化の強さ（小さいほど強い） |
| 線形回帰 | `Ridge`, `Lasso` | |
| 木系 | `RandomForestClassifier`, `HistGradientBoostingClassifier` | スケーリング不要。HGB は大きいデータでも速い |
| 近傍 | `KNeighborsClassifier` | k-NN 評価。特徴量は L2 正規化してから |
| SVM | `SVC(probability=True)` | 大きいデータでは遅い |
| 異常検知 | `IsolationForest`, `OneClassSVM`, `LocalOutlierFactor(novelty=True)` | |
| クラスタリング | `KMeans`, `MiniBatchKMeans`, `AgglomerativeClustering`, `DBSCAN`, `HDBSCAN` | |
| 次元削減 | `PCA`, `TSNE` | UMAP は別パッケージ（`umap-learn`） |

`class_weight="balanced"` でクラスの偏りを補正できる（LogisticRegression、SVC、RandomForest など）。

## 6. 評価指標

```python
from sklearn.metrics import (
    accuracy_score, balanced_accuracy_score, f1_score, roc_auc_score,
    average_precision_score, confusion_matrix, classification_report,
)

roc_auc_score(y_te, prob)                               # 2クラス: 陽性の確率を渡す
roc_auc_score(y_te, prob_mat, multi_class="ovr", average="macro")   # 多クラス
average_precision_score(y_te, prob)                     # PR-AUC（陽性が少ないとき）
f1_score(y_te, pred, average="macro")                   # macro: クラスを平等に扱う
balanced_accuracy_score(y_te, pred)                     # クラスの偏りに強い accuracy
print(classification_report(y_te, pred, digits=3))
confusion_matrix(y_te, pred, normalize="true")          # 行ごとに割合にする
```

| 指標 | 使いどころ |
|:---|:---|
| accuracy | クラスが均衡しているとき |
| balanced accuracy / macro-F1 | クラスが偏っているとき |
| ROC-AUC | 閾値に依存しない評価 |
| PR-AUC | 陽性が非常に少ないとき（異常検知など） |

### 曲線を描く

```python
from sklearn.metrics import RocCurveDisplay, ConfusionMatrixDisplay
RocCurveDisplay.from_predictions(y_te, prob)
ConfusionMatrixDisplay.from_predictions(y_te, pred, normalize="true")
```

## 7. 交差検証とハイパーパラメータ探索

```python
from sklearn.model_selection import cross_val_score, cross_validate, GridSearchCV

scores = cross_val_score(clf, X, y, cv=cv, groups=animal_ids, scoring="roc_auc", n_jobs=-1)
print(scores.mean(), scores.std())

grid = GridSearchCV(
    clf, {"logisticregression__C": [0.01, 0.1, 1, 10]},     # パイプライン内は「ステップ名__引数」
    cv=cv, scoring="roc_auc", n_jobs=-1,
)
grid.fit(X, y, groups=animal_ids)
grid.best_params_, grid.best_score_
```

`scoring` に使える名前: `"accuracy"`, `"balanced_accuracy"`, `"f1_macro"`, `"roc_auc"`, `"roc_auc_ovr"`, `"average_precision"` など。

## 8. 保存

```python
import joblib
joblib.dump(clf, "model.joblib")
clf = joblib.load("model.joblib")      # 同じバージョンの scikit-learn で読む
```

## 9. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| test の性能が高すぎる | リーク。グループ単位で分割しているか、前処理を test 込みで fit していないか確認する |
| `ConvergenceWarning` | `max_iter` を増やす、特徴量を標準化する |
| `roc_auc_score` でエラー | 確率ではなくラベルを渡している、または test に1クラスしかない |
| `n_jobs=-1` でノードの CPU を使い切る | Slurm で確保した CPU 数より多く使ってしまう。`n_jobs=int(os.environ["SLURM_CPUS_PER_TASK"])` のように指定する |
| 読み込んだモデルで警告・エラー | 保存時と scikit-learn のバージョンが違う |
| 多クラスの指標が思ったより低い | `average="micro"` と `"macro"` の違い。どちらで報告するか決めておく |
