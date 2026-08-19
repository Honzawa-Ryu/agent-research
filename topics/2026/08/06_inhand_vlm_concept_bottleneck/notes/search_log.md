# 検索ログ・思考メモ

## 調査テーマ
[PROP-05] INHAND国際オントロジー準拠のVision-Language基盤モデル＆Concept Bottleneck Model設計の先行事例調査

## 検索クエリと結果概要

| # | 検索クエリ | 使用ツール | ヒット件数（概算） | 選定基準・所感 |
|---|---|---|---|---|
| 1 | INHAND nomenclature toxicologic pathology vision-language model AI 2025 2026 | WebSearch | 9件 | INHANDとVLMを直接結びつけた研究はヒットせず。PathologyVLM, MUSK等の一般病理VLMのみ検出 → ギャップの初期確認 |
| 2 | Concept Bottleneck Model pathology histopathology interpretable diagnosis | WebSearch | 10件 | CBM×病理の主要系譜（Label-free Concept MIL, ConceptM3oE, MMCBM, MICCAI2024 Pang et al.）を把握 |
| 3 | CONCH PLIP pathology vision language foundation model contrastive learning 2024 | WebSearch | 9件 | CONCH/PLIPの技術的差分（学習データ規模・アーキテクチャ）を確認 |
| 4 | toxicologic pathology large language model INHAND terminology GLP explainability | WebSearch | 9件 | INHAND自体の公式解説（STP, Mann et al. 2012）は見つかるが、LLM/VLMとの統合研究は不在 |
| 5 | Label-free Concept Based Multiple Instance Learning Gigapixel Histopathology arxiv | WebSearch | 8件 | 本調査のコア実装候補として選定（Sun et al. 2025, arXiv:2501.02922） |
| 6 | interpretable MIL prototype pathology whole slide image concept explainable diagnosis grading | WebSearch | 7件 | ProtoMIL, PAMIL等プロトタイプ系手法を発見。CBM系とは異なるアプローチとして整理 |
| 7 | Pfizer TRACE toxicologic pathology explainability interpretability model 2024 2025 | WebSearch | 9件 | TRACE（毒性病理SOTA）の説明性がアテンションマップに留まり、明示的概念層を持たないことを確認 |
| 8 | "toxicologic pathology" ontology ordinal severity grading deep learning minimal mild moderate marked severe | WebSearch | 8件 | 重症度グレーディングの半定量性の課題、Turner et al. 2022レビュー、PathologAI論文を発見 |
| 9 | INHAND ontology structured text embedding veterinary pathology NLP dataset | WebSearch | 10件 | 獣医病理NLPレビュー（Stimmer et al. 2025）を発見。INHAND×NLPが「不在」であることの一次証拠として重要 |
| 10 | QuiltNet BiomedCLIP MedCLIP pathology text encoder contrastive concept alignment | WebSearch | 10件 | Quilt-1M, BiomedCLIP, ConceptCLIP等のテキストエンコーダ候補を整理 |
| 11 | "PathologAI" weakly supervised whole slide animal study toxicology Chemical Research Toxicology | WebSearch | 7件 | PathologAI（Bussola et al. 2023）の詳細（Open TG-GATEsラット肝壊死検出）を確認 |
| 12 | Turner Rudmann Society Toxicologic Pathology digital pathology AI special interest group review 2021 2022 deep learning | WebSearch | 7件 | STP SIGの一連の提言論文（Turner et al. 2020/2021）を再確認（topics/01と同一文献） |
| 13 | INHAND ontology structured text embedding veterinary pathology NLP dataset（再掲/深掘り） | WebFetch (journals.sagepub.com) | - | Stimmer et al. 2025全文からINHAND・CBMへの言及が皆無であることを直接確認 |
| 14 | Concept Bottleneck Model FDA GLP regulatory audit trail explainable AI medical device validation | WebSearch | 9件 | 規制側がCBM的な説明可能性を要求する文脈（FDA AI guidance 2025）を確認。詳細はPROP-09で深掘り予定 |
| 15 | MICCAI 2024 "Integrating Clinical Knowledge into Concept Bottleneck Models" | WebSearch | 8件 | Pang et al. 2024の詳細（臨床知識をCBMに統合する手法）を確認 |
| 16 | ConceptCLIP concept-enhanced vision-language pretraining biomedical arxiv 2501.15579 authors | WebSearch | 7件 | ConceptCLIP（Nie et al. 2025, MedConcept-23M, UMLS概念使用）の詳細を確認 |
| 17 | Quilt-1M million image-text pairs histopathology Ikezogwo NeurIPS 2023 citation | WebSearch | 10件 | Quilt-1Mの正式書誌情報を確定 |

## WebFetchで全文確認したソース
- arXiv:2501.02922 (Label-free Concept Based MIL) — アブストラクト・手法概要
- arXiv:2401.02044 (AFLoc) — 胸部X線中心でトキシコパソロジーとの直接関連は薄いと判断し不採用
- journals.sagepub.com/doi/10.1177/03009858251347529 (Stimmer et al. 2025 獣医病理NLPレビュー) — INHAND/CBM言及の不在を確認する決定的ソース
- pmc.ncbi.nlm.nih.gov/articles/PMC10445282 (PathologAI, Bussola et al. 2023) — 著者・書誌情報・手法詳細を確認

## 選定基準
- 2023年以降の文献を優先（VLM/CBMの進展が速いため）
- 病理領域のVLM/CBM論文はarXiv/Nature Medicine/MICCAI等の一次ソースを優先
- 「INHAND × AI」の直接的な先行研究の有無を複数の切り口（VLM, LLM, NLP, CBM, ontology）から多重検証し、不在であることの確度を高めた
- 毒性病理の実運用モデル（TRACE, PathologAI）については、既存レポート（topics/01, 04）との重複を避けつつ「説明可能性」の観点でのみ深掘り

## 気づき・思考メモ
- 「INHAND × VLM/CBM」を明示的に扱った論文は、2026年8月時点で一次文献ベースで確認できなかった。本フロンティアは真に未着手のホワイトスペースと判断できる。
- ただし要素技術（概念ボトルネック層の無annotation構築、医療オントロジーとのテキストエンコーダ統合）は疾患病理・一般医用画像分野で急速に成熟しており、INHAND統合への転用障壁は主に「INHAND定義文をどう構造化データ化するか」というデータエンジニアリング上の課題に集約されそうである。
- Stimmer et al. 2025（獣医病理NLPレビュー）が「veterinary-specific NLP applications remain largely unexplored」と明言している点は、本提案の妥当性を裏付ける強い状況証拠。
