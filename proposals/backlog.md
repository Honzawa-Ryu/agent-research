# 調査テーマ提案バックログ (Research Proposals Backlog)

過去の調査結果や最新の研究トレンドを踏まえて、Agentが提案した未着手の調査テーマ一覧です。  
ユーザーが承認（ステータスを「承認済み」に変更）したテーマから順次 `topics/` 配下で調査を開始します。

---

## 📋 提案一覧 (Active Proposals)

### [PROP-01] 動物種横断（Cross-Species）病理基盤モデルの構築手法と形態学的相同性アライメント技術の深掘り

- **提案日**: 2026-08-18
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
- **ステータス**: `調査完了` → [topics/2026/08/02_cross_species_pathology_fm/](topics/2026/08/02_cross_species_pathology_fm/report.md)
- **優先度**: `高`
- **タグ**: `#基盤モデル` `#CrossSpecies` `#ドメイン適応` `#毒性病理`

#### 提案の背景・動機
本調査（01_toxicology_vs_clinical_pathology）により、疾患病理で成功したUNI/Virchow等のヒト基盤モデルを動物種（ラット・マウス・イヌ・サル等）へ適応させる「動物種横断基盤モデル」が極めて有力なフロンティアであることが判明しました。生物種間の進化的保存形態と種特異的形態をどのように分離・アライメントするかの具体的な機械学習アーキテクチャを深掘り調査します。

#### 調査のコア問い（Research Questions）
1. ヒト基盤モデルの埋め込み表現は、げっ歯類組織に対してどのようなドメインシフトを起こすか？
2. 異種間の形態相同性を保持しつつ種特異的な構造を分離する損失関数（Contrastive alignment, Disentanglement）の最先端実装は何か？

#### 期待される調査アウトプット
- 比較生物学・病理学における計算科学的アプローチの最新文献レビュー
- 少量データでの動物種適応ファインチューニングのベストプラクティスまとめ

---

### [PROP-02] 毒性試験における用量反応性（Dose-Response）モデリングと階層的弱教師あり学習（Hierarchical Group-MIL）の数理・実装設計

- **提案日**: 2026-08-18
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
- **ステータス**: `調査完了` → [topics/2026/08/04_dose_response_hierarchical_mil/](topics/2026/08/04_dose_response_hierarchical_mil/report.md)
- **優先度**: `高`
- **タグ**: `#弱教師あり学習` `#MIL` `#用量反応性` `#DoseResponse` `#PfizerTRACE`

#### 提案の背景・動機
毒性病理の意思決定は個体単位ではなく「群間比較（Control vs Low/Mid/High Dose）」と「用量依存性」に基づきます。従来の1スライド単位MIL（CLAM, TransMIL等）を超え、試験群・個体・臓器・スライド・パッチの5階層構造をエンドツーエンドで最適化する最新の階層グラフ/アテンション手法および単調増加性正則化の数理モデルを調査します。

#### 調査のコア問い（Research Questions）
1. Pfizer TRACE等の最先端毒性病理モデルは、用量反応曲線をどのように深層学習内部でキャラクタライズしているか？
2. 毒性評価指標（NOAEL, LOAEL, BMD）の自動算出を可能にするMIL損失関数の設計手法とは？

#### 期待される調査アウトプット
- 階層的MILと用量反応曲線回帰の数理フレームワーク比較
- 孤発病変（偽陽性）を抑制する正則化アルゴリズムの整理

---

### [PROP-03] Open TG-GATEsを活用したトキシコゲノミクス×毒性病理WSIマルチモーダルAI（Patho-TGx）の先行研究とベンチマーク調査

- **提案日**: 2026-08-18
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
- **ステータス**: `調査完了` → [topics/2026/08/05_patho_toxicogenomics/](topics/2026/08/05_patho_toxicogenomics/report.md)
- **優先度**: `中`
- **タグ**: `#トキシコゲノミクス` `#OpenTGGATEs` `#マルチモーダル` `#VirtualPathology`

#### 提案の背景・動機
世界最大級の公開データセット「Open TG-GATEs」（170化合物、ラット肝・腎のWSI＋遺伝子発現データ）を活用した、組織形態から遺伝子発現を予測するモデルや、化合物構造式（SMILES）からの仮想病理組織像シミュレーション（Virtual Histopathology）の研究動向を調査します。

#### 調査のコア問い（Research Questions）
1. Open TG-GATEsを用いたこれまでの機械学習研究（PathologAI等）の到達点とデータセットの制約は何か？
2. 形態画像（HE染色）から毒性関連パスウェイ（CYP誘導、酸化ストレス等）を予測するマルチモーダル統合モデルの精度と課題は何か？

#### 期待される調査アウトプット
- Open TG-GATEsデータ構造・前処理パイプラインの解説
- 創薬初期スクリーニングにおけるバーチャル病理シミュレーションの実現可能性評価

---

### [PROP-04] 対照群Normativeモデルによる反事実的異常検知（Counterfactual Anomaly Detection）の生成AIアーキテクチャ深掘り

- **提案日**: 2026-08-18
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
- **ステータス**: `調査完了` → [topics/2026/08/03_counterfactual_anomaly_detection/](topics/2026/08/03_counterfactual_anomaly_detection/report.md)
- **優先度**: `高`
- **タグ**: `#異常検知` `#拡散モデル` `#反事実生成` `#背景病変`

#### 提案の背景・動機
本調査のフロンティア3で示した通り、毒性試験には「同一系統・同一飼育環境のControl群」が必ず並行して存在するという、疾患病理にはない構造的優位性があります。この対照群データを使い、加齢性・自然発生の背景病変（Spontaneous Lesions）と被験物質誘発病変を分離する「反事実生成（Counterfactual Synthesis）」の具体的な拡散モデル/VAEアーキテクチャと、病理領域での先行実装事例を深掘り調査します。

#### 調査のコア問い（Research Questions）
1. 医用画像分野（脳MRI等）で実用化されている反事実的異常検知（Counterfactual Diffusion, Cycle-GAN based normative modeling）の手法は、WSIパッチのような超高解像度・巨大画像にどう拡張可能か？
2. 動物種・系統別に異なる自然発生病変プロファイル（例：ラット系統ごとの慢性腎症発生率差）を条件付けとしてどう組み込むか？

#### 期待される調査アウトプット
- 反事実生成モデルのアーキテクチャ比較（拡散モデル vs GAN vs Normalizing Flow）
- WSIスケールへの計算コスト・実装上の課題整理

---

### [PROP-05] INHAND国際オントロジー準拠のVision-Language基盤モデル＆Concept Bottleneck Model設計の先行事例調査

- **提案日**: 2026-08-18
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
- **ステータス**: `調査完了` → [topics/2026/08/06_inhand_vlm_concept_bottleneck/](topics/2026/08/06_inhand_vlm_concept_bottleneck/report.md)
- **優先度**: `中`
- **タグ**: `#VisionLanguage` `#ConceptBottleneck` `#INHAND` `#説明可能AI` `#GLP規制`

#### 提案の背景・動機
フロンティア4で示した「INHAND用語体系準拠のTox-VLM / Concept Bottleneck Model」は、FDA/PMDA査察に耐えうる説明責任を提供する規制対応上の要衝です。疾患病理分野で先行するCONCH等のVision-Languageモデルや医療領域のConcept Bottleneck Modelの実装知見を、毒性病理の国際標準オントロジー（INHAND）にどう適用できるか調査します。

#### 調査のコア問い（Research Questions）
1. CONCH/PLIP等の病理VLMの対照学習レシピは、INHAND定義文のような構造化された鑑別診断基準テキストに対してどこまで有効か？
2. Concept Bottleneck Layerの「概念」をどう定義すれば、病理医の診断プロセス（核肥大・好酸性細胞質等の所見積み上げ）と整合するか？

#### 期待される調査アウトプット
- 医療VLM×Concept Bottleneck Modelの先行研究サーベイ
- INHAND用語体系のテキスト化・構造化データセット設計案

---

### [PROP-06] MELLODDY型・製薬企業間連合学習コンソーシアムの毒性病理領域への応用可能性と実装障壁の調査

- **提案日**: 2026-08-18
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
- **ステータス**: `調査完了` → [topics/2026/08/07_federated_learning_toxpath/](topics/2026/08/07_federated_learning_toxpath/report.md)
- **優先度**: `低`
- **タグ**: `#連合学習` `#MELLODDY` `#プライバシー保護` `#業界コンソーシアム`

#### 提案の背景・動機
フロンティア6の通り、1社単独ではカバーできない希少毒性所見（精巣毒性・神経毒性等）の学習データ不足は業界共通のボトルネックです。創薬化学領域で実績のあるMELLODDYコンソーシアムのアーキテクチャ・ガバナンスモデルを、毒性病理WSIという非構造化・大容量データに適用する際の技術的・組織的障壁を調査します。

#### 調査のコア問い（Research Questions）
1. MELLODDYで採用された準同型暗号・差分プライバシー・Secure Aggregationは、WSIパッチ特徴量の連合学習にそのまま転用可能か、計算コスト上の制約は何か？
2. スキャナ機種・染色プロトコルの企業間差異（バッチエフェクト）を連合学習の枠組み内でどう吸収するか？

#### 期待される調査アウトプット
- 連合学習フレームワーク（Flower, NVIDIA FLARE等）の病理WSI適用事例整理
- 参加企業間のデータガバナンス・契約上の論点整理

---

### [PROP-07] 動物毒性所見のヒト外挿性（Translational Toxicology）予測モデルの先行研究調査

- **提案日**: 2026-08-19
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)（7章 オープンクエスチョン1「種差外挿の生物学的限界」）
- **ステータス**: `調査完了` → [topics/2026/08/08_translational_toxicology_extrapolation/](topics/2026/08/08_translational_toxicology_extrapolation/report.md)
- **優先度**: `高`
- **タグ**: `#TranslationalToxicology` `#種差外挿` `#DILI` `#creative-outcome-prediction`

#### 提案の背景・動機
本調査のオープンクエスチョン1で指摘した通り、PROP-01（動物種横断基盤モデル）は「動物種間の形態を共通表現に落とし込む」技術に留まり、「その動物病理所見が実際にヒトのどの有害事象（DILI、心毒性、腎毒性等）に対応するか」という創薬意思決定に直結する外挿性予測は未着手です。DILIrank、ClinTox、FAERS等の既存ヒト毒性知識ベースと、動物WSI形態所見を紐付ける学習アーキテクチャを深掘りします。

#### 調査のコア問い（Research Questions）
1. 既存のTranslational Toxicology知識ベース（DILIrank, ClinTox, FAERS, ヒト臨床試験中止理由データベース等）と、動物病理WSI所見（肝細胞肥大・空胞変性等のINHAND所見）をどう対応付けて学習データセットを構築できるか？
2. 動物病理所見からヒト有害事象発生確率を予測するモデル（グラフニューラルネット、マルチモーダルLLM、知識グラフ埋め込み等）として何が有力か、また既存の定量的構造活性相関（QSAR）・PK/PDモデルとどう統合できるか？

#### 期待される調査アウトプット
- 動物→ヒト外挿性研究（Translational Safety Biomarker研究）の文献レビュー
- WSI形態情報を外挿性予測に組み込むためのデータ設計・PoC構想案

---

### [PROP-08] 光顕WSIからの微小構造推定（Virtual Ultrastructure / Sub-resolution Toxicity Inference）の実現可能性調査

- **提案日**: 2026-08-19
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)（7章 オープンクエスチョン2「微小毒性変化の検出限界」）
- **ステータス**: `調査完了` → [topics/2026/08/09_virtual_ultrastructure_inference/](topics/2026/08/09_virtual_ultrastructure_inference/report.md)
- **優先度**: `中`
- **タグ**: `#LabelFreePrediction` `#VirtualStaining` `#超解像` `#電子顕微鏡` `#ミトコンドリア毒性`

#### 提案の背景・動機
本調査のオープンクエスチョン2で指摘した通り、光顕HE染色WSI（20x/40x）の解像限界を超えるミトコンドリア変性やリソソーム蓄積などの微小毒性変化は、現状の全フロンティア（①〜⑥）のいずれでも扱われていません。細胞生物学分野で実績のあるLabel-free Prediction（In Silico Labeling等）やVirtual Staining（HE↔IHC変換）技術が、毒性病理の電顕相当所見の推定にどこまで転用可能かを調査します。

#### 調査のコア問い（Research Questions）
1. Label-free Prediction（Christiansen et al. In Silico Labeling等）やHE→蛍光/IHC変換のVirtual Staining技術は、HE-WSIから電顕（TEM）レベルの微細構造変化をどこまで推定可能か、既存の医用画像超解像研究との関係は？
2. 教師データとして必要なTEM画像とHE-WSIのペア収集・空間アライメント（同一組織ブロックからの連続切片対応付け等）はどう設計すべきか？

#### 期待される調査アウトプット
- Label-free Prediction / Virtual Staining / 医用超解像分野のサーベイ
- 毒性病理における微細構造推定PoCの実現可能性評価とデータ収集設計案

---

### [PROP-09] GLP規制環境下でのAIバリデーション・信頼性保証フレームワーク（Good Machine Learning Practice for GLP Toxicologic Pathology）調査

- **提案日**: 2026-08-19
- **関連する過去の調査**: [topics/2026/08/01_toxicology_vs_clinical_pathology/](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)（4.2節「GLPとバリデーションの厳格さ」、7章 オープンクエスチョン3「GLP規制当局とのコンセンサス形成」）
- **ステータス**: `提案中`
- **優先度**: `高`
- **タグ**: `#GLP` `#GoodMachineLearningPractice` `#FDA` `#PMDA` `#規制科学` `#21CFRPart11`

#### 提案の背景・動機
PROP-01〜08はいずれもモデル・アーキテクチャ面の技術調査ですが、本調査の4.2節・7章で指摘した通り、AIによる「陰性除外（Normal Triage）」を正式なGLP安全性試験プロセスとして規制当局に受容させるための客観的バリデーション基準は未確立のままです。技術が完成しても採用障壁を越えられなければ実用化しないため、規制科学（Regulatory Science）側からのギャップを独立テーマとして深掘りします。

#### 調査のコア問い（Research Questions）
1. FDA/PMDA/EMAが公表するAI/MLソフトウェア医療機器（SaMD）向けGood Machine Learning Practice（GMLP）ガイドラインや、Predetermined Change Control Plan等の枠組みは、GLP前臨床安全性試験のAIトリアージにどこまで転用・拡張可能か？
2. 21 CFR Part 11の電子記録・監査証跡要件を満たしつつ、AIの陰性除外判定の妥当性を客観的に検証するバリデーションプロトコル（感度/NPV基準、リファレンス病理医との一致率基準等）はどう設計すべきか？

#### 期待される調査アウトプット
- FDA/PMDA/EMA等のAI医療機器・GMLP関連ガイドライン整理
- GLP試験でのAIバリデーションプロトコル設計指針（受け入れ基準・監査証跡要件案）

---

## 🗄 運用ルール
- **提案時**: Agentは過去の調査レポート（`INDEX.md` や各 `report.md` の課題）を参考に新規テーマを提案し、本ファイルに追記する。
- **着手時**: ステータスを `調査中` に変更し、対応する `topics/YYYY/MM/NN_slug/` ディレクトリを作成して調査を進める。
- **完了時**: ステータスを `調査完了` に更新し、調査フォルダへのリンクを追記する。一定期間経過したものは `proposals/archive/` に退避可能。
