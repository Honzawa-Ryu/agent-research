# 収集論文・リソース一覧 (Curated Papers & Resources)

本調査（対照群Normativeモデルによる反事実的異常検知の生成AIアーキテクチャ深掘り）で精読・参照した主要論文および関連リソースの書誌情報と要約です。

---

## 📚 論文メタデータ一覧

| No. | タイトル | 著者 / 年 / 会議・誌 | リンク (arXiv/DOI/PMC) | ローカルPDF | 重要度 |
|:---:|:---|:---|:---:|:---:|:---:|
| 1 | [Diffusion Models for Medical Anomaly Detection](#paper-1) | Wolleb, Bieder, Sandkühler, Cattin (2022, MICCAI) | [arXiv:2203.04306](https://arxiv.org/abs/2203.04306) | [PDF](pdfs/2022_Wolleb_DiffusionMedicalAnomalyDetection.pdf) | ★★★ |
| 2 | [What is Healthy? Generative Counterfactual Diffusion for Lesion Localization](#paper-2) | Sanchez, Kascenas, Liu, O'Neil, Tsaftaris (2022, DGM4MICCAI Workshop) | [arXiv:2207.12268](https://arxiv.org/abs/2207.12268) | [PDF](pdfs/2022_Sanchez_WhatIsHealthyCounterfactualDiffusion.pdf) | ★★★ |
| 3 | [AnoDDPM: Anomaly Detection With Denoising Diffusion Probabilistic Models Using Simplex Noise](#paper-3) | Wyatt, Leach, Schmon, Willcocks (2022, CVPR Workshop) | [CVF OpenAccess](https://openaccess.thecvf.com/content/CVPR2022W/NTIRE/html/Wyatt_AnoDDPM_Anomaly_Detection_With_Denoising_Diffusion_Probabilistic_Models_Using_Simplex_CVPRW_2022_paper.html) | [PDF](pdfs/2022_Wyatt_AnoDDPM.pdf) | ★★☆ |
| 4 | [Counterfactual Diffusion Models for Interpretable Explanations of AI Models in Pathology (MoPaDi)](#paper-4) | Žigutytė, Lenz, Han, Hewitt, Reitsam, Foersch, Kather, et al. (2024–2025, bioRxiv) | [bioRxiv:10.1101/2024.10.29.620913](https://www.biorxiv.org/content/10.1101/2024.10.29.620913) | - | ★★★ |
| 5 | [Evaluating Normative Representation Learning in Generative AI for Robust Anomaly Detection in Brain Imaging](#paper-5) | Bercea, Wiestler, Rueckert, Schnabel (2025, Nature Communications) | [DOI:10.1038/s41467-025-56321-y](https://doi.org/10.1038/s41467-025-56321-y) / [PMC11825664](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11825664/) | - | ★★★ |
| 6 | [Counterfactual MRI Generation with Denoising Diffusion Models for Interpretable Alzheimer's Disease Effect Detection](#paper-6) | Dhinagar, Thomopoulos, Laltoo, Thompson (2024, bioRxiv / IEEE EMBC) | [bioRxiv:10.1101/2024.02.05.578983](https://www.biorxiv.org/content/10.1101/2024.02.05.578983) | - | ★★☆ |
| 7 | [Diffusion-based Generation of Histopathological Whole Slide Images at a Gigapixel Scale](#paper-7) | Harb, Pock, Müller (2023, WACV 2024) | [arXiv:2311.08199](https://arxiv.org/abs/2311.08199) | [PDF](pdfs/2023_Harb_GigapixelWSIDiffusion.pdf) | ★★★ |
| 8 | [ZoomLDM: Latent Diffusion Model for Multi-scale Image Generation](#paper-8) | Yellapragada, et al. (2024, CVPR 2025) | [arXiv:2411.16969](https://arxiv.org/abs/2411.16969) | [PDF](pdfs/2024_Yellapragada_ZoomLDM.pdf) | ★★☆ |
| 9 | [A Survey on Diffusion Models for Anomaly Detection](#paper-9) | Liu, Ma, Wang, Zou, Ren, Wang, Song, Hu, Liu, Leung (2025, arXiv) | [arXiv:2501.11430](https://arxiv.org/abs/2501.11430) | [PDF](pdfs/2025_Liu_DiffusionAnomalyDetectionSurvey.pdf) | ★★☆ |
| 10 | [Comparative Analysis of Chronic Progressive Nephropathy (CPN) Diagnosis in Rat Kidneys Using an AI Deep Learning Model](#paper-10) | (2024, Toxicological Research) | [DOI:10.1007/s43188-024-00247-y](https://doi.org/10.1007/s43188-024-00247-y) / [PubMed:39345736](https://pubmed.ncbi.nlm.nih.gov/39345736/) | - | ★★☆ |

---

## 📝 論文詳細サマリー

### <a id="paper-1"></a> [1] Diffusion Models for Medical Anomaly Detection
- **著者**: Julia Wolleb, Florentin Bieder, Robin Sandkühler, Philippe C. Cattin（バーゼル大学）
- **掲載**: MICCAI 2022 / arXiv:2203.04306

#### 概要 (Abstract TL;DR)
弱教師あり（画像レベルラベルのみ）で異常検知を行う手法。Denoising Diffusion Implicit Models (DDIM) の決定論的なノイズ付加・除去過程に**分類器ガイダンス**を組み合わせ、「病変あり画像」→「健常画像」への image-to-image translation を実現。差分画像から詳細な異常マップを複雑な訓練手順なしに生成できる。BRATS2020（脳腫瘍）・CheXpert（胸水）で検証。

#### 毒性病理への示唆
本分野で最も引用される反事実拡散異常検知の基礎手法（2024年時点でMICCAI論文の約20%が本手法を利用と報告される）。「対照群＝健常ドメイン」「投与群＝病変ドメイン」と読み替えれば、PROP-04が目指す枠組みの直接的な出発点になる。ただし脳MRI（256×256程度）が前提で、WSIのギガピクセルスケールへの適用は本論文の範囲外。

---

### <a id="paper-2"></a> [2] What is Healthy? Generative Counterfactual Diffusion for Lesion Localization
- **著者**: Pedro Sanchez, Antanas Kascenas, Xiao Liu, Alison Q. O'Neil, Sotirios A. Tsaftaris（エディンバラ大学）
- **掲載**: Deep Generative Models Workshop @ MICCAI 2022 / arXiv:2207.12268
- **コード**: [github.com/vios-s/Diff-SCM](https://github.com/vios-s/Diff-SCM)

#### 概要 (Abstract TL;DR)
「この患者に特定の病理がなかったら、どう見えるか」という反事実を拡散確率モデル（DPM）で合成し、観測画像との差分から病変位置を推論する弱教師ありセグメンテーション手法。分類器ガイダンスの代わりに、暗黙的ガイダンスとアテンション条件付けを用いて改善。

#### 毒性病理への示唆
「Counterfactual Synthesis」という用語・問題設定そのものが、PROP-04が構想する「被験物質を投与されていなかったらどう見えるか」を最も直接的に先取りしている先行研究。ただし単一患者内の反事実（構造的因果モデル: SCM）であり、毒性試験のような**群レベルの対照群データを条件として使う**設計ではない点が拡張の焦点になる。

---

### <a id="paper-3"></a> [3] AnoDDPM: Anomaly Detection With Denoising Diffusion Probabilistic Models Using Simplex Noise
- **著者**: Julian Wyatt, Adam Leach, Sebastian M. Schmon, Chris G. Willcocks（ダラム大学）
- **掲載**: CVPR Workshop (NTIRE) 2022

#### 概要 (Abstract TL;DR)
標準的なガウスノイズの代わりに**マルチスケールSimplexノイズ**を用いる部分拡散（Partial Diffusion）戦略により、高解像度画像へのスケーラビリティと、検出したい異常サイズの制御を両立。DDPMはGANよりモード網羅性が高く、VAEよりサンプル品質が高いという特性を異常検知に活用。

#### 毒性病理への示唆
「異常サイズを制御できるノイズ設計」は、毒性病理の病変が微細（単細胞壊死）からびまん性（広範な空胞変性）まで幅広いスケールを持つ点と相性が良く、WSIパッチスケールでの反事実生成における画像拡散スケジュール設計の参考になる。

---

### <a id="paper-4"></a> [4] Counterfactual Diffusion Models for Interpretable Explanations of AI Models in Pathology（MoPaDi）
- **著者**: Laura Žigutytė, Tim Lenz, Tianyu Han, Katherine Jane Hewitt, Nic Gabriel Reitsam, Sebastian Foersch, Jakob Nikolas Kather, et al.（KatherLab / RWTHアーヘン工科大学ほか）
- **掲載**: bioRxiv preprint, 2024-10-29投稿・2025年に複数回改訂 (v2〜v4) / DOI: 10.1101/2024.10.29.620913
- **コード/モデル**: [github.com/KatherLab/mopadi](https://github.com/KatherLab/mopadi) / [HuggingFace: KatherLab/MoPaDi](https://huggingface.co/KatherLab/MoPaDi)

> ⚠️ 本文全文はbioRxiv側のアクセス制限（403）により未読了。GitHub README・関連検索結果からの要約であり、手法詳細は一部推定を含む。

#### 概要 (Abstract TL;DR)
**Diffusion Autoencoder**とタスク特化MIL分類器を組み合わせ、分類器の予測を反転させる方向に潜在表現を操作することで、「どの形態学的特徴が予測を駆動しているか」を明らかにする反事実的説明生成フレームワーク（MOrphing histoPAthology DIffusion）。大腸・乳腺・肝臓・肺のがん種、組織型分類、施設由来判別、MSI（マイクロサテライト不安定性）バイオマーカーなど複数タスクで検証。TCGA、NCT-CRC-HE-100Kデータセットを使用。評価はSSIM/MS-SSIM/MSEに加え病理医によるユーザースタディ。

#### 毒性病理への示唆
**本調査における最重要発見の一つ**: 反事実拡散モデルを実際の病理WSI（パッチ）に適用した、本調査で発見できた唯一の実装例。ただし対象は疾患病理（がん）であり、「同一条件の対照群」という概念は存在せず、個々の画像を潜在空間内で操作する反事実（"この画像がMSI-highでなかったら"）である点が、PROP-04が目指す「群レベルの対照群分布を正規モデルとして学習する」設計とは異なる。Diffusion Autoencoderのアーキテクチャ自体は、対照群データで学習させることで毒性病理向けに転用しうる有力な技術的土台。

---

### <a id="paper-5"></a> [5] Evaluating Normative Representation Learning in Generative AI for Robust Anomaly Detection in Brain Imaging
- **著者**: Cosmin I. Bercea, Benedikt Wiestler, Daniel Rueckert, Julia A. Schnabel（ミュンヘン工科大学ほか）
- **掲載**: Nature Communications (2025-02-13) / DOI:10.1038/s41467-025-56321-y
- **コード**: [github.com/compai-lab/2024-ncomms-bercea](https://github.com/compai-lab/2024-ncomms-bercea)

#### 概要 (Abstract TL;DR)
「正規表現学習（Normative Representation Learning）」— 健常集団の典型的な解剖学的分布を大規模データから学習する能力 — を評価するための新規指標を提案し、拡散モデルを含む複数の生成AIフレームワークを多様な脳病理に対して比較。専門医との比較を含む大規模マルチリーダー研究を実施し、正規学習に優れたモデルほど未知の病態を汎用的に検出できることを実証。

#### 毒性病理への示唆
「正規モデルの学習品質をどう定量評価するか」という方法論は、PROP-04で構築する対照群Normative Modelの妥当性検証（背景病変を正しく"正常"として再構成できているか）に直接応用できる評価フレームワーク。毒性病理版の正規モデルを構築した際、本論文の評価指標を転用した検証設計が可能。

---

### <a id="paper-6"></a> [6] Counterfactual MRI Generation with Denoising Diffusion Models for Interpretable Alzheimer's Disease Effect Detection
- **著者**: Nikhil J. Dhinagar, Sophia I. Thomopoulos, Emily Laltoo, Paul M. Thompson（南カリフォルニア大学）
- **掲載**: bioRxiv (2024-02-05) / IEEE EMBC 2024

#### 概要 (Abstract TL;DR)
臨床診断ラベル（アルツハイマー病 vs 健常）を条件変数とした条件付き潜在拡散モデル(LDM)・DDPMを3D T1強調MRIで学習し、個体レベルでのアルツハイマー病の脳への効果を可視化する反事実MRIを生成。

#### 毒性病理への示唆
「診断ラベルという離散カテゴリ変数を条件として反事実生成する」設計は、PROP-04のコア問い2（動物種・系統別の自然発生病変プロファイルを条件付けとして組み込む）に対する最も直接的な先行実装パターン。系統（例: Fischer 344 vs Sprague-Dawley）や用量群を診断ラベルの代わりに条件変数として扱う拡張が考えられる。

---

### <a id="paper-7"></a> [7] Diffusion-based Generation of Histopathological Whole Slide Images at a Gigapixel Scale
- **著者**: Robert Harb, Thomas Pock, Heimo Müller（グラーツ工科大学）
- **掲載**: WACV 2024 / arXiv:2311.08199

#### 概要 (Abstract TL;DR)
既存の拡散生成モデルはWSIの高解像度化に伴う計算複雑性のため、ギガピクセル規模の生成ができていなかった課題に対し、**coarse-to-fineサンプリング**（低解像度から段階的に解像度を上げつつ細部を追加）を提案。定量評価と病理医によるユーザースタディにより、生成WSIが実WSIの構造的特徴を再現していることを確認。

#### 毒性病理への示唆
PROP-04のコア問い1（脳MRI等の反事実異常検知手法をWSIパッチのような超高解像度画像にどう拡張するか）に対する直接的な技術的解決策の一つ。ただし本論文自体は「生成」タスクの実証であり、「反事実（対照群への変換）」や「異常検知への応用」は行っていない — 両者を統合する研究は本調査では未発見。

---

### <a id="paper-8"></a> [8] ZoomLDM: Latent Diffusion Model for Multi-scale Image Generation
- **著者**: Srikar Yellapragada, et al.
- **掲載**: CVPR 2025 / arXiv:2411.16969

#### 概要 (Abstract TL;DR)
自己教師あり学習(SSL)埋め込みを用いた「倍率認識（magnification-aware）」条件付け機構により、複数スケール（20倍対物相当のパッチから縮小サムネイルまで）で一貫性のある画像を生成する潜在拡散モデル。4096×4096画像を約8分で生成（従来手法は4〜12時間）と、大幅な計算効率化を達成。

#### 毒性病理への示唆
Harb et al. (2023)と並び、WSIスケールでの生成コストを実用域に近づける技術。反事実生成をWSI全体（あるいは臓器単位のROI）に適用する場合の計算コスト見積もりの参考値として重要（1パッチあたり数分〜十数分オーダーが現実的な下限）。

---

### <a id="paper-9"></a> [9] A Survey on Diffusion Models for Anomaly Detection
- **著者**: Jing Liu, Zhenchao Ma, Zepu Wang, Chenxuanyin Zou, Jiayang Ren, Zehua Wang, Liang Song, Bo Hu, Yang Liu, Victor C. M. Leung
- **掲載**: arXiv preprint (2025-01-20) / arXiv:2501.11430

#### 概要 (Abstract TL;DR)
拡散モデルによる異常検知手法を「再構成ベース (reconstruction-based)」「密度ベース (density-based)」「ハイブリッド」の3系統に分類し、サイバーセキュリティ・不正検知・ヘルスケア・製造業等、幅広い応用分野を横断的に整理したサーベイ。

#### 毒性病理への示唆
本調査で収集した個別論文（Wolleb, Sanchez, Wyatt等はいずれも再構成ベース）を体系の中に位置づける際の参照分類軸として活用。毒性病理での反事実異常検知を設計する際、「密度ベース（尤度・スコアで異常度を測る）」アプローチが未検討の選択肢として残っていることも示唆。

---

### <a id="paper-10"></a> [10] Comparative Analysis of Chronic Progressive Nephropathy (CPN) Diagnosis in Rat Kidneys Using an AI Deep Learning Model
- **掲載**: Toxicological Research (2024) / DOI:10.1007/s43188-024-00247-y

#### 概要 (Abstract TL;DR)
ラット腎臓の慢性進行性腎症（CPN: 加齢性自然発生背景病変の代表例）について、33枚のWSIから抽出した約2,000枚の病変画像を用い、YOLOv8・Mask R-CNN・SOLOv2の3種のセグメンテーションモデルで、尿細管好塩基性変化を伴う萎縮・単核細胞浸潤・硝子円柱の3病変を検出・比較。Mask R-CNNが全病変でmAP50最高、YOLOv8とSOLOv2はセグメンテーション精度が低いと報告。

#### 毒性病理への示唆
毒性病理の代表的自然発生背景病変（CPN）に対する現行AIの到達点は、**検出・分割（判別モデル）に留まり、生成・反事実アプローチは本調査時点で皆無**であることを示す直接的な比較対象。PROP-04が構想する「背景病変の正規分布モデル」を構築する際、本研究のような病変アノテーション済みデータセットが学習・評価データの土台になり得る。
