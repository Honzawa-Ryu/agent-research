# 収集論文・リソース一覧

本調査で参照・精読した論文および関連リソースの書誌情報と要約です。

---

## 📚 論文メタデータ一覧

| No. | タイトル | 著者 / 年 / 会議 | リンク (arXiv/DOI) | Code | ローカルPDF | 重要度 |
|:---|:---|:---|:---|:---|:---|:---:|
| 1 | [TRACE: Deep Learning-based Modeling for Preclinical Drug Safety Assessment](#paper-1) | Jaume et al. (2024) | [bioRxiv](https://www.biorxiv.org/content/10.1101/2024.07.20.604430) / [PMC11291027](https://pmc.ncbi.nlm.nih.gov/articles/PMC11291027/) | - | - | ★★★ |
| 2 | [Toxicity Assessment in Preclinical Histopathology via Class-Aware Mahalanobis Distance for Known and Novel Anomalies](#paper-2) | Graf et al. (2026) | [arXiv:2602.02124](https://arxiv.org/abs/2602.02124) | - | [PDF](pdfs/2026_Graf_DINOv2LoRA_ToxPath.pdf) | ★★★ |
| 3 | [Data-efficient and weakly supervised computational pathology on whole-slide images (CLAM)](#paper-3) | Lu et al. (2021) | [arXiv:2004.09666](https://arxiv.org/abs/2004.09666) | [GitHub](https://github.com/mahmoodlab/CLAM) | - | ★★☆ |
| 4 | [TransMIL: Transformer based Correlated Multiple Instance Learning for WSI Classification](#paper-4) | Shao et al. (2021) | [arXiv:2106.00908](https://arxiv.org/abs/2106.00908) | [GitHub](https://github.com/szc19990412/TransMIL) | - | ★★☆ |
| 5 | [H²-MIL: Exploring Hierarchical Representation with Heterogeneous Multiple Instance Learning for WSI Analysis](#paper-5) | Hou et al. (2022) | [AAAI 2022](https://ojs.aaai.org/index.php/AAAI/article/view/19976) | [GitHub](https://github.com/lin-lcx/H2-MIL) | [PDF](pdfs/2022_Hou_H2MIL.pdf) | ★★★ |
| 6 | [A Graph-Based Multi-Scale Approach With Knowledge Distillation for WSI Classification](#paper-6) | Bontempo et al. (2023) | IEEE Transactions on Medical Imaging | - | - | ★★☆ |
| 7 | [Predicting Dose-Response Curves with Deep Neural Networks](#paper-7) | Alonso-Campana, Prasse & Scheffer (2024) | [ICML 2024 / PMLR v235](https://proceedings.mlr.press/v235/alonso-campana24a.html) | - | - | ★★★ |
| 8 | [Rank Consistent Ordinal Regression for Neural Networks with Application to Age Estimation (CORAL)](#paper-8) | Cao, Mirjalili & Raschka (2019) | [arXiv:1901.07884](https://arxiv.org/abs/1901.07884) | [GitHub](https://github.com/Raschka-research-group/coral-cnn) | [PDF](pdfs/2019_Cao_CORAL_OrdinalRegression.pdf) | ★★★ |
| 9 | [Constrained Monotonic Neural Networks](#paper-9) | Runje & Shankaranarayana (2023) | [ICML 2023 / PMLR v202](https://proceedings.mlr.press/v202/runje23a.html) | - | [PDF](pdfs/2023_Runje_ConstrainedMonotonicNN.pdf) | ★★☆ |
| 10 | [Unconstrained Monotonic Neural Networks (UMNN)](#paper-10) | Wehenkel & Louppe (2019) | [NeurIPS 2019](https://papers.neurips.cc/paper/8433-unconstrained-monotonic-neural-networks) | [GitHub](https://github.com/AWehenkel/UMNN) | [PDF](pdfs/2019_Wehenkel_UnconstrainedMonotonicNN.pdf) | ★★☆ |
| 11 | [Deep Learning Approaches and Applications in Toxicologic Histopathology: Current Status and Future Perspectives](#paper-11) | Mehrvar et al. (2021) | Journal of Pathology Informatics / [PMC8609289](https://pmc.ncbi.nlm.nih.gov/articles/PMC8609289/) | - | - | ★★☆ |

---

## 📝 論文詳細サマリー

### <a id="paper-1"></a> [1] TRACE: Deep Learning-based Modeling for Preclinical Drug Safety Assessment
- **著者**: Guillaume Jaume, Simone de Brot, Andrew H. Song, Drew F. K. Williamson, Lukas Oldenburg, Andrew Zhang, Richard J. Chen, Javier Asin, Sohvi Blatter, Martina Dettwiler, Christine Goepfert, Llorenç Grau-Roma, Sara Soto, Stefan M. Keller, Sven Rottenberg, Jorge del-Pozo, Rowland Pettit, Long Phi Le, Faisal Mahmood
- **所属 / 組織**: Mahmood Lab (Harvard Medical School / Brigham and Women's Hospital) を中心とした獣医病理学者との共同研究（多施設）
- **掲載**: bioRxiv preprint, 2024-07 (査読前)
- **リンク**: [bioRxiv:10.1101/2024.07.20.604430](https://www.biorxiv.org/content/10.1101/2024.07.20.604430) / [PMC11291027](https://pmc.ncbi.nlm.nih.gov/articles/PMC11291027/)

#### 概要 (Abstract TL;DR)
- 157試験・46,734枚のラット肝臓/腎臓WSIから抽出した1,500万パッチでiBOT自己教師あり学習した ViT-B (86M) 基盤モデル「TRACE」を構築し、病変検出・重症度スコアリング・形態検索・**自動用量反応特性評価**を実現。10名の獣医病理専門医とのコンセンサスを上回る一致率を達成。

#### 手法のポイント
- **アーキテクチャ / 入出力**: ViT-B/16、256×256パッチ（224×224にリサイズ）、iBOT（対照的自己蒸留＋Masked Image Modeling）で事前学習。下流タスクは5層MLP＋ゲート付きアテンションの AttnPatchMIL、および12病変マルチラベル二値分類でファインチューニングした TRACE(FT)。
- **用量反応の実装方法**: 各用量×時間ポイント群（1群5匹）についてTRACE(FT)のパッチ分類結果を病変領域の面積百分率として集計し、**群内で単純平均**。対照群の同一背景病変で正規化した上で log2 fold change として用量・時間の経過を表現。単調性制約や曲線フィッティング（Hill式等）は使用していない。
- **階層構造の扱い**: パッチ→スライド→試験（群）の3段階は扱うが、個体レベルの明示的なノードや、群間の統計的検定（有意差検定・単調性検定）は論文中に記述なし。
- **使用データセット**: 社内前臨床試験データ（非公開）。

#### 結果・貢献
- 従来の5段階カテゴリカルseverityスコアリング（Minimal〜Severe）を、パッチ単位の**定量的%表現**に置き換えることで再現性・客観性を向上。
- **本調査への示唆**: 「群レベル用量反応の自動特性評価」を謳う現状最先端の実運用モデルでも、実装は「集計＋対数比較」という統計的に素朴な後処理に留まり、PROP-02が構想する数理的に洗練された階層MIL＋単調性正則化は未実装。ここに明確な研究機会がある。

---

### <a id="paper-2"></a> [2] Toxicity Assessment in Preclinical Histopathology via Class-Aware Mahalanobis Distance for Known and Novel Anomalies
- **著者**: Olga Graf, Dhrupal Patel, Peter Groß, Charlotte Lempp, Matthias Hein, Fabian Heinemann
- **所属 / 組織**: Tübingen AI Center (University of Tübingen), Boehringer Ingelheim Pharma GmbH
- **掲載**: arXiv preprint, 2026-06（Scientific Reports 掲載も確認）
- **リンク**: [arXiv:2602.02124](https://arxiv.org/abs/2602.02124)
- **PDF**: [pdfs/2026_Graf_DINOv2LoRA_ToxPath.pdf](pdfs/2026_Graf_DINOv2LoRA_ToxPath.pdf)

#### 概要 (Abstract TL;DR)
- 44試験・742枚のラット肝臓WSI（H&E）でDINOv2 ViT-B/14をLoRA（rank=3）でパッチセグメンテーションにファインチューニングし、クラス別閾値のMahalanobis距離でOOD（未知病変）検知を実施。既知/新規異常を高精度（誤陰性率0.16%）で識別。

#### 手法のポイント
- **アーキテクチャ / 入出力**: DINOv2 (ViT-B/14, d=768) をバックボーンとして完全凍結し、LoRAモジュール（rank 1〜16、最適はrank 3）のみをAttention層に適用。672×672pxタイルの中心252×252pxを処理。推論時は36回のスライディングウィンドウ平均でロバスト化。
- **用量反応の扱い**: Figure 8で対照群/低/中/高用量群間の異常割合を比較し、化合物誘発性の用量依存的な肝細胞質空胞化を確認。**ただし統計的な用量反応分析（曲線フィッティング・有意差検定）は実施しておらず、個体間相関や群構造もモデルに明示的に組み込まれていない**、と論文中で明言。
- **階層構造の扱い**: 試験→個体→WSI→パッチの4階層のデータ構造を持つが、モデル自体は分布外検知（異常検知）に主眼があり、群レベル集約・用量反応モデリングは範囲外。

#### 結果・貢献
- 既知病変クラスの分類と未知病変（OOD）の同時検知を高精度で両立。産業実運用（Boehringer Ingelheim）を志向した現実的なデータ規模（742 WSI）でのバリデーション。
- **本調査への示唆**: 2026年時点の製薬企業発の最先端実装でも「用量依存性の存在は確認するが、それを深層学習内部で定量的にモデル化する」段階には至っていないことの直接的な一次証拠。PROP-01調査でも同論文が「ヒト病理基盤モデル非依存でImageNet/DINOv2ベース」の実例として参照されており、本調査と合わせて2つの角度から同じギャップを裏付ける。

---

### <a id="paper-3"></a> [3] CLAM: Data-efficient and weakly supervised computational pathology on whole-slide images
- **著者**: Ming Y. Lu, Drew F. K. Williamson, Tiffany Y. Chen, Richard J. Chen, Matteo Barbieri, Faisal Mahmood
- **掲載**: Nature Biomedical Engineering, 2021
- **リンク**: [arXiv:2004.09666](https://arxiv.org/abs/2004.09666)

#### 概要
- スライドレベルのラベルのみからアテンションベースでパッチの重要度を学習する弱教師ありMILの代表的手法。クラスタリングによるインスタンスレベル正則化も導入。PROP-02が拡張すべき「1スライド=1バッグ」MILのベースライン。

---

### <a id="paper-4"></a> [4] TransMIL: Transformer based Correlated Multiple Instance Learning for WSI Classification
- **著者**: Zhuchen Shao, Hao Bian, Yang Chen, Yifeng Wang, Jian Zhang, Xiangyang Ji, Yongbing Zhang
- **掲載**: NeurIPS 2021
- **リンク**: [arXiv:2106.00908](https://arxiv.org/abs/2106.00908)

#### 概要
- パッチ間の空間的相関をTransformerのSelf-Attentionで明示的にモデル化するMIL。CLAMがインスタンス独立を仮定するのに対し、TransMILはインスタンス間相関を扱える点でPROP-02の「臓器内スライド間相関」拡張のベースとして参考になる。

---

### <a id="paper-5"></a> [5] H²-MIL: Exploring Hierarchical Representation with Heterogeneous Multiple Instance Learning for WSI Analysis
- **著者**: Wentai Hou, Lequan Yu, Chengxuan Lin, Helong Huang, Rongshan Yu, Jing Qin, Liansheng Wang
- **掲載**: AAAI 2022, 36(1), 933–941
- **リンク**: [AAAI Proceedings](https://ojs.aaai.org/index.php/AAAI/article/view/19976)
- **PDF**: [pdfs/2022_Hou_H2MIL.pdf](pdfs/2022_Hou_H2MIL.pdf)

#### 概要 (Abstract TL;DR)
- WSIを異なる解像度のヘテロ型グラフとして表現し、GNNでメッセージパッシングすることで解剖学的スケール（パッチ→領域→スライド）の階層表現を学習する代表的な階層型MIL。

#### 手法のポイント
- **アーキテクチャ**: 解像度ピラミッド構造をヘテロジニアスグラフとしてエンコードし、動的にプーリング範囲を学習するGNN。
- **本調査への示唆**: 「階層」という語はPROP-02（実験デザイン階層：群→個体→臓器→スライド→パッチ）と共通するが、H²-MILが扱う階層は**画像の解剖学的スケール階層**であり、性質が異なる。両者を混同せず、H²-MILのグラフ構築・プーリング機構を「実験デザイン階層」に転用する場合は、ノード種別を病理解剖階層ではなく試験デザイン階層（群/個体/臓器/スライド）に置き換える設計変更が必要になる。

---

### <a id="paper-6"></a> [6] A Graph-Based Multi-Scale Approach With Knowledge Distillation for WSI Classification
- **著者**: Gianpaolo Bontempo, Federico Bolelli, Angelo Porrello, Simone Calderara, Elisa Ficarra
- **掲載**: IEEE Transactions on Medical Imaging, 2023
- **リンク**: [PubMed:38015690](https://pubmed.ncbi.nlm.nih.gov/38015690/)

#### 概要
- 異なる倍率（マグニフィケーション）の2つのパッチグラフ間で知識蒸留を行い、高解像度と低解像度の情報を階層的・双方向にメッセージパッシングする手法。マルチスケール統合の設計パターンとして、群-個体間の情報伝播アーキテクチャ設計の参考になる。

---

### <a id="paper-7"></a> [7] Predicting Dose-Response Curves with Deep Neural Networks
- **著者**: Pedro Alonso Campana, Paul Prasse, Tobias Scheffer
- **所属**: University of Potsdam
- **掲載**: ICML 2024 / PMLR Volume 235
- **リンク**: [PMLR](https://proceedings.mlr.press/v235/alonso-campana24a.html) / [OpenReview](https://openreview.net/forum?id=MDAg5Q7IsI)

#### 概要 (Abstract TL;DR)
- 薬剤分子と組織トランスクリプトームの相互作用埋め込みを用いて、スカラー指標（IC50等）ではなく**用量反応曲線全体**をニューラルネットで直接推定する手法。従来のHill方程式は単調・対称という強い仮定を置くため、多峰性・二相性の化合物（低用量で刺激、高用量で毒性等）を表現できない限界を指摘。

#### 手法のポイント
- **アーキテクチャ / 入出力**: 薬剤×組織トランスクリプトームの埋め込みから用量反応曲線を直接出力するニューラルモデル（生化学的知識をアーキテクチャに反映）。
- **本調査への示唆**: 病理画像を入力としない（トランスクリプトームベース）点でPROP-02とは異なるが、「Hill方程式による単調・対称の仮定が生物学的実態を歪める」という指摘は、PROP-02が提案する単調増加性正則化損失（Hill式フィッティング）の設計そのものに再考を促す重要な知見。毒性病理でも二相性用量反応（低用量ホルミシス・高用量毒性）が起こりうるため、硬い単調性制約は逆にモデルの表現力を損なう可能性がある。

---

### <a id="paper-8"></a> [8] Rank Consistent Ordinal Regression for Neural Networks with Application to Age Estimation (CORAL)
- **著者**: Wenzhi Cao, Vahid Mirjalili, Sebastian Raschka
- **掲載**: Pattern Recognition Letters, 2020（arXiv初出2019）
- **リンク**: [arXiv:1901.07884](https://arxiv.org/abs/1901.07884)
- **PDF**: [pdfs/2019_Cao_CORAL_OrdinalRegression.pdf](pdfs/2019_Cao_CORAL_OrdinalRegression.pdf)

#### 概要 (Abstract TL;DR)
- 順序回帰（Ordinal Regression）をK-1個の二値分類タスクに分解し、重み共有と順序付きバイアス項によって予測確率が単調非増加になることを理論的に保証する枠組み（CORAL）。

#### 手法のポイント
- **アーキテクチャ / 損失関数**: 共有重み＋クラスごとに異なるバイアス項を持つK-1個の二値分類器。ランク一貫性（Rank-monotonicity）を数学的に保証。
- **本調査への示唆**: INHAND重症度グレード（Minimal〜Severe）の順序性をモデル出力に保証する仕組みとして直接応用可能。さらに「用量群」というカテゴリ変数を順序変数として扱えば、CORALの枠組みを「用量群→重症度」の単調性制約に転用できる可能性がある（オープンクエスチョンとして報告書に記載）。

---

### <a id="paper-9"></a> [9] Constrained Monotonic Neural Networks
- **著者**: Davor Runje, Sharath M. Shankaranarayana
- **掲載**: ICML 2023 / PMLR Volume 202
- **リンク**: [PMLR](https://proceedings.mlr.press/v202/runje23a.html)
- **PDF**: [pdfs/2023_Runje_ConstrainedMonotonicNN.pdf](pdfs/2023_Runje_ConstrainedMonotonicNN.pdf)

#### 概要
- 重み符号制約と単調な活性化関数の組み合わせで単調性を保証しつつ、非凸関数も近似可能にする構成的手法。Deep Lattice Networks等の先行手法より表現力が高い。

#### 本調査への示唆
- MILのプーリング層（バッグレベル集約関数）自体を本手法で構成すれば、「用量群インデックスの増加に対しバッグレベルスコアが単調非減少」という制約をアーキテクチャレベルで保証でき、PROP-02の損失関数ベースの正則化（ソフト制約）よりも強い保証（ハード制約）を与える代替設計になり得る。

---

### <a id="paper-10"></a> [10] Unconstrained Monotonic Neural Networks (UMNN)
- **著者**: Antoine Wehenkel, Gilles Louppe
- **所属**: University of Liège
- **掲載**: NeurIPS 2019
- **リンク**: [NeurIPS Proceedings](https://papers.neurips.cc/paper/8433-unconstrained-monotonic-neural-networks)
- **PDF**: [pdfs/2019_Wehenkel_UnconstrainedMonotonicNN.pdf](pdfs/2019_Wehenkel_UnconstrainedMonotonicNN.pdf)

#### 概要
- 「導関数が正であれば関数は単調」という性質を利用し、自由な形のニューラルネットの出力の正値性のみを制約することで単調性を保証。重み符号制約方式より高い表現力を持つ。

#### 本調査への示唆
- Runje et al. 2023（ハード制約・軽量）とUMNN（高表現力・積分計算コスト大）はトレードオフの両極。用量反応曲線の形状（Hill式のような滑らかな単調曲線 vs 複雑な非線形単調曲線）に応じてどちらを採用すべきか、実装検討時の比較軸になる。

---

### <a id="paper-11"></a> [11] Deep Learning Approaches and Applications in Toxicologic Histopathology: Current Status and Future Perspectives
- **著者**: Shima Mehrvar, Lauren E. Himmel, Pradeep Babburi, Andrew L. Goldberg, Magali Guffroy, Kyathanahalli Janardhan, Amanda L. Krempley, Bhupinder Bawa
- **所属**: AbbVie
- **掲載**: Journal of Pathology Informatics, 2021
- **リンク**: [PMC8609289](https://pmc.ncbi.nlm.nih.gov/articles/PMC8609289/)

#### 概要
- 毒性病理領域における深層学習応用の総説。デジタル病理ワークフロー構築、モデル検証、GLP規制環境での適用可能性、染色バリエーション対応を扱う。用量反応性や階層構造モデリングへの直接言及はないが、業界の実運用課題（規制対応・検証プロトコル）の背景理解として有用。PROP-09（GLPバリデーション）とも関連。
