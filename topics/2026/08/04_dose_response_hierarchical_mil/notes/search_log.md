# 調査メモ・検索ログ (Search & Investigation Log)

本調査における検索クエリ、情報ソース、選定・除外基準、思考ログを記録します。

対象テーマ: [PROP-02] 毒性試験における用量反応性（Dose-Response）モデリングと階層的弱教師あり学習（Hierarchical Group-MIL）の数理・実装設計

---

## 🔍 検索クエリ履歴

| 日時 | ソース | 検索クエリ | ヒット件数（概算） | 採用件数 | 備考 |
|:---|:---|:---|:---|:---|:---|
| 2026-08-19 | Web検索 | `Pfizer TRACE toxicologic pathology deep learning dose response rat liver` | 9 | 1 | TRACE (Jaume et al. 2024) を特定。bioRxiv版とPMC版で書誌情報の表記ゆれあり |
| 2026-08-19 | PMC (フルテキスト) | PMC11291027 フルテキスト精読 | - | 1 | TRACEの用量反応特性評価・階層構造・severityスコアリング手法を抽出 |
| 2026-08-19 | Web検索 | `hierarchical multiple instance learning whole slide image group-level dose response` | 8 | 0（直接該当なし） | 「群レベル用量反応」に直接言及するMIL論文は不在。階層MIL自体の主要手法（H²-MIL等）を収集 |
| 2026-08-19 | Web検索 | `monotonicity constraint loss function deep learning ordinal regression dose response` | 9 | 2 | CORAL (Cao et al. 2019), SLACE (AAAI) を発見。用量反応特化の単調性損失は未発見 |
| 2026-08-19 | Web検索 | `benchmark dose modeling neural network NOAEL automated toxicology deep learning` | 9 | 0（背景理解用） | BMD/NOAELの標準的定義・限界を確認（Web検索要約のみ、DL直結の実装は未発見） |
| 2026-08-19 | Web検索 | `graph neural network multiple instance learning whole slide image patch-level hierarchical 2023 2024` | 10 | 2 | H²-MIL (Hou et al. 2022), Bontempo et al. 2023 (multi-scale graph KD) を採用候補に |
| 2026-08-19 | Web検索 | `monotonic neural network constrained lattice network unconstrained monotonic` | 7 | 2 | Runje et al. 2023 (Constrained Monotonic NN), Wehenkel & Louppe 2019 (UMNN) を特定 |
| 2026-08-19 | Web検索 | `Hill equation dose-response curve fitting neural network drug toxicity deep learning` | 10 | 1 | Alonso-Campana et al. 2024 (ICML) — Hill方程式の限界とニューラル代替を提示。WSI/病理特化ではないが用量反応曲線モデリングの参照として採用 |
| 2026-08-19 | Web検索 | `class-aware Mahalanobis distance preclinical histopathology anomaly detection Scientific Reports 2026` | 10 | 1 | Graf et al. 2026 (Boehringer Ingelheim) を特定。PROP-01調査でも参照された論文と同一 |
| 2026-08-19 | WebFetch | arXiv:2602.02124v2 フルテキスト精読 | - | - | Figure 8で用量群間比較はあるが、統計的用量反応分析・個体間相関のモデル化は未実施と明記されていることを確認（本調査の核心的ギャップの直接証拠） |
| 2026-08-19 | WebFetch | proceedings.mlr.press/v235 (ICML 2024) アブストラクト精読 | - | - | 著者・会議情報の確認 |
| 2026-08-19 | Web検索 | `Bontempo 2023 knowledge distillation multi-resolution graph whole slide image classification title` | 10 | 1 | 正式タイトル・掲載誌（IEEE TMI）を確認 |
| 2026-08-19 | WebFetch | PMC8609289 フルテキスト精読 | - | 1 | Mehrvar et al. 2021（毒性病理DLレビュー）を背景文献として採用。用量反応の直接記述はなし |
| 2026-08-19 | Web検索 | `monotonicity constraint multiple instance learning whole slide image dose group toxicology pathology` | 8 | 0（ギャップ確認） | 「単調性制約×MIL×用量群毒性病理」を直接組み合わせた先行研究は本調査時点で確認できず。オープンクエスチョンとして報告書に記載 |

---

## 🎯 論文の選定・除外基準
- **採用基準**:
  - 実際にWebSearch/WebFetchで内容を確認できたもの（書誌情報のみのハルシネーションは排除）
  - 毒性病理特化の実運用モデル（TRACE, Graf et al. 2026等）、階層/グラフMILの基盤技術、単調性制約付きニューラルネットの主要理論のいずれかに該当するもの
  - オープンアクセスでPDF取得可能なものは`papers/pdfs/`に保存
- **除外基準**:
  - 検索結果のタイトルのみでフルテキスト未確認のもの
  - 「用量反応」「毒性病理」に直接関係しない一般的なMIL/オンコロジー論文（比較参照が必要な場合を除き本文引用は最小限に）

---

## 💡 調査中の思考メモ・ブレインストーミング
- **最大の発見**: TRACE（Jaume et al. 2024, Pfizer/Roche系共同研究)は「自動用量反応特性評価」を謳うが、実装は「各群のAttnPatchMIL定量化スコアを平均し、対照群でlog2 fold change正規化」という比較的シンプルな統計的後処理であり、PROP-02が問う「5階層構造のエンドツーエンド最適化」「単調増加性正則化損失によるHill式適合」のような数理的洗練は実装されていない。NOAEL/BMDへの言及も本文中に一切なし。
- Graf et al. 2026 (Boehringer Ingelheim, arXiv:2602.02124) も同様に、Figure 8で用量群間の異常割合を目視比較しているのみで、統計的な用量反応モデリングや個体間相関のモデル化は明示的に「実施していない」と記載。→ 業界の実運用最前線でも、この数理的ギャップは埋まっていないことの直接証拠。
- 「階層MIL」自体（H²-MIL, Bontempo 2023等）は疾患病理領域で成熟しつつあるが、いずれも「解剖学的スケール階層（パッチ→領域→スライド）」であり、PROP-02が求める「群→個体→臓器→スライド→パッチ」という**実験デザイン階層**を扱うものは皆無。両者の階層の性質は異なる点に注意（考察に明記）。
- 単調性制約付きニューラルネット（CORAL, Runje 2023, UMNN等）は理論的に成熟しているが、いずれも「1つの連続的入力（年齢・用量濃度等）に対する単調な出力」を保証する枠組みであり、「MILのバッグレベル集約に単調性を課す」実装は本調査では発見できず。既存の単調性ニューラルネット理論をMILのプーリング関数に応用するのが具体的な実装ルートになり得る（report.mdの技術提案に反映）。
- Hill方程式ベースの用量反応曲線フィッティングをニューラルネットに置き換える試み（Alonso-Campana 2024）は創薬スクリーニング（細胞株×薬剤のIC50予測）分野では存在するが、病理画像を入力とする研究は未発見。
