# 検索ログ・思考メモ

## 調査テーマ
[PROP-06] MELLODDY型・製薬企業間連合学習コンソーシアムの毒性病理領域への応用可能性と実装障壁の調査

## 検索クエリと結果概要

| # | 検索クエリ | 使用ツール | ヒット件数（概算） | 選定基準・所感 |
|---|---|---|---|---|
| 1 | MELLODDY federated learning drug discovery consortium architecture | WebSearch | 9件 | MELLODDYの基本アーキテクチャ（Substra基盤、ブロックチェーン監査ログ、10社参加）を把握。QSARデータ限定であることを確認。 |
| 2 | federated learning whole slide image histopathology 2024 2025 | WebSearch | 10件 | FL×WSIのシステマティックレビュー（Schoenpflug et al. 2024）、HistoFS、TheOden、FedDP等を発見 |
| 3 | federated learning toxicologic pathology preclinical safety assessment | WebSearch | 8件 | 「FL×毒性病理」の直接文献は不在。Effirisハッカソン論文（毒性学×FLだが画像でない）を発見 |
| 4 | secure aggregation homomorphic encryption pathology federated learning computational cost | WebSearch | 8件 | HEの計算コストが極めて高い旨の一般的知見を確認。具体的数値は後続クエリで深掘り |
| 5 | NVIDIA FLARE digital pathology federated learning deployment | WebSearch | 8件 | Roche Digital PathologyとNVIDIAの内部シミュレーション事例、FLフレームワーク比較論文を発見 |
| 6 | HistoFS federated style transfer non-IID whole slide image CVPR 2025 | WebSearch | 10件 | HistoFSの手法詳細（Pseudo bag styles, Authenticity Module）を確認。RQ2への直接回答として選定 |
| 7 | TheOden federated learning histopathology whole slide imaging firewall Germany | WebSearch | 7件 | TheOdenのリバースプロキシ方式・独3大学病院実証の詳細を確認 |
| 8 | Effiris federated learning computational toxicology pharma hackathon Lhasa | WebSearch | 7件 | Effirisハッカソン（Roche含む7社参加）の詳細、FLuIDフレームワークの存在を確認 |
| 9 | Owkin Substra open source federated learning platform origin MELLODDY pathology | WebSearch | 9件 | SubstraがMELLODDYの技術基盤でありLinux Foundation配下でOSS化されたこと、Owkinが臨床病理領域でもFLを実証済みであることを確認 |
| 10 | differential privacy accuracy tradeoff medical imaging federated learning rare disease data scarcity | WebSearch | 10件 | DPの精度低下トレードオフ（ε≈1で大幅低下、ε≈10で許容範囲）に関する一般知見を収集 |
| 11 | eTOX IMI consortium toxicology database pharma companies data sharing precedent | WebSearch | 7件 | eTOXプロジェクト（13社、中央集約型、オネストブローカー方式）を発見。MELLODDYとの対比材料として重要 |
| 12 | "secure aggregation" OR "homomorphic encryption" federated learning overhead "15621" OR per-client computational cost multiplier | WebSearch | 10件 | 「15621倍」の一次情報源を特定できず、代わりにHybrid HE論文（Correia et al. 2025）の確定値を発見・採用 |
| 13 | MELLODDY encryption scheme technical details gradient aggregation homomorphic differential privacy which one actually used | WebSearch | 複数ラウンド | MELLODDYの主機構がSecure Aggregation（SMPCペアワイズマスキング）であり準同型暗号ではないことを確認。topics/01の記述との齟齬を発見（重要） |

## WebFetchで全文確認したソース
- https://arxiv.org/abs/2210.08871 （MELLODDYアーキテクチャ論文）— アブストラクト取得、参加企業数10社を確認
- https://pmc.ncbi.nlm.nih.gov/articles/PMC11584763/ （Schoenpflug et al. 2024 FLレビュー）— 全文からデータ異質性対策・DP性能低下率（0.2-6%）・前臨床言及の不在を確認
- https://pmc.ncbi.nlm.nih.gov/articles/PMC12357140/ （Schoenpflug et al. 2025 実運用ケーススタディ）— 全文から4つの実運用障壁とRoche Diagnostics関与を確認
- https://arxiv.org/abs/2509.03427 （Hybrid HE論文）— 著者・性能数値（帯域2000倍削減、実行時間30%削減、サーバコスト15,621倍増）を確認
- https://pmc.ncbi.nlm.nih.gov/articles/PMC4264217/ （eTOX論文）— 著者・書誌情報を確認
- https://arxiv.org/abs/2511.00037 （FLフレームワーク比較論文）— 著者・各フレームワークの強み（FLARE=本番運用, Flower=研究, Substra=プライバシー）を確認
- https://arxiv.org/pdf/2511.00037 （同論文PDF）— バイナリのためテキスト抽出失敗、abs版で代替取得
- https://pubmed.ncbi.nlm.nih.gov/42509385/ （TheOden論文）— Cookie同意ページのみでアブストラクト取得失敗。WebSearchのスニペットから技術概要のみ確認（著者リストは未確認のまま papers/index.md に明記）

## 選定基準
- 「MELLODDY型アーキテクチャ」「連合学習×WSI」「連合学習×毒性学（QSAR含む）」の3系統を独立に調査し、交差点（毒性病理WSI×連合学習）の実装例の有無を多角的に検証
- MELLODDYが採用した暗号技術について、一次資料（arXiv論文本文相当）と技術ブログの両方でクロスチェックし、report.md執筆前に事実確認を優先
- 2023年以降の文献を優先しつつ、eTOX（2014年論文）はガバナンスモデルの対比のため意図的に含めた
- WebFetchが失敗した場合（PDFバイナリ、Cookie同意ページ等）は無理に推測で埋めず、papers/index.mdに「未確認」と明記

## 気づき・思考メモ
- 最大の発見は「topics/01のレポート（フロンティア6）の記述に不正確な点がある」ことだった。MELLODDYは準同型暗号を使っていない（Secure Aggregation/SMPCが主機構）。今回のreport.mdではこの点を明示的に訂正する。
- 「毒性病理WSI×連合学習」は完全に手つかずだが、その前提となる2つの要素（連合学習×WSI＝臨床病理で成熟、連合学習×毒性学＝QSARで実証済み）はどちらも別々に実績がある。ゆえに本フロンティアの障壁は基盤技術の欠如ではなく、両者を組み合わせる「最初の一歩」が踏み出されていないことにある。
- eTOXとMELLODDYという2つの異なるガバナンス前例が毒性領域に既に存在することは、PROP-06の実現可能性を評価する上で技術面以上に重要な材料になりうる。
