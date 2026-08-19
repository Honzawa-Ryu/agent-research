# 収集論文・リソース一覧

本調査で参照・精読した論文および関連リソースの書誌情報と要約です。

---

## 📚 論文メタデータ一覧

| No. | タイトル | 著者 / 年 / 会議 | リンク (arXiv/DOI) | Code | ローカルPDF | 重要度 |
|:---|:---|:---|:---|:---|:---|:---:|
| 1 | [Open TG-GATEs: a large-scale toxicogenomics database](#paper-1) | Igarashi et al. (2015) | [DOI:10.1093/nar/gku955](https://doi.org/10.1093/nar/gku955) | - | - | ★★★ |
| 2 | [AI-driven Discovery of Morphomolecular Signatures in Toxicology (GEESE)](#paper-2) | Jaume et al. (2024) | [bioRxiv:10.1101/2024.07.19.604355](https://www.biorxiv.org/content/10.1101/2024.07.19.604355) / [PMC11291055](https://pmc.ncbi.nlm.nih.gov/articles/PMC11291055/) | - | [PDF](pdfs/2024_Jaume_GEESE.pdf) | ★★★ |
| 3 | [A Weakly Supervised Deep Learning Framework for Whole Slide Classification to Facilitate Digital Pathology in Animal Study (PathologAI)](#paper-3) | Bussola et al. (2023) | Chemical Research in Toxicology / [DOI:10.1021/acs.chemrestox.3c00058](https://doi.org/10.1021/acs.chemrestox.3c00058) / [PMC10445282](https://pmc.ncbi.nlm.nih.gov/articles/PMC10445282/) | - | - | ★★★ |
| 4 | [A multi-label learning model for predicting drug-induced pathology in multi-organ based on toxicogenomics data (Att-RethinkNet)](#paper-4) | Su et al. (2022) | PLOS Computational Biology / [DOI:10.1371/journal.pcbi.1010402](https://doi.org/10.1371/journal.pcbi.1010402) | [GitHub](https://github.com/RanSuLab/Drug-Toxicity-Prediction-MultiLabel) | [PDF](pdfs/2022_Su_AttRethinkNet.pdf) | ★★★ |
| 5 | [Transcriptomics-guided Slide Representation Learning in Computational Pathology (TANGLE)](#paper-5) | Jaume et al. (2024) | CVPR 2024 / [arXiv:2405.11618](https://arxiv.org/abs/2405.11618) | [GitHub](https://github.com/mahmoodlab/TANGLE) | [PDF](pdfs/2024_Jaume_TANGLE.pdf) | ★★★ |
| 6 | [Bridging organ transcriptomics for advancing multiple organ toxicity assessment with a generative AI approach (TransTox)](#paper-6) | Li, Chen & Tong (2024) | npj Digital Medicine / [DOI:10.1038/s41746-024-01317-z](https://doi.org/10.1038/s41746-024-01317-z) | - | [PDF](pdfs/2024_Li_TransTox.pdf) | ★★☆ |
| 7 | [A generative adversarial network model alternative to animal studies for clinical pathology assessment (AnimalGAN)](#paper-7) | Ding et al. (2023) | Nature Communications / [DOI:10.1038/s41467-023-42933-9](https://doi.org/10.1038/s41467-023-42933-9) | - | - | ★★☆ |
| 8 | [From slides (through tiles) to pixels: an explainability framework for weakly supervised models in pre-clinical pathology](#paper-8) | Bertolini et al. (2023) | [arXiv:2302.01653](https://arxiv.org/abs/2302.01653) | - | - | ★★☆ |
| 9 | [Mapping safety in space: the emerging role of spatial transcriptomics in safe drug development](#paper-9) | Golfinos-Owens et al. (2026) | Frontiers in Toxicology / [DOI:10.3389/ftox.2026.1817521](https://doi.org/10.3389/ftox.2026.1817521) | - | - | ★★☆ |
| 10 | [Evolution of artificial intelligence and machine learning in DILI toxicogenomics: from descriptive profiling to mechanistic insights](#paper-10) | (著者名 検索結果に不掲載) (2026) | Frontiers in Pharmacology / [DOI:10.3389/fphar.2026.1907884](https://doi.org/10.3389/fphar.2026.1907884) | - | - | ★★☆ |

---

## 📝 論文詳細サマリー

### <a id="paper-1"></a> [1] Open TG-GATEs: a large-scale toxicogenomics database
- **著者**: Yoshinobu Igarashi, Noriyuki Nakatsu, Tomoya Yamashita, et al.
- **掲載**: Nucleic Acids Research, 43(D1), D921–D927 (2015)
- **リンク**: [DOI:10.1093/nar/gku955](https://doi.org/10.1093/nar/gku955)

#### 概要
- 170化合物についてラット（in vivo肝・腎、in vitro初代肝細胞）およびヒト初代肝細胞への曝露データを収録した世界最大級の公開トキシコゲノミクスデータベース。生化学・血液学・病理組織所見・遺伝子発現マイクロアレイを化合物・用量・時点別に階層的に格納。**病理画像（WSI, SVS形式）自体も一部公開**されている（LSDB Archiveより配布）。
- **本調査への示唆**: PROP-03全体の基盤データセット。GEESE/PathologAI等はいずれも本データベースのWSIサブセットを利用。

---

### <a id="paper-2"></a> [2] AI-driven Discovery of Morphomolecular Signatures in Toxicology (GEESE)
- **著者**: Guillaume Jaume, Thomas Peeters, Andrew H. Song, ほか（Mahmood Lab: Brigham and Women's Hospital / Massachusetts General Hospital / Broad Institute / Dana-Farber Cancer Institute）
- **掲載**: bioRxiv preprint, 2024-07（未査読）
- **リンク**: [bioRxiv:10.1101/2024.07.19.604355](https://www.biorxiv.org/content/10.1101/2024.07.19.604355) / [PMC11291055](https://pmc.ncbi.nlm.nih.gov/articles/PMC11291055/)
- **PDF**: [pdfs/2024_Jaume_GEESE.pdf](pdfs/2024_Jaume_GEESE.pdf)

#### 概要 (Abstract TL;DR)
- Open TG-GATEs全156試験・10,234枚のラット肝臓H&E WSIと対応する遺伝子発現プロファイルのペアを用い、iBOT自己教師あり学習ViTベースのMILモデル「GEESE」を構築。WSIから1,536遺伝子ターゲットのバルク発現量を回帰予測し、パッチ単位の擬似空間発現マップを生成。

#### 手法のポイント
- **アーキテクチャ/入出力**: 256×256px（128μm解像度）パッチ→iBOT ViTエンコーダ（768次元embedding）→MILアグリゲーションで1,536遺伝子の発現量を回帰。スライドレベルラベルのみで学習（パッチレベル注釈不要）。
- **データ分割**: 開発127試験・8,231枚、テスト29試験・2,002枚（うち病変含有17%）。
- **評価結果**: 全遺伝子平均Pearson相関 r=0.29、予測精度上位100遺伝子でr=0.63（最高はTNFRSF12A r=0.722）。**6種類の肝病変分類ではmacro-AUC 98.9%**という高精度を達成。化合物により相関はr<0.5〜r>0.8とばらつき（チオアセトアミドが最良）。
- **順方向のみ**: WSI→遺伝子発現の回帰予測に特化。化合物構造や遺伝子発現からのWSI生成（逆方向）は扱わない。

#### 結果・貢献と限界
- **本調査への示唆**: 報告書01がフロンティア5で「未開拓」と位置づけたPatho-TGxの順方向（形態→分子）予測は、同一グループ（TRACE開発チーム）により2024年半ばの時点で相当規模で実装済み。ただし「病変分類（カテゴリカル）」の精度と「遺伝子発現値そのもの（連続値）」の予測精度には大きな乖離があり、後者は依然として発展途上。
- 論文自身が明記する限界: 単一臓器（肝臓）限定、in vitro（初代肝細胞）とin vivoの乖離、種差（ラット/ヒト肝細胞）、空間トランスクリプトミクスによる直接検証データの不足。

---

### <a id="paper-3"></a> [3] A Weakly Supervised Deep Learning Framework for Whole Slide Classification to Facilitate Digital Pathology in Animal Study (PathologAI)
- **著者**: Nicole Bussola, Jie Xu, Leihong Wu, Luca Gorini, Yiyi Zhang, Cesare Furlanello, Weida Tong
- **所属**: FDA National Center for Toxicological Research (NCTR) / FBK (Fondazione Bruno Kessler)
- **掲載**: Chemical Research in Toxicology, 36(8), 1321–1331 (2023)
- **リンク**: [DOI:10.1021/acs.chemrestox.3c00058](https://doi.org/10.1021/acs.chemrestox.3c00058) / [PMC10445282](https://pmc.ncbi.nlm.nih.gov/articles/PMC10445282/)

#### 概要 (Abstract TL;DR)
- FDA NCTRの「AI4TOX」プログラム（AnimalGAN, SafetAI, BERTox, PathologAIの4本柱）の一角。Open TG-GATEsラット肝臓WSI 816枚（Control-NF 257 / Control-F 120 / Mild-F SP 157 / Mild-F NSP 167 / Positive-F 115）を用い、BiGAN埋め込み＋CNNアンサンブル（5モデル）で壊死の自発性/薬剤誘発性を弱教師あり分類。

#### 手法のポイント
- 前処理（histolabタイル抽出）→BiGANで128次元埋め込み→7層CNN×5のアンサンブル分類器。外部検証で自発性壊死82.5%、治療関連壊死58.7〜66.7%の精度。自発性 vs 治療関連の識別MCC=0.48。
- **トキシコゲノミクス統合は未実施**（病理画像単体の弱教師あり分類）。論文末尾で「将来的にToxGANと統合し、遺伝子発現シグネチャの空間的マッピングを行う」という構想を明記。

#### 本調査への示唆
- この「ToxGAN×PathologAI統合」構想こそがPROP-03の「逆方向モデル」に最も近いが、**2026年8月時点で実装論文は確認できず**、未解決フロンティアであることの直接的な一次証拠となる。

---

### <a id="paper-4"></a> [4] A multi-label learning model for predicting drug-induced pathology in multi-organ based on toxicogenomics data (Att-RethinkNet)
- **著者**: Ran Su, Haitang Yang, Leyi Wei, Siqi Chen, Quan Zou
- **掲載**: PLOS Computational Biology, 18(9), e1010402 (2022)
- **リンク**: [DOI:10.1371/journal.pcbi.1010402](https://doi.org/10.1371/journal.pcbi.1010402) / [GitHub](https://github.com/RanSuLab/Drug-Toxicity-Prediction-MultiLabel)
- **PDF**: [pdfs/2022_Su_AttRethinkNet.pdf](pdfs/2022_Su_AttRethinkNet.pdf)

#### 概要 (Abstract TL;DR)
- 化合物・用量・投与期間の情報とトキシコゲノミクス（遺伝子発現）データを入力に、肝臓・腎臓の複数の病理所見を**同時にマルチラベル予測**するAttention＋RethinkNet（マルチラベル分類のメモリ構造）モデル。ラベル間の相関を明示的に活用する点が新規性。

#### 手法のポイント
- 単一所見の二値分類に留まっていた従来手法（毒性あり/なしのみ）を拡張し、複数の病理所見を同時かつ相互依存的に予測。肝・腎両臓器に適用可能。
- **画像（WSI）は入力に含まない** — 遺伝子発現ベクトル＋化合物メタデータのみで病理所見ラベルを予測する「TGx→病理」の逆方向的タスクだが、出力は画像ではなくカテゴリカルラベル。

#### 本調査への示唆
- 「遺伝子発現から病理所見を予測する」枠組みとしてはPatho-TGxの部分的先行例だが、出力が「所見ラベル」に留まり「病理組織像そのもの」を生成しない点で、報告書01が構想する"Virtual Histopathology"（画像生成）とは異なる。

---

### <a id="paper-5"></a> [5] Transcriptomics-guided Slide Representation Learning in Computational Pathology (TANGLE)
- **著者**: Guillaume Jaume, Lukas Oldenburg, Anurag Vaidya, Richard J. Chen, Drew F. K. Williamson, Thomas Peeters, Andrew Song, Faisal Mahmood
- **所属**: Mahmood Lab (Harvard Medical School / Brigham and Women's Hospital)
- **掲載**: CVPR 2024 (Oral)
- **リンク**: [arXiv:2405.11618](https://arxiv.org/abs/2405.11618) / [GitHub](https://github.com/mahmoodlab/TANGLE)
- **PDF**: [pdfs/2024_Jaume_TANGLE.pdf](pdfs/2024_Jaume_TANGLE.pdf)

#### 概要 (Abstract TL;DR)
- WSIと対応する遺伝子発現プロファイルをCLIP型の対照学習でアライメントする「Slide + Expression (S+E)」事前学習フレームワーク。肝臓（n=6,597ペア）・乳腺（n=1,020）・肺（n=1,012）で事前学習し、few-shot分類・スライド検索で教師ありベースラインを上回る性能を実証。

#### 本調査への直接的な関連（独自発見）
- **肝臓の事前学習データがヒト（Homo sapiens）とラット（Rattus norvegicus）の両種、計6,597ペアで構成されている**ことを確認。毒性病理を明示的な応用先とは謳っていないが、結果として「動物種横断（フロンティア1）」と「トキシコゲノミクス統合（フロンティア5）」を同時に部分的に満たす、意図せざる先行事例になっている。
- Frontiers in Pharmacology 2026の総説は、この「分子情報でガイドされた表現学習」が毒性病理のfew-shot病変分類に有望と指摘している（TANGLE論文自体にはその明記なし）。
- PROP-01（動物種横断）の既存レポートでは本論文は未言及であり、本調査で新たに発見した接続点。

---

### <a id="paper-6"></a> [6] Bridging organ transcriptomics for advancing multiple organ toxicity assessment with a generative AI approach (TransTox)
- **著者**: Ting Li, Xi Chen, Weida Tong
- **所属**: FDA National Center for Toxicological Research (NCTR)
- **掲載**: npj Digital Medicine, 7, 314 (2024)
- **リンク**: [DOI:10.1038/s41746-024-01317-z](https://doi.org/10.1038/s41746-024-01317-z)
- **PDF**: [pdfs/2024_Li_TransTox.pdf](pdfs/2024_Li_TransTox.pdf)

#### 概要 (Abstract TL;DR)
- GANベースの「TransTox」により、肝臓⇔腎臓間で薬剤誘発性の遺伝子発現プロファイルを双方向に相互翻訳。片方の臓器のトキシコゲノミクスデータしかない場合でも、もう一方の臓器での毒性機序を仮想的に推定する「デジタルツイン」的アプローチ。

#### 本調査への示唆
- 「仮想毒性病理（Virtual Toxicopathology）」という思想の先行例だが、**入出力はあくまで遺伝子発現ベクトルであり、病理組織像（WSI）は生成しない**。報告書01フロンティア5が構想する「SMILES→仮想病理像シミュレーション」とは異なるレイヤーでの"virtual"実装であることに注意。

---

### <a id="paper-7"></a> [7] A generative adversarial network model alternative to animal studies for clinical pathology assessment (AnimalGAN)
- **著者**: Xi Chen, ほか (FDA NCTR, Weida Tongグループ)
- **掲載**: Nature Communications, 14, 7040 (2023)
- **リンク**: [DOI:10.1038/s41467-023-42933-9](https://doi.org/10.1038/s41467-023-42933-9)

#### 概要
- GANにより38種のラット臨床病理検査値（血液生化学等）を生成し、実動物を用いずに肝毒性等を評価する仮想動物実験モデル。12種の従来QSAR手法を上回る性能。FDA AI4TOXプログラムの一部。

#### 本調査への示唆（隣接領域として言及）
- **WSIや組織形態は対象外**（血液生化学パラメータのみ）。Patho-TGxと直接の技術的重なりはないが、同じFDA NCTRエコシステム内で「仮想動物実験」思想が実装されている隣接事例として参考に採用。

---

### <a id="paper-8"></a> [8] From slides (through tiles) to pixels: an explainability framework for weakly supervised models in pre-clinical pathology
- **著者**: Marco Bertolini, Van-Khoa Le, Jake Pencharz, Andreas Poehlmann, Djork-Arné Clevert, Santiago Villalba, Floriane Montanari
- **掲載**: arXiv preprint, 2023-02
- **リンク**: [arXiv:2302.01653](https://arxiv.org/abs/2302.01653)

#### 概要
- 前臨床（動物）病理WSIの弱教師あり多施設インスタンス学習（MIL）モデルに対し、パッチ→タイル→ピクセルレベルの説明可能性（XAI）ヒートマップを生成するフレームワーク。病理医アノテーションとの相関を確認。

#### 本調査への示唆
- トキシコゲノミクス統合ではなく説明可能性が主眼だが、TG-GATEs系ではない社内前臨床WSIデータでのMIL実装例として、PathologAI/GEESEとの技術比較の参考に採用。

---

### <a id="paper-9"></a> [9] Mapping safety in space: the emerging role of spatial transcriptomics in safe drug development
- **著者**: Golfinos-Owens, ほか
- **掲載**: Frontiers in Toxicology, 2026
- **リンク**: [DOI:10.3389/ftox.2026.1817521](https://doi.org/10.3389/ftox.2026.1817521)

#### 概要
- 空間トランスクリプトミクス（ST）を前臨床安全性評価に統合する総説。acetaminophen肝毒性（Cao 2024）、虚血再灌流（Xin 2023）、TCDD曝露（Nault 2023）、losartan腎所見（Onoda 2022）、cisplatin腎毒性（Wijaya 2025）等、HE組織像とSTを直接対応付けた実装例を整理。焦点性・帯状分布病変に最も有効で、種特異的プローブ開発が前臨床種への適用の障壁と指摘。

#### 本調査への示唆
- 報告書01フロンティア5「空間オミクス統合」の毒性病理版の現在地を示す最新総説。GEESE/TANGLEのようなバルク発現予測とは異なり、真の空間解像度でのWSI-オミクス対応付けはまだ個別の実験系（LCM、Visium等）に依存し、汎用AIモデル化は未成熟。

---

### <a id="paper-10"></a> [10] Evolution of artificial intelligence and machine learning in DILI toxicogenomics: from descriptive profiling to mechanistic insights
- **掲載**: Frontiers in Pharmacology, 2026
- **リンク**: [DOI:10.3389/fphar.2026.1907884](https://doi.org/10.3389/fphar.2026.1907884)

#### 概要
- DILIトキシコゲノミクス領域におけるAI/ML手法の総説。INSIGHT (Zhao et al. 2024, 235遺伝子DILIシグネチャ, AUC≈0.71)、GEESE (macro-AUC 98.9%)等をTable形式で整理。

#### 本調査への示唆・注意点
- 本調査ではこの総説を経由してGEESE・INSIGHTの存在を発見した。**INSIGHT原論文には直接到達できておらず**、本総説のTable記載値（235遺伝子、AUC≈0.71）は二次情報として扱う。原論文の書誌情報（雑誌名・巻号）が確認できていない点は限界として明記する。
