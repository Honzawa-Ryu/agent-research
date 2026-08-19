# 検索ログ・思考メモ

## 調査テーマ
[PROP-07] 動物毒性所見のヒト外挿性（Translational Toxicology）予測モデルの先行研究調査

## 検索クエリ一覧

| # | クエリ | ツール | ヒット数(目視) | 採用論文数 | メモ |
|---|---|---|---|---|---|
| 1 | translational toxicology machine learning animal to human extrapolation preclinical pathology | WebSearch | 8 | 1 (AI4TOX概要) | NCTR AI4TOXプログラムの存在を確認。TranslAIが本テーマに最も近い公式イニシアチブ |
| 2 | DILIrank machine learning prediction drug induced liver injury preclinical histopathology | WebSearch | 8 | 2 (DILIrank原著, Puri 2020) | Puri 2020はラット肝WSIの壊死パターン分類のみでヒト外挿はしていない点に注意 |
| 3 | predicting human adverse events from animal toxicology study machine learning | WebSearch | 10 | 0（背景理解用） | 「動物毒性試験はヒト臨床AEを予測できていない」という問題設定自体を確認する目的 |
| 4 | AI model predicts drug toxicity differences between animal models and humans 2025 | WebSearch | 6 | 1 (Kim et al. 2025 eBioMedicine) | 本調査のコア問いに最も近い最新論文。ただし入力は遺伝子発現ネットワーク特徴でありWSI形態は不使用 |
| 5 | ClinTox dataset graph neural network toxicity prediction knowledge graph drug safety | WebSearch | 8 | 1 (Xie et al. 2025 Toxics) | KG+GNNのアーキテクチャ候補。化学構造ベースで病理画像は未使用 |
| 6 | FAERS pharmacovigilance machine learning predict preclinical animal findings translation | WebSearch | 8 | 1 (Maciejewski et al. 2017 eLife、詳細確認) | 逆方向（ヒトFAERS→前臨床オフターゲット予測）のprecedent |
| 7 | NCTR AI4TOX PathologAI AnimalGAN TranslAI official FDA program description | WebSearch | 9 | 1 (FDA公式ページ) | AI4TOXの5イニシアチブ（AnimalGAN, SafetAI, PathologAI, BERTox, TranslAI）を確認 |
| 8 | Bussola 2023 PathologAI FDA NCTR histopathology rat liver deep learning | WebSearch | 6 | 1 (Bussola et al. 2023) | DILIrank著者とPathologAI著者に共通してWeida Tong (FDA/NCTR)がいることを発見。同一グループが両ピースを保有しながら未統合という構図の裏付け |
| 9 | "histopathology" OR "pathology finding" animal-to-human translation deep learning multimodal drug safety 2025 2026 | WebSearch | 10 | 1 (Mehrvar et al. 2021レビュー、詳細確認) | 毒性病理DLレビューはヒト外挿には未言及であることを確認（ネガティブ結果として重要） |
| 10 | FDA Roadmap Reducing Animal Testing Preclinical Safety Studies April 2025 | WebSearch | 10 | 1 (FDA Roadmap公式PDF) | 規制側の動機付けとして採用 |
| 11 | TG-GATEs Open TG-GATEs human hepatotoxicity clinical outcome prediction rat gene expression translational | WebSearch | 9 | 1 (O'Donovan et al. 2020 PLOS ONE) | rat→human遺伝子発現変換の先行例。WSI形態は不使用 |
| 12 | "Use of deep learning methods to translate drug-induced gene expression changes..." authors citation | WebSearch | 5 | (citation確認のみ) | O'Donovan論文の正確な著者名確認 |

WebFetchによる詳細確認: PMC5548487 (Maciejewski 2017), PMC7293302 (Gardiner 2020), PMC8609289 (Mehrvar 2021), PMC12656225 (Xie 2025), eurekalert.org記事(Kim 2025), FDA公式AI4TOXページ, PubMed 31149832 (Puri 2020)。

eBioMedicine原文(thelancet.com)とFDA TranslAI専用ページは403/404で直接取得不可のため、EurekAlert!プレスリリースとFDA全体ページ経由の情報で代替した。

## 選定基準
- 「動物病理所見（特にWSI形態）→ヒト有害事象予測」という直接統合を扱う論文を最優先
- 見つからない場合は、(a) 動物→ヒトの分子・遺伝子発現レベルでの外挿研究、(b) ヒト毒性知識ベース(DILIrank/ClinTox/FAERS)を用いたモデル、(c) 逆方向（ヒト→動物）の外挿研究、を代替エビデンスとして採用し、WSI形態情報が組み込まれていない旨を明記する
- 実在確認できない論文は一切引用しない
