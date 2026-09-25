# scikit-learn 詳細編（応用・トラブル対応）

> 基本編: [sklearn.md](sklearn.md)
> 対象バージョン: scikit-learn 1.8（動作確認は 1.8.0）

## Part 1. 応用・上級機能

### 1.1 Pipeline の応用

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, FunctionTransformer
from sklearn.decomposition import PCA
from sklearn.linear_model import LogisticRegression

pipe = Pipeline([
    ("log", FunctionTransformer(np.log1p, feature_names_out="one-to-one")),
    ("scale", StandardScaler()),
    ("pca", PCA(n_components=0.95)),          # 0〜1 の小数なら累積寄与率で次元数を決める
    ("clf", LogisticRegression(max_iter=2000)),
], memory="cache_dir")                         # 前段の fit 結果をキャッシュ（探索で同じ前処理を繰り返すとき）

pipe.set_output(transform="pandas")            # transform の出力を DataFrame（列名付き）にする
pipe[:-1].transform(X_te)                      # 最後の推定器を除いた前処理だけ適用
pipe[:-1].get_feature_names_out()              # 前処理後の列名
pipe.named_steps["pca"].explained_variance_ratio_
pipe.set_params(clf__C=0.1)

import sklearn
sklearn.set_config(transform_output="pandas")  # 全体に適用する場合
```

### 自作の変換器

```python
from sklearn.base import BaseEstimator, TransformerMixin, OneToOneFeatureMixin
from sklearn.utils.validation import check_is_fitted, validate_data   # validate_data は 1.6+

class ClipByQuantile(OneToOneFeatureMixin, TransformerMixin, BaseEstimator):
    def __init__(self, q=0.99):                # __init__ では引数を保存するだけ（検証や加工はしない）
        self.q = q

    def fit(self, X, y=None):
        X = validate_data(self, X)             # n_features_in_ / feature_names_in_ を設定
        self.upper_ = np.quantile(X, self.q, axis=0)   # 学習で決まる値は末尾に _
        return self

    def transform(self, X):
        check_is_fitted(self)
        X = validate_data(self, X, reset=False)   # 列数・列名が fit 時と同じか確認
        return np.minimum(X, self.upper_)
```

- `__init__` の引数名と属性名を同じにする（`get_params` / `clone` / GridSearch が依存する）。
- `OneToOneFeatureMixin` を付けると `get_feature_names_out` が入力の列名をそのまま返す。`validate_data` を使わず `feature_names_in_` が無いと、`set_output(transform="pandas")` 時に列名の不一致エラーになる。
- `sklearn.utils.estimator_checks.check_estimator(ClipByQuantile())` で API の規約を確認できる。

### 1.2 メタデータのルーティング（sample_weight・groups）

`sample_weight` などを Pipeline や交差検証の中の特定のステップにだけ渡す仕組み。

```python
import sklearn
sklearn.set_config(enable_metadata_routing=True)

clf = LogisticRegression(max_iter=2000).set_fit_request(sample_weight=True)   # fit で受け取ると宣言
scaler = StandardScaler().set_fit_request(sample_weight=False)                # 受け取らないと宣言
pipe = make_pipeline(scaler, clf)

from sklearn.metrics import make_scorer, roc_auc_score
scorer = make_scorer(roc_auc_score, response_method="predict_proba").set_score_request(sample_weight=True)

cross_validate(pipe, X, y, cv=StratifiedGroupKFold(5), scoring=scorer,
               params={"sample_weight": w, "groups": animal_ids})   # groups も params で渡す
```

- ルーティング有効時は `cross_validate(..., groups=...)` の書き方はエラーになる。`params={"groups": ...}` に変える。
- `GroupKFold` などのグループ分割器は `groups` を自動で要求するので、`set_split_request` は不要。
- 要求を宣言していない推定器に `sample_weight` を渡すとエラーになる（黙って無視されない）。

### 1.3 入れ子の交差検証（nested CV）

ハイパーパラメータ探索と性能評価を同じ fold で行うと、性能が楽観的になる。外側で評価、内側で探索する。

```python
sklearn.set_config(enable_metadata_routing=True)       # 内側の分割にも groups を渡すため

inner = StratifiedGroupKFold(n_splits=3, shuffle=True, random_state=0)
outer = StratifiedGroupKFold(n_splits=5, shuffle=True, random_state=1)
search = GridSearchCV(pipe, {"clf__C": [0.01, 0.1, 1, 10]}, cv=inner, scoring="roc_auc")

res = cross_validate(search, X, y, cv=outer, scoring="roc_auc",
                     params={"groups": animal_ids}, return_estimator=True)
res["test_score"]                                     # 外側 fold の性能（これを報告する）
[e.best_params_ for e in res["estimator"]]            # fold ごとに選ばれたパラメータ
```

ルーティングを有効にしないと、外側の `groups` は内側の `GridSearchCV.fit` に渡らない。内側がグループ分割器だと `The 'groups' parameter should not be None` で止まる。

### 1.4 ハイパーパラメータ探索の応用

```python
from scipy.stats import loguniform, randint
from sklearn.model_selection import RandomizedSearchCV

search = RandomizedSearchCV(
    pipe,
    {"clf__C": loguniform(1e-3, 1e2), "pca__n_components": randint(16, 257)},
    n_iter=40, cv=cv, random_state=0, n_jobs=n_cpu,
    scoring={"auc": "roc_auc", "bacc": "balanced_accuracy"},   # 複数指標
    refit="auc",                                                # どの指標で最良を選ぶか
)
search.fit(X, y, groups=animal_ids)
res = pd.DataFrame(search.cv_results_).sort_values("rank_test_auc")
res[["params", "mean_test_auc", "std_test_auc", "mean_fit_time"]].head()
```

候補を段階的に絞る探索（まだ experimental）:

```python
from sklearn.experimental import enable_halving_search_cv  # noqa: F401  このimportが必要
from sklearn.model_selection import HalvingGridSearchCV, HalvingRandomSearchCV

HalvingGridSearchCV(pipe, grid, factor=3, resource="n_samples", cv=cv, random_state=0)
```

少ないサンプルで全候補を試し、上位 1/factor だけをサンプル数を増やして再評価する。サンプルが少ない段階で良さが出にくいモデル（深い木など）では外れやすい。

### 1.5 確率の較正（calibration）

```python
from sklearn.calibration import CalibratedClassifierCV, CalibrationDisplay, calibration_curve
from sklearn.metrics import brier_score_loss

cal = CalibratedClassifierCV(base_clf, method="isotonic", cv=5)   # 少数データなら "sigmoid"
cal.fit(X_tr, y_tr)

# 学習済みモデルを別データで較正する（1.8 では cv="prefit" は使えない）
from sklearn.frozen import FrozenEstimator
cal = CalibratedClassifierCV(FrozenEstimator(fitted_clf), method="sigmoid").fit(X_cal, y_cal)

prob_true, prob_pred = calibration_curve(y_te, prob, n_bins=10, strategy="quantile")
CalibrationDisplay.from_predictions(y_te, prob, n_bins=10)
brier_score_loss(y_te, prob)                   # 小さいほど良い
```

| method | 向いている場面 |
|:---|:---|
| `"sigmoid"`（Platt） | 較正用データが少ない（数百以下）、歪みが S 字型 |
| `"isotonic"` | データが多い。少ないと過学習する |

ROC-AUC は較正しても変わらない（順位が変わらないため）。確率そのものを使う場合（リスク推定、閾値の解釈）に較正する。

### 1.6 閾値の調整

既定の `predict` は確率 0.5 で切る。クラスが偏っていると最適ではない。

```python
from sklearn.model_selection import TunedThresholdClassifierCV, FixedThresholdClassifier

tuned = TunedThresholdClassifierCV(pipe, scoring="balanced_accuracy", cv=5, random_state=0)
tuned.fit(X_tr, y_tr)
tuned.best_threshold_, tuned.best_score_

fixed = FixedThresholdClassifier(pipe, threshold=0.3).fit(X_tr, y_tr)   # 閾値を固定
```

`scoring` に `make_scorer(recall_score)` などを渡せる。感度を一定以上に保つなど制約付きの閾値は、`roc_curve` の結果から自分で選ぶほうが確実。閾値は test ではなく train（の CV）で決める。

### 1.7 クラス不均衡への対応

| 方法 | 書き方 | 備考 |
|:---|:---|:---|
| 重み付け | `class_weight="balanced"` | まず試す。多くの線形モデル・木・SVM で使える |
| サンプル重み | `fit(X, y, sample_weight=w)` | `HistGradientBoostingClassifier` など class_weight の無いモデルにも |
| 閾値調整 | 1.6 | 確率の順位は良いのに F1 が低いとき |
| 評価指標 | balanced accuracy / PR-AUC / macro-F1 | accuracy は使わない |
| 分割 | `StratifiedKFold` / `StratifiedGroupKFold` | 少数クラスが fold に入らない事故を防ぐ |

オーバーサンプリング（SMOTE など）は別パッケージ（imbalanced-learn）。使う場合は CV の各 fold の train 内でだけ行う。

### 1.8 モデルの解釈（inspection）

```python
from sklearn.inspection import permutation_importance, PartialDependenceDisplay

r = permutation_importance(pipe, X_te, y_te, scoring="roc_auc", n_repeats=10,
                           random_state=0, n_jobs=n_cpu)
imp = pd.Series(r.importances_mean, index=X_te.columns).sort_values(ascending=False)

PartialDependenceDisplay.from_estimator(pipe, X_te, features=["age", ("age", "dose")], kind="average")
```

- permutation importance は相関の強い特徴量があると、どちらも重要度が低く出る（片方を壊しても他方で代用される）。
- `feature_importances_`（木の不純度ベース）は値の種類が多い特徴量を過大評価する。比較には permutation を使う。
- test データで計算すると汎化に効く特徴量、train で計算すると学習に使った特徴量がわかる。

### 1.9 特徴量選択

```python
from sklearn.feature_selection import SelectKBest, f_classif, mutual_info_classif, SelectFromModel, RFECV

make_pipeline(StandardScaler(), SelectKBest(f_classif, k=50), LogisticRegression(max_iter=2000))
SelectFromModel(LogisticRegression(l1_ratio=1, solver="liblinear", C=0.1))   # L1 で係数 0 の特徴を落とす
RFECV(LogisticRegression(max_iter=2000), step=0.1, cv=cv, scoring="roc_auc", n_jobs=n_cpu)
```

特徴量選択は必ず Pipeline の中に入れる（全データで選んでから CV するとリークする。2.1 参照）。

### 1.10 クラスタリングの評価

| 指標 | 正解ラベル | 範囲 | 良い方向 |
|:---|:---|:---|:---|
| `silhouette_score(X, labels)` | 不要 | -1〜1 | 大 |
| `davies_bouldin_score(X, labels)` | 不要 | 0〜 | 小 |
| `calinski_harabasz_score(X, labels)` | 不要 | 0〜 | 大 |
| `adjusted_rand_score(y, labels)` | 必要 | -0.5〜1（偶然なら約0） | 大 |
| `normalized_mutual_info_score(y, labels)` | 必要 | 0〜1 | 大 |

```python
from sklearn.metrics import silhouette_score
silhouette_score(X, labels, sample_size=10_000, random_state=0)   # O(N^2) なので大きいデータはサンプリング
```

クラスタ番号とラベルの対応は任意なので、`accuracy_score(y, labels)` は使えない。

### 1.11 次元削減の詳細

```python
pca = PCA(n_components=50, whiten=True, random_state=0).fit(X_tr)   # whiten: 各成分の分散を1にする
np.cumsum(pca.explained_variance_ratio_)                            # 累積寄与率

from sklearn.decomposition import TruncatedSVD
TruncatedSVD(n_components=100, random_state=0).fit_transform(X_sparse)   # 疎行列向け（中心化しない）

from sklearn.manifold import TSNE
Z = TSNE(n_components=2, perplexity=30, init="pca", learning_rate="auto",
         max_iter=1000, random_state=0, n_jobs=n_cpu).fit_transform(X_pca50)
```

| 注意点 | 内容 |
|:---|:---|
| t-SNE の入力 | 高次元のままより、PCA で 50 次元程度に落としてからが速い |
| t-SNE の `perplexity` | 近傍の広さ。サンプル数より小さくする必要がある |
| t-SNE は `transform` が無い | 新しいデータを同じ空間に写せない。距離やクラスタの大きさは解釈しない |
| `max_iter` | 1.5 で `n_iter` から改名 |
| PCA の `whiten=True` | 小さい成分のノイズも増幅される。距離ベースの後段モデルでだけ使う |

### 1.12 外れ値検出のスコアの向き

| モデル | `predict` | `score_samples` / `decision_function` |
|:---|:---|:---|
| `IsolationForest` | 正常 1、外れ値 -1 | 大きいほど正常。`decision_function = score_samples - offset_` で、負なら外れ値 |
| `OneClassSVM` | 同上 | 同上 |
| `LocalOutlierFactor(novelty=True)` | 同上 | 同上。`novelty=False` だと新しいデータに `predict` できず、`negative_outlier_factor_` を使う |

異常を陽性（1）として AUC を計算するときは符号を反転する。

```python
anomaly = -iso.score_samples(X_te)
roc_auc_score(y_te_is_anomaly, anomaly)
```

### 1.13 AUC のブートストラップ信頼区間

```python
def bootstrap_auc(y, p, groups=None, n_boot=2000, seed=0):
    rng = np.random.default_rng(seed)
    y, p = np.asarray(y), np.asarray(p)
    if groups is None:
        units = [np.array([i]) for i in range(len(y))]
    else:                                              # 動物単位で再標本化する
        g = np.asarray(groups)
        units = [np.flatnonzero(g == u) for u in np.unique(g)]
    aucs = []
    for _ in range(n_boot):
        pick = rng.integers(0, len(units), len(units))
        idx = np.concatenate([units[k] for k in pick])
        if len(np.unique(y[idx])) < 2:                # 片方のクラスしかない標本は捨てる
            continue
        aucs.append(roc_auc_score(y[idx], p[idx]))
    return np.percentile(aucs, [2.5, 97.5]), len(aucs)
```

パッチ単位のデータで行単位のブートストラップをすると、区間が狭く出すぎる。評価の単位（動物・スライド）で再標本化する。

### 1.14 Slurm 上での並列化

scikit-learn の並列は2層ある: `n_jobs`（joblib のプロセス・スレッド）と、NumPy/SciPy の BLAS・OpenMP のスレッド。

```python
import os
from joblib import parallel_config
from threadpoolctl import threadpool_limits

n_cpu = int(os.environ.get("SLURM_CPUS_PER_TASK", 1))

# 探索を並列化するなら、各ワーカーの BLAS は 1 スレッドに
with threadpool_limits(limits=1):
    GridSearchCV(pipe, grid, cv=cv, n_jobs=n_cpu).fit(X, y)

with parallel_config(backend="loky", n_jobs=n_cpu):   # n_jobs を指定しない推定器にも既定値を与える
    ...
```

- joblib の loky ワーカーは、BLAS のスレッド数を「CPU 数 / ワーカー数」に自動で制限する。ただし joblib が数える CPU 数が Slurm の割り当てと一致するとは限らないので、`n_jobs` は明示する。
- `HistGradientBoosting*` や `KMeans` は `n_jobs` を持たず OpenMP で並列化する。スレッド数は `OMP_NUM_THREADS` で決まる。

### 1.15 モデルの保存

```python
import joblib, sklearn
joblib.dump({"model": pipe, "sklearn": sklearn.__version__, "features": list(X.columns)}, "model.joblib")
obj = joblib.load("model.joblib")
```

- バージョンが違うと `InconsistentVersionWarning` が出る。予測が正しい保証は無いので、同じバージョンで使うか再学習する。
- pickle / joblib は読み込み時に任意のコードを実行できる。他人から受け取ったファイルは開かない。安全な形式が必要なら skops（別パッケージ）を検討する。
- 長期保存するなら、モデルと一緒に学習コードのコミット、`uv.lock`、学習データの識別子を残す。

### 1.16 1.8 での変更点（このマシンで確認）

| 変更 | 対処 |
|:---|:---|
| `LogisticRegression(penalty=...)` が非推奨（1.10 で削除） | `l1_ratio=0`（L2、既定）/ `l1_ratio=1`（L1）/ `0<l1_ratio<1`（elastic net）、`C=np.inf`（正則化なし）で指定する |
| `LogisticRegression(n_jobs=...)` が非推奨 | 外側の CV・探索で並列化する |
| `LogisticRegression(multi_class=...)` は削除済み | 多クラスは常に multinomial。OvR が必要なら `OneVsRestClassifier` |
| `CalibratedClassifierCV(cv="prefit")` は削除済み | `FrozenEstimator` で包む（1.5） |
| 1クラスしかない `roc_auc_score` が nan を返す | 例外ではなく `UndefinedMetricWarning` + nan（2.5） |

## Part 2. トラブル対応

### 2.1 CV の性能が高すぎる（前処理・特徴量選択のリーク）

- **症状**: CV の AUC が非常に高いのに、外部データや新しい試験で大きく下がる。
- **確認**:
```bash
grep -nE "fit_transform\(X\b|\.fit\(X\b" analysis.py     # CV の前に全データで fit している行を探す
```
```python
# ラベルをシャッフルしても性能が出るなら、どこかでリークしている
from sklearn.model_selection import permutation_test_score
score, perm_scores, pval = permutation_test_score(pipe, X, y, groups=animal_ids, cv=cv,
                                                  scoring="roc_auc", n_permutations=100)
print(score, perm_scores.mean(), pval)                           # perm_scores.mean() が 0.5 付近であるべき
```
- **原因**: 標準化・PCA・欠損補完・特徴量選択・オーバーサンプリングを全データで行ってから CV している。特に「全データで有意な特徴を選んでから CV」は、ランダムなデータでも高い性能が出る。
- **対処**: 学習データから統計量を得る処理はすべて Pipeline に入れ、Pipeline ごと CV に渡す。

### 2.2 グループのリーク（同じ動物が train と test に入る）

- **症状**: パッチ単位・スライド単位で分割したときだけ性能が高い。
- **確認**:
```python
for k, (tr, te) in enumerate(cv.split(X, y, groups=animal_ids)):
    overlap = set(animal_ids[tr]) & set(animal_ids[te])
    print(k, len(overlap), np.bincount(y[te], minlength=n_classes))   # 重複 0 か、各 fold のクラス数
```
- **原因**: `StratifiedKFold` / `train_test_split` などグループを考慮しない分割を使った、または `groups` を渡し忘れた（`GroupKFold` 以外の分割器は `groups` を黙って無視する）。
- **対処**: `GroupKFold` / `StratifiedGroupKFold` を使い、`groups` を必ず渡す。入れ子 CV の場合は 1.3 のとおりメタデータのルーティングを有効にする。

### 2.3 ConvergenceWarning

- **症状**: `ConvergenceWarning: lbfgs failed to converge after 100 iteration(s)`。
- **確認**:
```python
print(X.mean(0)[:5], X.std(0)[:5])                  # スケールが揃っているか
m = pipe[-1]; print(m.n_iter_, m.max_iter)          # 上限に達したか
print(np.abs(m.coef_).max())                        # 係数が巨大なら分離可能データ・正則化不足
```
- **原因**: 特徴量のスケールが揃っていない、`max_iter` が不足、正則化が弱すぎる（`C` が大きい）、クラスが完全に分離できる。
- **対処**: `StandardScaler` を前に入れる → `max_iter=2000〜5000` → `C` を小さくする、の順に試す。警告を消すだけの `warnings.filterwarnings` は、原因を確認してからにする。

### 2.4 `predict_proba` が無い

- **症状**: `AttributeError: This 'SVC' has no attribute 'predict_proba'`、`LinearSVC` や `SGDClassifier(loss="hinge")` で同様。`roc_auc` のスコアラーは動くのに自分のコードで失敗する。
- **確認**:
```python
print(hasattr(clf, "predict_proba"), hasattr(clf, "decision_function"))
print(clf.get_params().get("probability"), clf.get_params().get("loss"))
```
- **原因**: 確率を出さないモデル・設定。`SVC` は `probability=False` が既定。
- **対処**: AUC や順位の評価だけなら `decision_function` のスコアで十分（`roc_auc_score` にそのまま渡せる）。確率が必要なら `CalibratedClassifierCV(LinearSVC())` で包む。`SVC(probability=True)` は内部で 5-fold CV を行うので遅く、`predict` と `predict_proba` の結果が一致しないことがある。

### 2.5 fold の AUC が nan・多クラス AUC のエラー

- **症状**: `cross_validate` の `test_score` に nan が混ざる。`UndefinedMetricWarning: Only one class is present in y_true. ROC AUC score is not defined in that case.`（1.8 では例外ではなく nan）。多クラスでは `Number of classes in y_true not equal to the number of columns in 'y_score'`。
- **確認**:
```python
for tr, te in cv.split(X, y, groups=animal_ids):
    print(np.unique(y[te], return_counts=True))
```
- **原因**: グループ分割でクラスの偏りが大きく、ある fold の test に1クラスしか入らない。多クラスでは test に出現しないクラスがあり、確率の列数と合わない。
- **対処**: `StratifiedGroupKFold` を使う、fold 数を減らす、`LeaveOneGroupOut` なら fold ごとの AUC ではなく全 fold の予測をまとめて（`cross_val_predict`）評価する。多クラスは `roc_auc_score(..., labels=clf.classes_)` を指定する。平均するときは `np.nanmean` ではなく、nan の fold 数も報告する。

### 2.6 特徴量名の警告

- **症状**: `UserWarning: X does not have valid feature names, but LogisticRegression was fitted with feature names`、またはその逆 `X has feature names, but ... was fitted without feature names`。`The feature names should match those that were passed during fit` で止まる。
- **確認**:
```python
print(getattr(model, "feature_names_in_", None))
print(type(X_new), getattr(X_new, "columns", None))
set(model.feature_names_in_) ^ set(X_new.columns)      # 名前の差分
```
- **原因**: fit は DataFrame、predict は NumPy 配列（またはその逆）。列の順序や名前が学習時と違う。
- **対処**: fit と predict で同じ型に揃える。DataFrame なら `X_new[model.feature_names_in_]` で列順を合わせる。列の順序違いは警告なしで誤った予測になる場合があるので、名前付きで渡すほうが安全。

### 2.7 UndefinedMetricWarning（precision / recall / F1）

- **症状**: `UndefinedMetricWarning: Precision is ill-defined and being set to 0.0 ... Use zero_division parameter to control this behavior.`（Recall / F-score でも同様）
- **確認**:
```python
print(np.unique(pred, return_counts=True), np.unique(y_te, return_counts=True))
print(confusion_matrix(y_te, pred))
```
- **原因**: あるクラスを一度も予測していない（precision の分母が 0）、または test にそのクラスが無い（recall の分母が 0）。クラス不均衡で少数クラスを全く予測しないモデルが典型。
- **対処**: 指標の問題ではなくモデルの問題であることが多い。`class_weight="balanced"`、閾値調整（1.6）を検討する。集計の定義を固定したい場合は `zero_division=0`（または `np.nan`）を明示する。

### 2.8 GridSearch が終わらない・メモリが溢れる

- **症状**: 探索が何時間も終わらない、`n_jobs` を増やしたらメモリ不足で落ちた。
- **確認**:
```python
from sklearn.model_selection import ParameterGrid
print(len(ParameterGrid(grid)), "候補 ×", cv.get_n_splits(), "fold")
print(X.nbytes / 1e9, "GB ×", n_jobs, "ワーカー")          # 各ワーカーにデータがコピーされうる
```
- **原因**: 候補数 × fold 数の学習が必要。`n_jobs` のワーカーごとにデータと中間結果を持つ。`SVC` や `KNN` はサンプル数に対して計算量が急増する。
- **対処**:
  - 小さいサブセットで1回の fit 時間を測ってから全体を見積もる
  - `RandomizedSearchCV` / `HalvingRandomSearchCV` に切り替える
  - 重い前処理は `Pipeline(memory=...)` でキャッシュする、または探索の外で一度だけ計算できるもの（学習を伴わない変換）は先に済ませる
  - `n_jobs` を減らす、`pre_dispatch="n_jobs"` で同時に投入するジョブを抑える
  - float32 に変換してから渡す（多くのモデルで float32 のまま計算できる）

### 2.9 実行ごとに結果が違う

- **症状**: 同じコードなのに CV スコアや選ばれたパラメータが実行ごとに変わる。
- **確認**:
```python
print({k: v for k, v in pipe.get_params().items() if k.endswith("random_state")})
print(cv)                                            # shuffle=True なのに random_state=None になっていないか
```
- **原因**: `random_state` を指定していない推定器・分割器がある（`shuffle=True` の KFold、RandomForest、`HistGradientBoosting` の早期終了用の検証分割、`PCA(svd_solver="randomized")`、`KMeans` の初期値、t-SNE）。並列実行の順序や BLAS による浮動小数点の差もある。
- **対処**: すべての `random_state` を整数で固定する。報告では複数シードで回した平均と標準偏差を示す（1つのシードの結果に依存しない）。

### 2.10 保存したモデルが読めない・警告が出る

- **症状**: `InconsistentVersionWarning: Trying to unpickle estimator ... from version 1.5.2 when using version 1.8.0`、`AttributeError: Can't get attribute ...`、`ModuleNotFoundError`。
- **確認**:
```python
import sklearn, warnings
from sklearn.exceptions import InconsistentVersionWarning
warnings.simplefilter("error", InconsistentVersionWarning)
try:
    joblib.load("model.joblib")
except InconsistentVersionWarning as w:
    print(w.original_sklearn_version, sklearn.__version__)
```
- **原因**: 保存時と読み込み時で scikit-learn（または NumPy）のバージョンが違う。自作クラスを含むモデルで、読み込み側からそのクラスを import できない。
- **対処**: 保存時と同じバージョンの環境で読む（`uv.lock` を残しておく）。自作クラスは `__main__` ではなくモジュールに置いて import できるようにする。バージョンを上げる必要があるなら再学習する。

### 2.11 n_jobs で CPU を取り合う・かえって遅い

- **症状**: `n_jobs=-1` にしたらノードの他のジョブまで遅くなった、`n_jobs` を増やしても速くならない、ジョブが CPU 時間超過で止められる。
- **確認**:
```python
import os, joblib
from threadpoolctl import threadpool_info
print(os.environ.get("SLURM_CPUS_PER_TASK"), os.cpu_count(), joblib.cpu_count())
print([(i["internal_api"], i["num_threads"]) for i in threadpool_info()])
```
- **原因**: `n_jobs=-1` は見える全 CPU を使おうとする。さらにワーカーごとに BLAS / OpenMP のスレッドが立ち、ワーカー数 × スレッド数に膨らむ。探索の `n_jobs` と推定器の `n_jobs`（RandomForest など）を両方指定すると入れ子になる。
- **対処**: `n_jobs=int(os.environ["SLURM_CPUS_PER_TASK"])` を明示する。並列は1層だけにする（探索で並列化するなら推定器側は `n_jobs=1`）。sbatch で `OMP_NUM_THREADS` などを設定する（[numpy_deep.md](numpy_deep.md) の 1.12）。

### 2.12 1.8 で LogisticRegression の FutureWarning

- **症状**: `FutureWarning: 'penalty' was deprecated in version 1.8 and will be removed in 1.10.`、`'l1_ratio=None' was deprecated ...`。
- **確認**:
```python
import warnings
warnings.simplefilter("error", FutureWarning)       # 警告箇所で止める
```
```bash
grep -rn "penalty=" src/ scripts/
```
- **原因**: 1.8 で正則化の種類は `l1_ratio`、有無は `C` で指定する方式に変わった。
- **対処**: `penalty="l2"` は削除（既定の `l1_ratio=0` と同じ）、`penalty="l1"` → `l1_ratio=1`、`penalty="elasticnet", l1_ratio=r` → `l1_ratio=r`、`penalty=None` → `C=np.inf`。solver との組み合わせ制約（L1 は `liblinear` / `saga`）は従来どおり。
