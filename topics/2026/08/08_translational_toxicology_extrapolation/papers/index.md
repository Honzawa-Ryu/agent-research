# 論文インデックス

[PROP-07] 動物毒性所見のヒト外挿性（Translational Toxicology）予測モデルの先行研究調査。論文11件。

---

### 1. Chen, M., Suzuki, A., Thakkar, S., Yu, K., Hu, C., Tong, W. (2016)
**DILIrank: the largest reference drug list ranked by the risk for developing drug-induced liver injury in humans.**
*Drug Discovery Today*, 21(4), 648–653. [DOI:10.1016/j.drudis.2016.02.015](https://doi.org/10.1016/j.drudis.2016.02.015)

FDA承認薬1,036件をヒトDILIリスクでランク付けしたベンチマークデータセットDILIrankの原著。後継のDILIrank 2.0は1,336件に拡張済み（2021年までの承認薬を追加）。**PROP-07の核心的な「ヒト側正解ラベル」となり得る資源。** 筆頭ラストオーサーのWeida TongはFDA/NCTRのAI4TOXプログラム（後述）の責任者でもあり、PathologAI（#4）の著者にも名を連ねる。

---

### 2. Park, M., Song, W., Ahn, H., Kim, S. (2025)
**Drug toxicity prediction based on genotype-phenotype differences between preclinical models and humans.**
*eBioMedicine* (published 2025-10-28). [https://www.thelancet.com/journals/ebiom/article/PIIS2352-3964(25)00438-4/fulltext](https://www.thelancet.com/journals/ebiom/article/PIIS2352-3964(25)00438-4/fulltext)（本文403のためEurekAlert!プレスリリース経由で確認）

POSTECH（浦項工科大学）Sanguk Kim研究室。遺伝子必須性（perturbationによる細胞生存への影響）・組織別発現パターン・ネットワーク結合性の3種の「遺伝子型-表現型差分（GPD）」特徴を種間で定量化し、これを毒性予測モデルに追加することでAUROC 0.50→0.75、AUPRC 0.35→0.63に改善。434件の有害薬・790件の承認薬で検証。1991年以前データのみで学習し1991年以降の市場撤退薬を95%の精度で的中。**入力は遺伝子発現ネットワーク特徴のみで、WSI形態情報は一切使用していない** — 本テーマのコア問いに最も近い定量結果を持つ論文だが、病理画像との統合は未着手であることを示す好例。

---

### 3. Puri, M. (2020)
**Automated Machine Learning Diagnostic Support System as a Computational Biomarker for Detecting Drug-Induced Liver Injury Patterns in Whole Slide Liver Pathology Images.**
*Assay and Drug Development Technologies*, published online 2020-01. [PubMed:31149832](https://pubmed.ncbi.nlm.nih.gov/31149832/)

ラット肝WSI 1,277枚（10薬剤×4用量）から、フラクタル次元・ラクナリティ特徴とAutoMLで壊死パターンを分類（平均適合率約98.6%）。**ラット組織内での壊死パターン分類に留まり、ヒトDILIアウトカムへの外挿は行っていない。** WSI形態情報を扱う数少ない毒性病理AI論文の一つとして、PROP-07が埋めるべきギャップ（WSI特徴→ヒト外挿）の「片側（動物側特徴抽出）」が既に存在することを示す。

---

### 4. Bussola, N., Xu, J., Wu, L., Gorini, L., Zhang, Y., Furlanello, C., Tong, W. (2023)
**A Weakly Supervised Deep Learning Framework for Whole Slide Classification to Facilitate Digital Pathology in Animal Study.**
*Chemical Research in Toxicology*, 36(8), 1321–1331. [DOI:10.1021/acs.chemrestox.3c00058](https://pubs.acs.org/doi/10.1021/acs.chemrestox.3c00058)

FDA/NCTR AI4TOXプログラムの「PathologAI」イニシアチブの中核論文。Open TG-GATEs由来のラット肝WSI 816枚（対照377枚含む）に対し、GANベースのタイル表現学習+5モデルアンサンブルCNNで壊死を弱教師あり分類。**著者にDILIrank原著者のWeida Tongが含まれる**（#1参照）。同一研究グループがヒトDILIデータベースと動物WSI分類モデルの両方を保有しながら、両者を接続する論文は本調査で確認できなかった——これがPROP-07の最も具体的で実行可能性の高い出発点となり得る。

---

### 5. FDA/NCTR (2024–2025)
**AI4TOX Program — AnimalGAN, SafetAI, PathologAI, BERTox, TranslAI Initiatives.**
[https://www.fda.gov/about-fda/nctr-research-focus-areas/artificial-intelligence](https://www.fda.gov/about-fda/nctr-research-focus-areas/artificial-intelligence)（公式FDAページ。TranslAI専用ページ・AI at NCTR個別ページは404で直接取得不可だったため、検索結果スニペットで内容を確認）

FDA国立毒性学研究センター（NCTR）の公式AI研究プログラム。5イニシアチブのうち**TranslAI**が「臓器システム・IVIVE・ゲノミクス技術を横断する実験知見の翻訳を促進する生成AIモデルの開発」を目的として明記されており、PROP-07と最も近い公式ミッションを持つ。ただし2026年8月時点で査読済み論文としての具体的アーキテクチャ・定量結果は本調査では確認できなかった（プログラム記述レベルの情報のみ）。

---

### 6. Xie, J., Liu, W., Hu, W., Ouyang, M., Huang, T. (2025)
**Graph Neural Network-Based Toxicity Prediction by Integrating Molecular Fingerprints and Knowledge Graph Features.**
*Toxics*. [PMC12656225](https://pmc.ncbi.nlm.nih.gov/articles/PMC12656225/)

ComptoxAI/PubChem/Reactome/ChEMBL由来の毒性知識グラフ(ToxKG)と分子フィンガープリント(MACCS, FP2, Morgan, Atom-Pair, ECFP4)を統合したヘテロ型GNN(GPS等)。Tox21の12タスクで平均AUC 0.911。**化学構造ベースであり病理画像は不使用。** コア問い2（動物病理所見からヒト有害事象を予測するアーキテクチャ）に対し、「知識グラフ+GNN」という設計パターンをそのまま流用できる候補として位置づけられる——ノードを化合物ではなくINHAND病変概念に置き換える拡張が考えられる。

---

### 7. Maciejewski, M., Lounkine, E., Whitebread, S., Farmer, P., DuMouchel, W., Shoichet, B. K., Urban, L. (2017)
**Reverse translation of adverse event reports paves the way for de-risking preclinical off-targets.**
*eLife*. [PMC5548487](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5548487/)

FAERS（870万件超の有害事象報告、1997–2015年）を化学構造(InChIKey)経由で薬剤にマッピングし、標的薬理学・薬物動態パラメータ（Cmax/IC50比等）と相関させて前臨床オフターゲットのリスクを同定。**方向はヒト市販後データ→前臨床（逆方向）**であり、動物データは不使用。PROP-07が目指す「動物→ヒト」の対となる先行研究として、同じFAERSデータソースの活用可能性を示す。

---

### 8. Gardiner, L. J., Carrieri, A. P., Wilshaw, J., Checkley, S., Pyzer-Knapp, E. O., Krishna, R. (2020)
**Combining human cell line transcriptome analysis and Bayesian inference to build trustworthy machine learning models for prediction of animal toxicity in drug development.**
*Scientific Reports*. [PMC7293302](https://pmc.ncbi.nlm.nih.gov/articles/PMC7293302/) / [arXiv:1911.04374](https://arxiv.org/pdf/1911.04374)

ヒトL1000遺伝子発現プロファイル(964遺伝子)+化学構造(MACCS)をガウス過程ベイズ回帰で統合し、ラット腎毒性（BUN値、DrugMatrix由来429化合物）を予測。特徴量を組み合わせることでテストR²が0.021→0.418（階層的SVD次元削減で0.661まで改善）。**方向はヒトin vitro→動物in vivo（逆方向）。** 不確実性定量化を組み込んだ「信頼できる」種間外挿モデルの設計思想は、PROP-07のモデル設計（予測の信頼区間提示）にも参考になる。

---

### 9. O'Donovan, S. D., Driessens, K., Lopatta, D., Wimmenauer, F., Lukas, A., Neeven, J., Smirnov, E., Lenz, M., Ertaylan, G., Jennen, D. G. J., van Riel, N. A. W., Cavill, R., Peeters, R. L. M., de Kok, T. M. C. M. (2020)
**Use of deep learning methods to translate drug-induced gene expression changes from rat to human primary hepatocytes.**
*PLOS ONE*, 15(8), e0236392. [DOI:10.1371/journal.pone.0236392](https://doi.org/10.1371/journal.pone.0236392)

Open TG-GATEs由来のラット肝細胞in vitro遺伝子発現データから、ボトルネック構造の深層ニューラルネットでヒト初代肝細胞の遺伝子発現変化を予測。CNN/ボトルネックDNNが古典的ML手法を上回る精度を達成。**ラット→ヒトの遺伝子発現レベル翻訳としては最も具体的な実装例の一つだが、WSI形態情報は一切関与しない。**

---

### 10. Mehrvar, S., Himmel, L. E., Babburi, P., Goldberg, A. L., Guffroy, M., Janardhan, K., Krempley, A. L., Bawa, B. (2021)
**Deep Learning Approaches and Applications in Toxicologic Histopathology: Current Status and Future Perspectives.**
*Journal of Pathology Informatics*, 12:42. [PMC8609289](https://pmc.ncbi.nlm.nih.gov/articles/PMC8609289/)

毒性病理DL全般の包括的レビュー（AbbVieの毒性病理医チーム）。WSI評価ワークフローのDL化を論じるが、**動物病理所見を用いてヒト臨床アウトカムを予測・外挿するという用途には一切言及がない**——毒性病理は「ヒト臨床試験の安全性を担保する前提条件」として位置づけられており、予測的な翻訳機構としては扱われていない。このネガティブな結果自体が、PROP-07のホワイトスペースを裏付ける状況証拠となる。

---

### 11. FDA (2025)
**Roadmap to Reducing Animal Testing in Preclinical Safety Studies.**
公式PDF: [https://www.fda.gov/files/newsroom/published/roadmap_to_reducing_animal_testing_in_preclinical_safety_studies.pdf](https://www.fda.gov/files/newsroom/published/roadmap_to_reducing_animal_testing_in_preclinical_safety_studies.pdf)（2025年4月発表）

FDA Modernization Act 2.0を受け、モノクローナル抗体を皮切りに動物試験を「例外」とし、オルガノイド・Organ-on-a-chip・in silicoモデリング・AIツール等のNew Approach Methodologies（NAMs）へ段階的に移行する3〜5年ロードマップ。2026年4月に1年目の目標達成を発表済み。**動物データそのものへの依存を減らす政策的圧力は、逆説的に「既存の動物毒性病理データからヒトへの外挿価値を最大化する」PROP-07的アプローチの規制的な追い風にもなり得る**（少ない動物データをより高精度に翻訳する必要性が増すため）。

---

## 未収録・確認できなかった事項
- FDA公式TranslAI専用ページ（`/translai-initiative`）は本調査時点で404を返し、詳細アーキテクチャを一次情報源から直接確認できなかった。
- 「動物病理WSI形態特徴を直接の入力としてヒト臨床アウトカムを予測する」論文は、本調査（WebSearch/WebFetchベース）の範囲では一件も発見できなかった。
