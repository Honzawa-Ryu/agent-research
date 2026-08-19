# 収集論文・リソース一覧

本調査で参照・精読した論文および関連リソースの書誌情報と要約です。対象テーマ: [PROP-08] 光顕WSIからの微小構造推定（Virtual Ultrastructure / Sub-resolution Toxicity Inference）の実現可能性調査

---

## 📚 論文メタデータ一覧

| No. | タイトル | 著者 / 年 / 会議 | リンク (arXiv/DOI) | Code | ローカルPDF | 重要度 |
|:---|:---|:---|:---|:---|:---|:---:|
| 1 | [Label-free prediction of 3D fluorescence images from transmitted-light microscopy](#paper-1) | Ounkomol et al. (2018), Nature Methods | [DOI](https://doi.org/10.1038/s41592-018-0111-2) | [GitHub](https://github.com/AllenCellModeling/pytorch_fnet) | - | ★★★ |
| 2 | [In Silico Labeling: Predicting Fluorescent Labels in Unlabeled Images](#paper-2) | Christiansen et al. (2018), Cell | [DOI](https://doi.org/10.1016/j.cell.2018.03.040) | [GitHub](https://github.com/google/in-silico-labeling) | - | ★★★ |
| 3 | [Virtual histological staining of unlabelled tissue-autofluorescence images via deep learning](#paper-3) | Rivenson et al. (2019), Nat. Biomed. Eng. | [DOI](https://doi.org/10.1038/s41551-019-0362-y) | - | - | ★★★ |
| 4 | [Digital staining in optical microscopy using deep learning — a review](#paper-4) | Kreiss et al. (2023), arXiv | [arXiv:2303.08140](https://arxiv.org/abs/2303.08140) | - | - | ★★☆ |
| 5 | [Pixel super-resolved virtual staining of label-free tissue using diffusion models](#paper-5) | Zhang, Huang, Pillar, Li, Chen, Ozcan (2025), arXiv | [arXiv:2410.20073](https://arxiv.org/abs/2410.20073) | - | - | ★★☆ |
| 6 | [Diffusion-based deep learning method for augmenting ultrastructural imaging and volume electron microscopy (EMDiffuse)](#paper-6) | Lu, Chen, Qiu, Chen, Chen, Qi, Jiang (2024), Nature Communications | [DOI](https://doi.org/10.1038/s41467-024-49125-z) | [GitHub](https://github.com/Luchixiang/EMDiffuse) | - | ★★★ |
| 7 | [DeepCLEM: automated registration for correlative light and electron microscopy using deep learning](#paper-7) | Seifert, Markert, Britz, Perschin, Erbacher, Stigloher, Kollmannsberger (2023), F1000Research 9:1275 | [DOI](https://doi.org/10.12688/f1000research.27158.3) | [GitHub](https://github.com/CIA-CCTB/Deep_CLEM) | - | ★★★ |
| 8 | [CLEM-Reg: an automated point cloud-based registration algorithm for volume correlative light and electron microscopy](#paper-8) | Krentzel et al. (2025), Nature Methods 22(9), 1923–1934 | [DOI](https://doi.org/10.1038/s41592-025-02794-0) | - | - | ★★☆ |
| 9 | [Predicting fluorescent labels in label-free microscopy images with pix2pix and adaptive loss in Light My Cells challenge](#paper-9) | Liu, Li, Wang, Fan, Xu, Oguz (2024), arXiv | [arXiv:2406.15716](https://arxiv.org/abs/2406.15716) | - | - | ★★☆ |
| 10 | [Drug-induced phospholipidosis](#paper-10) | Anderson & Borlak (2006), FEBS Letters 580(23), 5533–5540 | [DOI](https://doi.org/10.1016/j.febslet.2006.08.061) | - | - | ★★☆ |

---

## 📝 論文詳細サマリー

### <a id="paper-1"></a> [1] Label-free prediction of three-dimensional fluorescence images from transmitted-light microscopy
- **著者**: Chawin Ounkomol, Sharmishtaa Seshamani, Mary M. Maleckar, Forrest Collman, Gregory R. Johnson
- **所属**: Allen Institute for Cell Science
- **掲載**: Nature Methods 15, 917–920 (2018)
- **リンク**: [DOI:10.1038/s41592-018-0111-2](https://doi.org/10.1038/s41592-018-0111-2)
- **コード**: [pytorch_fnet (GitHub)](https://github.com/AllenCellModeling/pytorch_fnet)

#### 概要
U-Netベースの3D CNNにより、明視野（transmitted-light）zスタック画像から核・細胞膜・ミトコンドリア等の複数の細胞内構造の蛍光シグナルを予測する。Label-free predictionの基礎を築いた代表研究。

#### 手法のポイント
- **アーキテクチャ/入出力**: 3D U-Net、入力=明視野zスタック、出力=各構造の蛍光チャネル（構造ごとに個別モデル）
- **学習方法**: 教師あり回帰（実測蛍光画像を正解ラベルとするペア学習）
- **使用データセット**: Allen Institute for Cell Scienceの共焦点蛍光顕微鏡ペアデータ

#### 結果・貢献
- 複数の細胞内構造（核内小体、細胞膜、DNA、ミトコンドリア等）で定量的に有意な予測精度を達成。
- **本調査における位置づけ**: あくまで光顕モダリティ間（明視野→蛍光）の予測であり、EM相当の解像度・情報量には遠く及ばない。しかし「異なる撮像原理の画像から特定の生物構造情報を統計的に予測する」という枠組み自体はPROP-08の目標と同型。

---

### <a id="paper-2"></a> [2] In Silico Labeling: Predicting Fluorescent Labels in Unlabeled Images
- **著者**: Eric M. Christiansen, Samuel J. Yang, D. Michael Ando, Ashkan Javaherian, et al. (Google/CZ Biohub)
- **掲載**: Cell 173(3), 792–803 (2018)
- **リンク**: [DOI:10.1016/j.cell.2018.03.040](https://doi.org/10.1016/j.cell.2018.03.040)
- **コード**: [in-silico-labeling (GitHub)](https://github.com/google/in-silico-labeling)

#### 概要
明視野（bright-field/phase-contrast/DIC）のzスタック画像のみから、複数の蛍光ラベル（核、生死判定、細胞種別等）を同時に予測する深層学習手法。

#### 手法のポイント
- **入力の検証**: GitHub READMEを一次情報として確認した結果、入力は一貫して「transmitted light」モダリティ（明視野・位相差・DIC）のみであり、電子顕微鏡画像は一切使用されていないことを確認。（Web検索の自動要約に「EM画像からIF予測を行う」という誤った記載があったため、この一次情報確認により訂正した。）
- **学習方法**: マルチタスク学習、複数の蛍光ラベルを同時予測するネットワーク

#### 結果・貢献
- 未見の顕微鏡条件（Condition E: DIC）への汎化性能を実証。
- **本調査における位置づけ**: 論文1と並ぶLabel-free predictionの基礎研究だが、EMは対象外。

---

### <a id="paper-3"></a> [3] Virtual histological staining of unlabelled tissue-autofluorescence images via deep learning
- **著者**: Yair Rivenson, Hongda Wang, Zhensong Wei, Kevin de Haan, Yibo Zhang, Yichen Wu, Harun Günaydın, Jonathan E. Zuckerman, Thomas Chong, Anthony E. Sisk, Lindsey M. Westbrook, W. Dean Wallace, Aydogan Ozcan
- **掲載**: Nature Biomedical Engineering 3(6), 466–477 (2019)
- **リンク**: [DOI:10.1038/s41551-019-0362-y](https://doi.org/10.1038/s41551-019-0362-y)

#### 概要
無染色組織の自家蛍光（autofluorescence）画像1枚から、GANベースの深層学習でH&E相当の仮想染色画像を生成する。Virtual staining分野の代表的な基盤研究。

#### 手法のポイント
- **アーキテクチャ**: CNN（GAN学習）、入力=単一波長の自家蛍光画像、出力=H&E様式の明視野画像
- **学習方法**: 敵対的学習により実際のH&E染色画像分布に近づける

#### 結果・貢献
- 腎臓・肝臓・肺組織で病理医のブラインド評価により診断的に許容可能な品質を実証。
- **本調査における位置づけ**: 光顕モダリティ間（自家蛍光→H&E）の変換に留まり、EM解像度への拡張は範囲外。ただし染色プロセス自体を「学習された変換」として置き換える発想は、PROP-08が目指す変換（HE→EM相当情報）の思考の出発点となる。

---

### <a id="paper-4"></a> [4] Digital staining in optical microscopy using deep learning — a review
- **著者**: Lucas Kreiss, Shaowei Jiang, Xiang Li, Shiqi Xu, Kevin C. Zhou, Alexander Mühlberg, Kyung Chul Lee, Kanghyun Kim, Amey Chaware, Michael Ando, Laura Barisoni, Seung Ah Lee, Guoan Zheng, Kyle Lafata, Oliver Friedrich, Roarke Horstmeyer
- **掲載**: arXiv (2023)
- **リンク**: [arXiv:2303.08140](https://arxiv.org/abs/2303.08140)

#### 概要
光学顕微鏡内でのデジタル染色（virtual/digital staining）技術の包括的レビュー。一次情報（アブストラクト）で確認した限り、スコープは光学顕微鏡内の変換（無染色画像↔各種化学染色）に限定され、電子顕微鏡は扱われていない。

#### 本調査における位置づけ
「virtual staining」という用語が指す研究群の境界を確認するために参照。EMへの拡張はこのレビューの対象外であり、PROP-08が扱う領域が既存レビューのスコープ外にあることの裏付けとなる。

---

### <a id="paper-5"></a> [5] Pixel super-resolved virtual staining of label-free tissue using diffusion models
- **著者**: Yijie Zhang, Luzhe Huang, Nir Pillar, Yuzhu Li, Hanlong Chen, Aydogan Ozcan
- **掲載**: arXiv (2025)
- **リンク**: [arXiv:2410.20073](https://arxiv.org/abs/2410.20073)

#### 概要
拡散モデル（Brownian bridgeプロセス）を用いて、無染色組織から超解像度の仮想染色画像を生成。ラベルフリー肺組織で従来法比4〜5倍の超解像係数、空間帯域幅積16〜25倍を達成。

#### 本調査における位置づけ
光顕内での「超解像virtual staining」の最新到達点。ただし目標解像度はあくまで高品質な光学顕微鏡スケールであり、EM（サブミクロン〜ナノメートル）スケールへの外挿ではない。PROP-08のコア問い1（既存の医用画像超解像研究との関係）に対する直接的な比較対象。

---

### <a id="paper-6"></a> [6] Diffusion-based deep learning method for augmenting ultrastructural imaging and volume electron microscopy (EMDiffuse)
- **著者**: Chixiang Lu, Kai Chen, Heng Qiu, Xiaojun Chen, Gu Chen, Xiaojuan Qi, Haibo Jiang
- **掲載**: Nature Communications 15, 4677 (2024)
- **リンク**: [DOI:10.1038/s41467-024-49125-z](https://doi.org/10.1038/s41467-024-49125-z)
- **コード**: [EMDiffuse (GitHub)](https://github.com/Luchixiang/EMDiffuse)

#### 概要
拡散モデルベースのEM画質拡張スイート。EMDiffuse-n（デノイズ）、EMDiffuse-r（超解像、6.6nm→3.3nm画素）、vEMDiffuse-i/a（異方性ボリュームEMの等方化）の4モジュールから成る。

#### 手法のポイント
- **入出力**: 一貫して電子顕微鏡画像のみを扱う（PMC本文で確認、光顕入力は一切なし）
- **性能**: デノイズで撮像時間を約18倍、超解像で約36倍削減しつつ解像度を倍化。わずか300万画素相当の画像1組でドメイン転移（fine-tuning）が可能なほど汎化性が高い。
- **対象構造**: ミトコンドリア（クリステ含む）、小胞体、ゴルジ体、シナプス小胞など

#### 結果・貢献
- **本調査における最重要の否定的知見**: EMDiffuseは「既に取得済みのEM画像」を強化する技術であり、「光顕画像からEM相当の情報を生成する」技術ではない。この違いはPROP-08の実現可能性を評価する上で決定的：入力モダリティ間の情報量ギャップ（光顕の回折限界 vs EMのナノメートル分解能）を埋める研究は、EM画質強化研究とは全く別の課題であり、EMDiffuseの成功はPROP-08の実現可能性を直接には保証しない。

---

### <a id="paper-7"></a> [7] DeepCLEM: automated registration for correlative light and electron microscopy using deep learning
- **著者**: Rick Seifert, Sebastian M. Markert, Sebastian Britz, Veronika Perschin, Christoph Erbacher, Christian Stigloher, Philip Kollmannsberger
- **掲載**: F1000Research 9:1275 (初出2020年10月、Version 3: 2023年12月)
- **リンク**: [DOI:10.12688/f1000research.27158.3](https://doi.org/10.12688/f1000research.27158.3)
- **コード**: [Deep_CLEM (GitHub)](https://github.com/CIA-CCTB/Deep_CLEM)

#### 概要
CNN（ProjectionCARE、CSBDeepベースの2D U-Net）によりEM画像から「仮想的な蛍光クロマチンシグナル」を予測し、実測の蛍光画像と相関ベースで自動位置合わせするCLEM登録ワークフロー。

#### 手法のポイント
- **方向性**: 入力=EM画像、出力=予測蛍光シグナル（PROP-08が求める方向＝光顕→EMとは逆方向）
- **目的**: 高忠実度の画像生成ではなく、位置合わせを可能にする程度の粗い類似シグナル生成
- **学習データ**: *C. elegans*およびヒト皮膚組織、SIM+走査電子顕微鏡のペア（cryo-immobilization, freeze substitution, メタクリレート樹脂包埋, 100nm切片）

#### 結果・貢献
- **本調査における位置づけ**: 「EM→蛍光」という、PROP-08が求める方向（光顕→EM）とは逆方向の粗い予測に成功した例。この非対称性（情報量の少ない方向への予測は成功しているが、情報量を増やす方向＝光顕→EMは前例なし）は、PROP-08の技術的難度を評価する上で重要な参照点。

---

### <a id="paper-8"></a> [8] CLEM-Reg: an automated point cloud-based registration algorithm for volume correlative light and electron microscopy
- **著者**: Daniel Krentzel, Matouš Elphick, Marie-Charlotte Domart, Christopher J. Peddie, Romain F. Laine, Cameron Shand, Ricardo Henriques, Lucy M. Collinson, Martin L. Jones
- **掲載**: Nature Methods 22(9), 1923–1934 (2025)
- **リンク**: [DOI:10.1038/s41592-025-02794-0](https://doi.org/10.1038/s41592-025-02794-0)

#### 概要
両モダリティでミトコンドリアをセグメンテーションし、点群化した上で確率的点群登録により3D CLEMデータセットを自動位置合わせする手法。従来手動で約2時間かかっていた登録を数分に短縮、サブミクロン精度を達成。

#### 本調査における位置づけ
CLEM位置合わせのSOTAだが、これも画像生成ではなくセグメンテーション＋幾何学的登録のアプローチであり、PROP-08が求める「画素レベルでの微細構造の生成的予測」とは異なる問題設定。

---

### <a id="paper-9"></a> [9] Predicting fluorescent labels in label-free microscopy images with pix2pix and adaptive loss in Light My Cells challenge
- **著者**: Han Liu, Hao Li, Jiacheng Wang, Yubo Fan, Zhoubing Xu, Ipek Oguz
- **掲載**: arXiv (2024)
- **リンク**: [arXiv:2406.15716](https://arxiv.org/abs/2406.15716)

#### 概要
「Light My Cells」チャレンジ（明視野/位相差/DIC画像から核・ミトコンドリア・チューブリン・アクチンの蛍光ラベルを予測する国際コンペ）への提出手法。pix2pixベースで適応的損失関数を導入。

#### 手法のポイント
- **データセット規模**: Light My Cellsデータセットは約57,000枚の2D顕微鏡画像、8つの画像センター・30研究にまたがる（別途確認したScientific Data誌のデータセット論文情報より）
- **対象構造にミトコンドリアが含まれる**: PROP-08が対象とする毒性微小構造（ミトコンドリア変性）と直接関連する対象構造が、光顕からの予測対象として既に確立していることを示す

#### 本調査における位置づけ
ミトコンドリアという構造そのものは、光顕から蛍光レベルで予測する研究基盤（Light My Cells）が既に存在する。しかしこれは「ミトコンドリアの有無・位置」の予測であり、「ミトコンドリア内部のクリステ変性・膨化」といったEMレベルの微細形態変化の予測とは全く異なる粒度の課題である。

---

### <a id="paper-10"></a> [10] Drug-induced phospholipidosis
- **著者**: Nicholas Anderson, Jürgen Borlak
- **掲載**: FEBS Letters 580(23), 5533–5540 (2006)
- **リンク**: [DOI:10.1016/j.febslet.2006.08.061](https://doi.org/10.1016/j.febslet.2006.08.061)

#### 概要
薬剤性リン脂質症（drug-induced phospholipidosis）のメカニズムと評価法に関するレビュー。50種類以上のカチオン性両親媒性薬剤（CAD）がリスクを持つとされ、リソソーム内の同心円状ミエリン様構造（ラメラ体）がTEMによる形態学的診断の指標となることを説明。

#### 本調査における位置づけ
毒性病理において、光顕解像度を超える微小構造（リソソーム内のラメラ体）が実際の規制毒性評価・創薬安全性評価で意味を持つ具体例として引用。PROP-08が扱う問題設定（HE-WSIからのミトコンドリア/リソソーム変化推定）の産業的動機付けの裏付けとなる一次情報。
