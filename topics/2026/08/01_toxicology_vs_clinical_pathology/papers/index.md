# 収集論文・リソース一覧 (Curated Papers & Resources)

本調査（毒性病理画像分野と疾患病理画像分野の比較・応用可能技術）で精読・参照した主要論文および関連リソースの書誌情報と要約です。

---

## 📚 論文メタデータ一覧

| No. | タイトル | 著者 / 年 / 会議・誌 | リンク (arXiv/DOI/PMC) | ローカルPDF | 重要度 |
|:---:|:---|:---|:---:|:---:|:---:|
| 1 | [UNI: A general-purpose foundation model for computational pathology](#paper-1) | Chen et al. (2024, Nature Medicine) | [DOI:10.1038/s41591-024-02857-3](https://doi.org/10.1038/s41591-024-02857-3) / [arXiv:2308.15474](https://arxiv.org/abs/2308.15474) | [PDF](pdfs/2024_Chen_UNI_PathologyFM.pdf) | ★★★ |
| 2 | [CONCH: A visual-language foundation model for computational pathology](#paper-2) | Lu et al. (2024, Nature Medicine) | [DOI:10.1038/s41591-024-02856-4](https://doi.org/10.1038/s41591-024-02856-4) / [arXiv:2308.16147](https://arxiv.org/abs/2308.16147) | [PDF](pdfs/2024_Lu_CONCH_VisionLanguage.pdf) | ★★★ |
| 3 | [Prov-GigaPath: A whole-slide foundation model for digital pathology from real-world data](#paper-3) | Xu et al. (2024, Nature) | [DOI:10.1038/s41586-024-07441-w](https://doi.org/10.1038/s41586-024-07441-w) / [arXiv:2405.13031](https://arxiv.org/abs/2405.13031) | [PDF](pdfs/2024_Xu_ProvGigaPath.pdf) | ★★★ |
| 4 | [Virchow: A Million-Slide Foundation Model for Cancer Diagnosis](#paper-4) | Vorontsov et al. (2024, Nature Medicine) | [DOI:10.1038/s41591-024-03141-0](https://doi.org/10.1038/s41591-024-03141-0) / [arXiv:2309.07778](https://arxiv.org/abs/2309.07778) | [PDF](pdfs/2023_Vorontsov_Virchow.pdf) | ★★★ |
| 5 | [TRACE: Deep Learning-based Modeling for Preclinical Drug Safety Assessment](#paper-5) | Bhattacharya et al. (2024, bioRxiv / PMC11291027) | [bioRxiv:10.1101/2024.07.24.604928](https://doi.org/10.1101/2024.07.24.604928) / [PMC11291027](https://pmc.ncbi.nlm.nih.gov/articles/PMC11291027/) | - | ★★★ |
| 6 | [STP SIG Opinion: Application of AI and ML to Digital Toxicologic Pathology](#paper-6) | Turner et al. (2020, Toxicologic Pathology) | [DOI:10.1177/0192623320959049](https://doi.org/10.1177/0192623320959049) | - | ★★★ |
| 7 | [The Last Mile: Opportunities and Challenges for Machine Learning in Digital Toxicologic Pathology](#paper-7) | Rudmann et al. (2021, Toxicologic Pathology) | [DOI:10.1177/01926233211041388](https://doi.org/10.1177/01926233211041388) | - | ★★★ |
| 8 | [Open TG-GATEs: a large-scale toxicogenomics database & AI applications](#paper-8) | Igarashi et al. (2015, NAR) / Karasawa et al. | [DOI:10.1093/nar/gku955](https://doi.org/10.1093/nar/gku955) | - | ★★★ |
| 9 | [CLAM: Data-efficient and weakly supervised computational pathology on whole-slide images](#paper-9) | Lu et al. (2021, Nature Biomedical Engineering) | [DOI:10.1038/s41551-020-00682-w](https://doi.org/10.1038/s41551-020-00682-w) / [arXiv:2004.09666](https://arxiv.org/abs/2004.09666) | [PDF](pdfs/2021_Lu_CLAM_WSI.pdf) | ★★☆ |
| 10 | [TransMIL: Transformer based Correlated Multiple Instance Learning for WSI Classification](#paper-10) | Shao et al. (2021, NeurIPS) | [NeurIPS 2021](https://proceedings.neurips.cc/paper/2021/hash/10c272d06794d3e5785d5e79c7943961-Abstract.html) / [arXiv:2106.00908](https://arxiv.org/abs/2106.00908) | [PDF](pdfs/2021_Shao_TransMIL.pdf) | ★★☆ |

---

## 📝 論文詳細サマリー

### <a id="paper-1"></a> [1] UNI: A general-purpose foundation model for computational pathology
- **著者**: Richard J. Chen, Tong Ding, Ming Y. Lu, Drew F. K. Williamson, Faisal Mahmood, et al.
- **所属**: Harvard Medical School, Brigham and Women's Hospital, Broad Institute
- **掲載**: Nature Medicine (2024) / arXiv:2308.15474
- **コード / モデル**: [Hugging Face (MahmoodLab/UNI)](https://huggingface.co/MahmoodLab/UNI)
- **PDF**: [pdfs/2024_Chen_UNI_PathologyFM.pdf](pdfs/2024_Chen_UNI_PathologyFM.pdf)

#### 概要 (Abstract TL;DR)
100,000枚以上のヒト病理WSI（20以上の主要臓器・組織）から抽出した1億枚超のパッチ画像を用い、DINOv2（自己教師あり学習）により事前学習された汎用病理ビジョン基盤モデル（ViT-Large）。

#### 手法のポイント
- **アーキテクチャ**: Vision Transformer (ViT-L/16)、解像度 224x224、1024次元特徴量。
- **学習方法**: DINOv2自己蒸留（Masked Image Modeling + Multi-crop Contrastive）。
- **評価タスク**: 34種類の臨床タスク（がん診断、サブタイピング、遺伝子変異予測、臓器移植拒絶反応判定等）で従来ImageNet事前学習や既存基盤モデルを大幅に凌駕。

#### 毒性病理への示唆
ヒト組織で学習された豊かな形態特徴表現は、動物病理へのファインチューニングやZero-shot/Few-shot転移の強力なバックボーンとなり得る。

---

### <a id="paper-2"></a> [2] CONCH: A visual-language foundation model for computational pathology
- **著者**: Ming Y. Lu, Bowen Chen, Drew F. K. Williamson, Richard J. Chen, Faisal Mahmood, et al.
- **所属**: Harvard Medical School, Brigham and Women's Hospital
- **掲載**: Nature Medicine (2024) / arXiv:2308.16147
- **コード / モデル**: [Hugging Face (MahmoodLab/CONCH)](https://huggingface.co/MahmoodLab/CONCH)
- **PDF**: [pdfs/2024_Lu_CONCH_VisionLanguage.pdf](pdfs/2024_Lu_CONCH_VisionLanguage.pdf)

#### 概要 (Abstract TL;DR)
117万件以上の病理組織画像と病理診断レポートテキストのペアで対照学習（Contrastive Learning + Captioning）を行った視覚言語（Vision-Language）基盤モデル。

#### 手法のポイント
- **アーキテクチャ**: CoCa (Contrastive Captioners) アーキテクチャを病理画像向けに最適化。
- **機能**: Zero-shot画像分類、病理テキストによる画像検索（Text-to-Image / Image-to-Text）、キャプション生成。
- **性能**: 14のベンチマークにおいて、CLIPやPubMedCLIPを圧倒するZero-shot精度を達成。

#### 毒性病理への示唆
毒性病理の標準用語体系（INHAND）のテキスト定義と画像特徴を直接結びつける「Toxicology-CONCH」の設計基盤となる。

---

### <a id="paper-3"></a> [3] Prov-GigaPath: A whole-slide foundation model for digital pathology from real-world data
- **著者**: Hanwen Xu, Naoto Usuyama, Jaspreet Bagga, Sheng Zhang, Rajesh Rao, Hoifung Poon, et al.
- **所属**: Microsoft Research, Providence Health System, University of Washington
- **掲載**: Nature (2024) / arXiv:2405.13031
- **コード / モデル**: [Hugging Face (providence/prov-gigapath)](https://huggingface.co/providence/prov-gigapath)
- **PDF**: [pdfs/2024_Xu_ProvGigaPath.pdf](pdfs/2024_Xu_ProvGigaPath.pdf)

#### 概要 (Abstract TL;DR)
13億枚以上のタイル画像（17万枚以上の全スライドWSI、31の主要組織）を用いたタイプレベル事前学習に加え、ギガピクセルWSI全体をコンテキストとして保持するLongNetアーキテクチャ（スライドレベル基盤モデル）を提案。

#### 手法のポイント
- **2段階基盤モデル**:
  1. Tile-level: DINOv2によるViT事前学習。
  2. Slide-level: LongNet（膨大なパッチ列に対するDilated Attention）によるWSI全体の文脈統合。
- **成果**: がんサブタイプ分類、全スライド変異予測、EGFR/BRAF等の遺伝子予測で最高精度を達成。

#### 毒性病理への示唆
毒性病理における「臓器全体に広がるびまん性変化」や「局所と周囲組織の相関」をモデル化するスライドレベル基盤モデルとして直接応用可能。

---

### <a id="paper-4"></a> [4] Virchow: A Million-Slide Foundation Model for Cancer Diagnosis
- **著者**: Eugene Vorontsov, Alican Bozkurt, Adam Casson, Siqi Liu, Matthew G. Hanna, Thomas J. Fuchs, et al.
- **所属**: Paige, Memorial Sloan Kettering Cancer Center (MSKCC)
- **掲載**: Nature Medicine (2024) / arXiv:2309.07778
- **PDF**: [pdfs/2023_Vorontsov_Virchow.pdf](pdfs/2023_Vorontsov_Virchow.pdf)

#### 概要 (Abstract TL;DR)
MSKCCから収集された10万枚〜100万枚規模の臨床WSIデータセットを用いて学習された632MパラメータのVision Transformerモデル。希少がんを含む多様な組織型で極めて高い汎化性能を発揮。

---

### <a id="paper-5"></a> [5] TRACE: Deep Learning-based Modeling for Preclinical Drug Safety Assessment
- **著者**: Sabyasachi Bhattacharya, et al. (Pfizer Preclinical Research)
- **掲載**: bioRxiv (2024), PMID: 39091793, PMCID: PMC11291027
- **概要**:
  - ラット（*Rattus norvegicus*）の157前臨床試験（46,734スライド、1,500万画像）を対象に学習された、毒性病理（肝臓）特化のマルチスケール深層学習フレームワーク。
  - 病変の重症度スコアリング、組織形態検索、そして**自動的な用量反応性（Dose-Response）キャラクタリゼーション**を実現。
  - 独立した毒性病理医リーダー研究において、病理医の合意判定（Consensus）との一致率で個々の病理医平均を上回る精度を実証。

---

### <a id="paper-6"></a> [6] STP SIG Opinion: Application of AI and ML to Digital Toxicologic Pathology
- **著者**: O. Turner, B. Aeffner, P. V. Bangari, R. B. Christian, et al. (Society of Toxicologic Pathology)
- **掲載**: Toxicologic Pathology (2020), Vol. 48(8), pp. 959-968.
- **概要**:
  - 米国毒性病理学会（STP）の専門部会による公式見解論文。
  - 毒性病理におけるAI導入のロードマップ、用語の標準化（INHAND）、GLP（Good Laboratory Practice）規制下におけるバリデーション要件、病理医の意思決定支援としてのAIの役割を体系的に定義。

---

### <a id="paper-7"></a> [7] The Last Mile: Opportunities and Challenges for Machine Learning in Digital Toxicologic Pathology
- **著者**: David G. Rudmann, et al.
- **掲載**: Toxicologic Pathology (2021), Vol. 49(8), pp. 1386-1393.
- **概要**:
  - アルゴリズム開発から実際の毒性試験ワークフロー（非臨床創薬パイプライン）への日常的統合における「ラストマイル問題」を論理的に分析。
  - 背景病変（Spontaneous lesions）のノイズ、大量のスライドスクリーニング（95%以上が正常）、規制当局（FDA/PMDA）の承認要件、説明責任を指摘。

---

### <a id="paper-8"></a> [8] Open TG-GATEs: A Large-scale Toxicogenomics Database & PathologAI
- **著者**: Y. Igarashi, et al. (National Institute of Biomedical Innovation, Health and Nutrition, Japan)
- **掲載**: Nucleic Acids Res (2015), Vol. 43(Database issue), pp. D921–D927.
- **概要**:
  - 170種類の化合物投与後のラット肝臓・腎臓組織のWSI（数万枚）とマイクロアレイ遺伝子発現データを統合した世界最大級の公開毒性データベース。
  - PathologAI等の後続研究により、弱教師あり学習（Weakly Supervised Learning）を用いた壊死・空胞変性の自動検出や、自己教師あり学習バックボーンの構築に広く利用されている。

---

### <a id="paper-9"></a> [9] CLAM: Data-efficient and weakly supervised computational pathology on whole-slide images
- **著者**: Ming Y. Lu, Drew F. K. Williamson, Tiffany Y. Chen, Faisal Mahmood, et al.
- **掲載**: Nature Biomedical Engineering (2021) / arXiv:2004.09666
- **PDF**: [pdfs/2021_Lu_CLAM_WSI.pdf](pdfs/2021_Lu_CLAM_WSI.pdf)
- **概要**:
  - Attentionベースの弱教師あり学習（MIL）とクラスタリング正則化を組み合わせ、スライドレベルのラベルのみからギガピクセルWSIの分類と病変部位の高解像度ヒートマップ可視化を実現した画期的研究。

---

### <a id="paper-10"></a> [10] TransMIL: Transformer based Correlated Multiple Instance Learning
- **著者**: Zhuchen Shao, Hao Bian, Yang Chen, Yifeng Wang, Jian Zhang, et al.
- **掲載**: NeurIPS (2021) / arXiv:2106.00908
- **PDF**: [pdfs/2021_Shao_TransMIL.pdf](pdfs/2021_Shao_TransMIL.pdf)
- **概要**:
  - パッチ間の空間的・形態的相関をSelf-Attention（Transformer）によって捉えるMIL手法。局所パッチの独立性を仮定する従来のMILの限界を打破。
