# 論文インデックス

本調査（PROP-05: INHAND準拠 Tox-VLM & Concept Bottleneck Model）で厳選した10論文の書誌情報・要約。

---

### 1. International Harmonization of Toxicologic Pathology Nomenclature
- **著者**: Mann, P. C., Vahle, J., Keenan, C. M., et al.
- **出典**: *Toxicologic Pathology*, 40(4S), 7S–13S (2012)
- **リンク**: https://journals.sagepub.com/doi/full/10.1177/0192623312438738
- **概要**: INHAND（International Harmonization of Nomenclature and Diagnostic Criteria for Lesions in Rats and Mice）プロジェクトの基盤論文。ラット・マウスの病変を臓器系統ごとに国際統一用語・定義文・重症度基準で標準化する枠組みを提示。本調査における「概念（Concept）」の一次ソースとなる統制語彙の出典。

### 2. A visual-language foundation model for computational pathology (CONCH)
- **著者**: Lu, M. Y., Chen, B., Williamson, D. F. K., et al.
- **出典**: *Nature Medicine*, 30, 863–874 (2024) / arXiv:2307.12914
- **リンク**: https://www.nature.com/articles/s41591-024-02856-4
- **概要**: 117万件の画像-キャプションペアで対照学習（CoCa方式）されたヒト病理VLM。BRCAサブタイピングZero-shot精度91.3%でPLIP(50.7%)を大幅に上回る。INHAND定義文のような構造化テキストとの対照学習レシピの土台候補。

### 3. A visual–language foundation model for pathology image analysis using medical Twitter (PLIP)
- **著者**: Huang, Z., Bianchi, F., Yuksekgonul, M., Montine, T. J., Zou, J.
- **出典**: *Nature Medicine*, 29, 2307–2316 (2023)
- **概要**: CLIPをMedical Twitterの20万超の病理画像-キャプションペアで再学習した先駆的病理VLM。CONCHとの性能比較のベースラインとして参照。データソースの脆弱性（SNS由来の非構造化キャプション）がINHAND準拠テキストとの対比点。

### 4. Quilt-1M: One Million Image-Text Pairs for Histopathology
- **著者**: Ikezogwo, W. O., Seyfioglu, M. S., Ghezloo, F., et al.
- **出典**: NeurIPS 2023 (Datasets and Benchmarks Track, Oral) / arXiv:2306.11207
- **リンク**: https://arxiv.org/abs/2306.11207
- **概要**: YouTube教育動画の音声認識・LLM・ハンドクラフトルールを組み合わせ約100万件の病理画像-テキストペアを自動構築。INHAND定義文のような専門文書が乏しい毒性病理領域で、同種のマルチモーダルデータ自動構築パイプラインが参考になる。

### 5. Label-free Concept Based Multiple Instance Learning for Gigapixel Histopathology
- **著者**: Sun, S., Tessier, L., Meeuwsen, F., Grisi, C., van Midden, D., Litjens, G., Baumgartner, C. F.
- **出典**: arXiv:2501.02922 (2025年1月)
- **リンク**: https://arxiv.org/abs/2501.02922
- **概要**: 人手による概念アノテーションなしに、VLMでパッチから病理概念を予測し、WSIレベル予測を「Top-Kパッチの概念の線形結合」として構成する解釈可能MIL。Camelyon16/PANDAでAUC・Accuracy 0.9超。INHAND-CBM実装の最有力な技術的雛形。

### 6. Integrating Clinical Knowledge into Concept Bottleneck Models
- **著者**: Pang, W., Ke, X., Tsutsui, S., Wen, B.
- **出典**: MICCAI 2024, LNCS 15004 / arXiv:2407.06600
- **リンク**: https://arxiv.org/abs/2407.06600
- **概要**: データ駆動のみで学習したCBMは分布外画像で臨床的に不自然な概念相関を学習しうるという問題を指摘し、臨床知識（概念間の既知の関係）を明示的に統合してアライメントを改善する手法。INHAND鑑別診断基準（所見の共起パターン）をCBMに組み込む際の直接的な参考実装。

### 7. An Explainable Biomedical Foundation Model via Large-Scale Concept-Enhanced Vision-Language Pre-training (ConceptCLIP)
- **著者**: Nie, Y. ほか（計17名）
- **出典**: arXiv:2501.15579 (2025年1月)
- **リンク**: https://arxiv.org/abs/2501.15579
- **概要**: UMLS（Unified Medical Language System）由来の医療概念を用い、2,300万件の画像-テキスト-概念トリプレットで事前学習した説明可能生物医学基盤モデル。汎用医療オントロジー（UMLS）を概念源とする設計は、INHAND（動物病理特化オントロジー）を概念源に置き換える際の直接的なアーキテクチャ参照になる。

### 8. Deep Learning-based Modeling for Preclinical Drug Safety Assessment (TRACE)
- **著者**: Bhattacharya, S. ほか（Pfizer）
- **出典**: bioRxiv (2024) / PMC11291027
- **リンク**: https://pmc.ncbi.nlm.nih.gov/articles/PMC11291027/
- **概要**: ラット肝臓157試験・4.6万スライドで学習した現行最先端の毒性病理AI。病理医合意と高い一致度を達成する一方、説明性はアテンション/ヒートマップに留まり、概念ボトルネック層やINHAND用語への明示的な対応付けは実装されていない。「SOTAだが説明可能性は未解決」というギャップの実例。

### 9. A Weakly Supervised Deep Learning Framework for Whole Slide Classification to Facilitate Digital Pathology in Animal Study (PathologAI)
- **著者**: Bussola, N., Xu, J., Wu, L., Gorini, L., Zhang, Y., Furlanello, C., Tong, W.
- **出典**: *Chemical Research in Toxicology*, 36(8), 1321–1331 (2023)
- **リンク**: https://pubs.acs.org/doi/10.1021/acs.chemrestox.3c00058
- **概要**: Open TG-GATEsのラット肝WSI（816枚）を用いた弱教師あり壊死検出。BiGANタイル圧縮＋CNNアンサンブル。重症度・用量依存性の弁別も行うが、こちらも概念ベースの解釈性は持たない。

### 10. Natural language processing in veterinary pathology: A review
- **著者**: Stimmer, L., Kuiper, R. V., Polledo, L., Ressel, L., Monné Rodriguez, J. M., Veiga, I. B., Williams, J., Herder, V.
- **出典**: *Veterinary Pathology*, 62(6) (2025年6月30日オンライン公開、11月印刷)
- **リンク**: https://journals.sagepub.com/doi/10.1177/03009858251347529
- **概要**: 獣医病理へのNLP応用に関する2025年時点の最新レビュー。ヒト病理向けVLM（PathChatなど）には言及するが、INHAND用語体系・構造化診断オントロジー・Concept Bottleneck Modelへの言及は皆無。「獣医病理特化のNLP応用はほぼ未開拓」と明言しており、本提案テーマ（PROP-05）が真にホワイトスペースであることを裏付ける最有力の状況証拠。

---

## 論文選定から見えた全体傾向
- INHANDとVLM/CBMを直接組み合わせた研究は、2026年8月時点で一次文献上に存在しない（#10が最も強い反証的証拠）。
- 要素技術（#5 概念の無annotation予測、#6 臨床知識の概念統合、#7 医療オントロジー概念源の大規模事前学習）はいずれも2024〜2025年に成熟しており、INHANDというドメイン特化オントロジーへの置き換えは技術的には射程内。
- 現行の毒性病理SOTAモデル（#8 TRACE, #9 PathologAI）はどちらも精度面で高い到達点を示す一方、説明可能性はアテンションマップ止まりで、規制監査に耐える概念レベルの追跡可能性を提供していない。
