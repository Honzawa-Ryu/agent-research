# 収集論文・リソース一覧 (Curated Papers & Resources)

本調査（動物種横断病理基盤モデルの構築手法と形態学的相同性アライメント技術）で精読・参照した主要論文および関連リソースの書誌情報と要約です。

---

## 📚 論文メタデータ一覧

| No. | タイトル | 著者 / 年 / 会議・誌 | リンク (arXiv/DOI/PMC) | ローカルPDF | 重要度 |
|:---:|:---|:---|:---:|:---:|:---:|
| 1 | [Bridging clinic and wildlife care with AI-powered pan-species computational pathology](#paper-1) | AbdulJabbar, Castillo, et al. (2023, Nature Communications) | [DOI:10.1038/s41467-023-37879-x](https://doi.org/10.1038/s41467-023-37879-x) / [PMC10133243](https://pmc.ncbi.nlm.nih.gov/articles/PMC10133243/) | - | ★★★ |
| 2 | [Lost in Translation: How Language Re-Aligns Vision for Cross-Species Pathology](#paper-2) | Arora (2026, arXiv preprint) | [arXiv:2603.04405](https://arxiv.org/abs/2603.04405) | [PDF](pdfs/2026_Arora_LostInTranslation_CrossSpeciesCLIP.pdf) | ★★☆ |
| 3 | [Self-supervised large-scale kidney abnormality detection in drug safety assessment studies](#paper-3) | Slootweg, García-De-La-Puente, Litjens, Dammak (2025, arXiv) | [arXiv:2509.00131](https://arxiv.org/abs/2509.00131) | [PDF](pdfs/2025_Slootweg_SSL_KidneyAbnormality.pdf) | ★★★ |
| 4 | [Toxicity Assessment in Preclinical Histopathology via Class-Aware Mahalanobis Distance for Known and Novel Anomalies](#paper-4) | Graf, et al. (2026, Scientific Reports / arXiv) | [arXiv:2602.02124](https://arxiv.org/abs/2602.02124) / [DOI:10.1038/s41598-026-56510-9](https://doi.org/10.1038/s41598-026-56510-9) | [PDF](pdfs/2026_Graf_MahalanobisToxicityAssessment.pdf) | ★★★ |
| 5 | [Comparison of an Attention-Based MIL With a Visual Transformer Model for Histopathologic Lesions in the Rat Liver](#paper-5) | Funk, Clement, Togninalli, et al. (2025, Toxicologic Pathology) | [DOI:10.1177/01926233251339653](https://doi.org/10.1177/01926233251339653) | - | ★★★ |
| 6 | [Diagnostic classification in toxicologic pathology using attention-guided weak supervision and WSI features: a pilot study in rat livers](#paper-6) | (2025, Scientific Reports) | [DOI:10.1038/s41598-025-86615-6](https://doi.org/10.1038/s41598-025-86615-6) | - | ★★☆ |
| 7 | [A Weakly Supervised Deep Learning Framework (PathologAI) for Whole Slide Classification in Animal Study](#paper-7) | Bussola, Xu, Wu, Furlanello, Tong, et al. (2023, Chem. Res. Toxicol.) | [DOI:10.1021/acs.chemrestox.3c00058](https://doi.org/10.1021/acs.chemrestox.3c00058) / [PMC10445282](https://pmc.ncbi.nlm.nih.gov/articles/PMC10445282/) | - | ★★☆ |
| 8 | [Deep Learning Approaches and Applications in Toxicologic Histopathology: Current Status and Future Perspectives](#paper-8) | Mehrvar, Himmel, Babburi, et al. (2021, J Pathol Inform, AbbVie) | [DOI:10.4103/jpi.jpi_36_21](https://doi.org/10.4103/jpi.jpi_36_21) / [PMC8609289](https://pmc.ncbi.nlm.nih.gov/articles/PMC8609289/) | - | ★★★ |
| 9 | [PathoTune: Adapting Visual Foundation Model to Pathological Specialists](#paper-9) | Lu, et al. (2024, MICCAI) | [arXiv:2403.16497](https://arxiv.org/abs/2403.16497) | - | ★★☆ |
| 10 | [Digital Pathology and Artificial Intelligence Applied to Nonclinical Toxicology Pathology—The Current State, Challenges, and Future Directions](#paper-10) | Pohlmeyer-Esch, Halsey, Boisclair, et al. (2025, Toxicologic Pathology) | [DOI:10.1177/01926233251340622](https://doi.org/10.1177/01926233251340622) / [PMC12612283](https://pmc.ncbi.nlm.nih.gov/articles/PMC12612283/) | - | ★★★ |
| 11 | [Reporting transparency in veterinary pathology deep learning: A systematic review of reproducibility-critical details](#paper-11) | Banerjee, Bertram, Weiss, et al. (2026, Veterinary Pathology) | [DOI:10.1177/03009858261459452](https://doi.org/10.1177/03009858261459452) | - | ★★☆ |

---

## 📝 論文詳細サマリー

### <a id="paper-1"></a> [1] Bridging clinic and wildlife care with AI-powered pan-species computational pathology
- **著者**: Khalid AbdulJabbar, Simon P. Castillo（共同筆頭）, et al.
- **掲載**: Nature Communications (2023) / DOI:10.1038/s41467-023-37879-x

#### 概要 (Abstract TL;DR)
ヒト肺がんで学習した細胞検出・分類パイプラインを**改変なしに**脊椎動物20種（哺乳類・爬虫類・鳥類・両生類）120枚のがん病理スライドへ適用し、汎種病理アトラス「panspecies.ai」を構築。核検出（空間制約型CNN: MicroNet）→4クラス細胞分類（がん細胞/リンパ球/非炎症性間質細胞/その他）という2段パイプラインを、種を問わず単一モデルで走らせた。

#### 手法のポイント
- **morphospace overlap指標**: 各細胞から27種の形態学的特徴量を抽出→t-SNEで2次元に圧縮→2クラス間の分布重複度を計算。この値と分類精度がPearson相関0.79（p=2.8×10⁻⁵）で強く相関し、「対象種・病変タイプがヒト学習モデルにどれだけ転移しやすいかを事前に定量評価できる」ことを実証。
- **精度は病変タイプに強く依存**: 上皮腫瘍で0.79〜0.94と高精度な一方、円形細胞腫瘍（リンパ腫等）では0.57〜0.70に低下。免疫区画の種間多様性が主因。
- 犬の黒色腫（88例）・前立腺癌（12例）等の臨床コホートにも適用し、空間免疫スコア（腫瘍細胞へのリンパ球近接度）が予後（HR=0.55, p=0.038）と関連することを確認。

#### 毒性病理への示唆
- morphospace overlap指標は、動物種横断病理基盤モデルを「どの病変・どの種に適用可能か」を事前スクリーニングする定量ツールとして直接転用できる。ただし本研究は腫瘍性病変が対象であり、毒性病理で中心となる**非腫瘍性病変（肥大・壊死・空胞変性等）**への拡張は未検証（PROP-01のコア問いへ直結する未解決課題）。
- 「種特異的モデルよりヒト学習モデルの方が汎種転移に優れる」（犬前立腺癌学習モデルの汎種平均精度0.62 vs ヒト学習モデル）という逆説的知見は、動物種ごとに個別モデルを作るより、ヒト基盤モデルを起点にする設計思想を後押しする。

---

### <a id="paper-2"></a> [2] Lost in Translation: How Language Re-Aligns Vision for Cross-Species Pathology
- **著者**: Ekansh Arora（Thomas Jefferson High School for Science and Technology）
- **掲載**: arXiv preprint (2026-01-24) / arXiv:2603.04405
- **PDF**: [pdfs/2026_Arora_LostInTranslation_CrossSpeciesCLIP.pdf](pdfs/2026_Arora_LostInTranslation_CrossSpeciesCLIP.pdf)

> ⚠️ **査読前プレプリント・独立研究者による単著論文**。手法的着想は有用だが、査読済み論文と同列には扱わず、アイデアの一つとして参照する。

#### 概要 (Abstract TL;DR)
病理視覚言語基盤モデル「CPath-CLIP」（ViT-L/14ベース）を犬の乳腺癌WSI（22,239パッチ）で少数ショット微調整したところ、同一がん種内・交差がん種では性能改善する一方、交差種（犬→ヒトTCGA-BRCA）評価では性能が頭打ちになる現象を発見。原因分析の結果、腫瘍/正常プロトタイプ間のコサイン類似度が0.99以上という「意味的崩壊（Semantic Collapse）」が種特異的形態情報にドミネートされて生じていることを特定。

#### 手法のポイント
- **Semantic Anchoring**: 視覚バックボーンを完全凍結し、正規化視覚埋め込みと正規化テキスト埋め込みのコサイン類似度で分類する手法に置き換えることで、プロトタイプ崩壊を回避。
- 実験結果（AUC）: ゼロショット交差種63.96%→テキストアンカリングで78.39%（+14.43pt改善）。ただし交差がん種設定では汎用プロンプトの影響で逆に低下するトレードオフも確認。

#### 毒性病理への示唆
「視覚バックボーンは凍結したまま、テキスト（INHAND定義文等）を意味的な座標軸として使い種差を補正する」というアプローチは、PROP-05（INHAND準拠Tox-VLM）と技術的に直結し、動物種横断基盤モデルとVLMアプローチを統合する接点になり得る。

---

### <a id="paper-3"></a> [3] Self-supervised large-scale kidney abnormality detection in drug safety assessment studies
- **著者**: Ivan Slootweg, Natalia P. García-De-La-Puente, Geert Litjens, Salma Dammak
- **掲載**: arXiv preprint (2025-08-29) / arXiv:2509.00131
- **PDF**: [pdfs/2025_Slootweg_SSL_KidneyAbnormality.pdf](pdfs/2025_Slootweg_SSL_KidneyAbnormality.pdf)

#### 概要 (Abstract TL;DR)
158化合物にわたる大規模な薬剤安全性評価試験の腎臓WSIを対象に、**ヒト病理基盤モデルUNIの特徴量**を用いた自己教師あり異常検知を実施。単純なk近傍分類器では性能不十分であり、自己教師あり学習による改良が必要だったと報告。

#### 結果・毒性病理への示唆
AUC 0.62、陰性的中率89%という結果は、「ヒト病理基盤モデル（UNI）をそのまま動物毒性病理の異常検知に転用した場合、実用に耐えるが完全ではない中程度の性能に留まる」ことを示す具体的な定量エビデンス。本調査で発見した中で、**ヒト病理FMを実際に動物毒性病理へ適用した数少ない実証研究**であり、フロンティア1の「なぜ未開拓か」を裏付ける最重要データポイントの一つ。

---

### <a id="paper-4"></a> [4] Toxicity Assessment in Preclinical Histopathology via Class-Aware Mahalanobis Distance for Known and Novel Anomalies
- **著者**: Olga Graf, et al.（テュービンゲン大学 AI Center / Boehringer Ingelheim）
- **掲載**: Scientific Reports (2026-06) / arXiv:2602.02124
- **PDF**: [pdfs/2026_Graf_MahalanobisToxicityAssessment.pdf](pdfs/2026_Graf_MahalanobisToxicityAssessment.pdf)

#### 概要 (Abstract TL;DR)
C57BL/6Jマウス肝臓WSI 742枚（44前臨床試験）を対象に、**自然画像で自己教師あり学習されたDINOv2**を凍結エンコーダとして採用し、LoRA（rank=3）でセグメンテーションヘッドのみ適応。クラス別Mahalanobis距離のしきい値で「既知の病理」と「学習データにない未知の異常（OOD）」を分離する。

#### 結果・毒性病理への示唆
病理→正常への誤分類0.16%、正常→病理への誤分類0.35%、既知異常のクラス割当精度93.93%、未知異常（OOD）検知精度89.38%と高性能。ただし**マウス肝臓1臓器・1種に限定**され、著者ら自身が「他臓器・他種への一般化にはデータセット固有の再適応が必要」と明記。LoRAによる効率的適応の実証例としては最も具体的だが、種横断ではなく同一種内での検証に留まる点が今後の拡張余地。

---

### <a id="paper-5"></a> [5] Comparison of an Attention-Based MIL With a Visual Transformer Model for Histopathologic Lesions in the Rat Liver
- **著者**: Juergen Funk, Gregoire Clement, Matteo Togninalli, et al.（Roche/Genentech ほか）
- **掲載**: Toxicologic Pathology, 53(5) (2025) / DOI:10.1177/01926233251339653

#### 概要 (Abstract TL;DR)
ラット肝臓WSI 1,641枚（58試験）・腎臓WSI 646枚（24試験）を用い、Attention-based MIL（Campanellaアルゴリズム改変）とVision Transformerの2手法を比較。両モデルとも**ImageNet事前学習**（ResNet34/EfficientNet）を出発点とし、ヒト病理基盤モデルは未使用。

#### 結果・毒性病理への示唆
隠蔽テストセットAUROC: MIL 65.5〜73.9% に対しTransformer 77.3%と、Transformerが優位。広範な病変（巣状壊死等）の検出は得意な一方、単細胞レベルの微小病変検出には課題が残ることを明記。**製薬企業の実データ（1,600枚超）での直接比較**という点で信頼性が高く、「現行の毒性病理MILはヒト病理FMではなく自然画像事前学習に依存している」という業界の現状を示す重要な実証。

---

### <a id="paper-6"></a> [6] Diagnostic classification in toxicologic pathology using attention-guided weak supervision and WSI features: a pilot study in rat livers
- **掲載**: Scientific Reports (2025-02-04) / DOI:10.1038/s41598-025-86615-6

#### 概要 (Abstract TL;DR)
ラット肝臓の病変分類に対し、最小限のアノテーションと自己教師ありVision Transformerによる特徴抽出、および毒性病理特有の高度なクラス不均衡（95%以上が正常）に適した新しいguided attention機構を組み合わせた弱教師あり手法を提案。既存のCLAM（clustering-constrained-attention MIL）に対し、複数病変クラスでAUCが38%改善。

#### 毒性病理への示唆
「クラス不均衡に強いattention機構」という設計思想は、PROP-02（用量反応性階層Group-MIL）とも接続する。自己教師あり特徴抽出器の由来（動物データ自己教師あり事前学習かヒトFM転移か）は要追加確認。

---

### <a id="paper-7"></a> [7] A Weakly Supervised Deep Learning Framework (PathologAI) for Whole Slide Classification to Facilitate Digital Pathology in Animal Study
- **著者**: Nicole Bussola, Joshua Xu, Leihong Wu, Cesare Furlanello, Weida Tong, et al.（University of Trento / FDA National Center for Toxicological Research）
- **掲載**: Chemical Research in Toxicology, 36 (2023) / DOI:10.1021/acs.chemrestox.3c00058

#### 概要 (Abstract TL;DR)
Open TG-GATEsのラット肝臓WSI 816枚（377対照群含む）を対象に、BiGAN（双方向GAN）でタイルを128次元ベクトルに圧縮し、5モデルのアンサンブルCNNで分類する弱教師あり手法「PathologAI」。自然発生壊死と被験物質誘発性壊死を区別し、外部検証で67〜87%の精度を達成。

#### 毒性病理への示唆
本手法は種横断ではなくラット単一種内の分類だが、「自然発生病変と誘発病変の混在」という毒性病理特有の交絡問題（PROP-04と直結）への具体的な対処例として重要。topics/2026/08/01のフロンティア5（Patho-TGx）でも参照されている基礎技術。

---

### <a id="paper-8"></a> [8] Deep Learning Approaches and Applications in Toxicologic Histopathology: Current Status and Future Perspectives
- **著者**: Shima Mehrvar, Lauren E. Himmel, Pradeep Babburi, et al.（AbbVie）
- **掲載**: Journal of Pathology Informatics, 12:42 (2021) / DOI:10.4103/jpi.jpi_36_21

#### 概要 (Abstract TL;DR)
毒性病理DLの概念と応用例を包括的にレビュー。**動物種横断の実証データとして最も具体的**: ラット組織スライド（1,690枚・46組織）で学習したモデルが非ヒト霊長類・ミニブタへの転移学習の基盤として機能した一方、**再学習なしの直接転移は失敗**し、新ドメインでの再訓練が必要だったと明記。

#### 毒性病理への示唆
「マウス肝臓で開発したモデルは、転移学習なしではサル肝臓で同等性能を発揮できない」という2021年時点の知見は、PROP-01のコア問い（ヒトFMは動物へどのようなドメインシフトを起こすか）に対する最も古典的かつ具体的な先行証拠。2025〜2026年の最新研究（Slootweg, Graf, Funk等）と比較しても、この根本課題は依然未解決であることが確認できる。

---

### <a id="paper-9"></a> [9] PathoTune: Adapting Visual Foundation Model to Pathological Specialists
- **掲載**: MICCAI 2024 / arXiv:2403.16497

#### 概要 (Abstract TL;DR)
Task-specific/Instance-specific な視覚・テキストプロンプトを用い、汎用視覚基盤モデルを病理特化タスクへ全体の5.9%のパラメータのみで適応させるマルチモーダルプロンプトチューニング手法。線形プロービングを大幅に上回り、フルファインチューニングに近い性能を達成。

#### 毒性病理への示唆
少量の動物病理データでヒト病理FMを効率的に適応させる際の直接的な技術的基盤（LoRA等と並ぶ選択肢）。動物種特異的プロンプトを設計すれば、種横断アライメントのプラグイン的手法になり得る。

---

### <a id="paper-10"></a> [10] Digital Pathology and Artificial Intelligence Applied to Nonclinical Toxicology Pathology—The Current State, Challenges, and Future Directions
- **著者**: Gabriele Pohlmeyer-Esch, Charles Halsey, Julie Boisclair, et al.（複数製薬企業共著）
- **掲載**: Toxicologic Pathology, 53(6):516-535 (2025) / DOI:10.1177/01926233251340622

#### 概要 (Abstract TL;DR)
非臨床毒性病理におけるデジタルパソロジー・AIの現状を包括的にレビュー。2025年1月のFDA AI規制ドラフトガイダンス、IMI Bigpictureプロジェクト（複数組織源からのWSIリポジトリ構想）、ISPE AI成熟度モデル（GxP環境での6段階自動化レベル）等、規制・標準化動向を整理。

#### 毒性病理への示唆
動物種横断基盤モデルの実運用化には、本論文が指摘する規制ガイダンス不足・グラウンドトゥルース生成課題の解決が前提条件となる。PROP-09（GLPバリデーション）と直接接続。

---

### <a id="paper-11"></a> [11] Reporting transparency in veterinary pathology deep learning: A systematic review of reproducibility-critical details
- **著者**: Sweta Banerjee, Christof A. Bertram, Viktoria Weiss, et al.
- **掲載**: Veterinary Pathology (2026) / DOI:10.1177/03009858261459452

#### 概要 (Abstract TL;DR)
獣医病理DL研究のPRISMA準拠システマティックレビュー。コード公開率わずか3%、augmentation未報告56%、主要ハイパーパラメータ欠落40%など、再現性に関わる報告の欠如を定量化。

#### 毒性病理への示唆
動物種横断病理基盤モデルの研究分野全体が、まだ再現性・標準化の基盤整備段階にあることを示す。今後の調査・PoCでは、本論文が提案する5次元の報告基準（研究設計、データ透明性、データリーク管理、モデル・学習詳細、性能評価）に沿った設計・記録が望ましい。
