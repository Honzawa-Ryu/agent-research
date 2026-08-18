

# 調査メモ・検索ログ (Search & Investigation Log)

本調査における検索クエリ、情報ソース、選定・除外基準、思考ログを記録します。

---

## 🔍 検索クエリ履歴

| 日時 | ソース (arXiv / PubMed / Google Scholar 等) | 検索クエリ | ヒット件数 | 採用件数 | 備考 |
|:---|:---|:---|:---:|:---:|:---|
| 2026-08-18 | PubMed / Google Scholar | `"toxicologic pathology" "artificial intelligence" OR "machine learning" "computational pathology" review` | 42 | 4 | 毒性病理AIの現状と課題（Turner 2020, Rudmann 2021, Song 2024等）を抽出 |
| 2026-08-18 | PubMed / arXiv | `"toxicologic pathology" "clinical pathology" difference deep learning OR "artificial intelligence"` | 35 | 3 | 臨床診断病理と毒性病理の目的・データ・ワークフローの相違点を整理 |
| 2026-08-18 | arXiv / Google Scholar | `"computational pathology" "foundation model" UNI CONCH Virchow CHIEF Prov-GigaPath` | 120+ | 5 | 疾患病理基盤モデル（SOTA技術動向）の整理 |
| 2026-08-18 | BioRxiv / PubMed | `"Open TG-GATEs" deep learning pathology OR "whole slide"` | 28 | 3 | トキシコゲノミクスデータベースと病理AIの先行研究を抽出 |
| 2026-08-18 | BioRxiv / PubMed | `"TRACE: Multi-scale deep learning" toxicologic liver histopathology` | 12 | 2 | 毒性病理特化型深層学習モデルTRACEの調査 |
| 2026-08-18 | arXiv / PubMed | `"cross-species" "computational pathology" OR "digital pathology" human mouse OR rat` | 24 | 3 | 種差克服・トランスレーショナル病理AIの動向 |
| 2026-08-18 | PubMed / arXiv | `"toxicologic pathology" "anomaly detection" OR "out-of-distribution"` | 19 | 3 | 毒性病理における異常検知・背景病変識別の技術調査 |

---

## 🎯 論文の選定・除外基準

- **採用基準**:
  1. 毒性病理（前臨床・非臨床安全性評価）と疾患病理（臨床腫瘍・診断病理）の比較に寄与する査読付き論文・国際学会論文・重要サーベイ。
  2. 疾患病理においてブレイクスルーをもたらしている最新技術（基盤モデル、Vision-Language、MIL、Spatial Biology、反事実生成、連邦学習）。
  3. 毒性病理特化のAI研究（Open TG-GATEs活用、TRACE、INHAND準拠、用量反応性モデリング等）。
  4. コード、モデルウェイト、またはオープンアクセスPDFが入手可能な研究を優先。
- **除外基準**:
  1. 特定の単一化合物・単一症例のみを対象とし、技術的汎用性・比較視点のない報告。
  2. 実験設定やバリデーションが不透明で再現性の低いプレプリント。

---

## 💡 調査中の思考メモ・ブレインストーミング

### 1. 疾患病理と毒性病理の本質的な相違点
- **疾患病理のAIタスク**: 個別患者（N=1）の診断・分類・予後予測。腫瘍の存在やサブタイプなど「局所的な顕著病変」の検出が中心。データセット（TCGA等）やコンペティションが充実しており、研究開発が先行。
- **毒性病理のAIタスク**: 投与群（Control vs Low/Mid/High）の統計的有意差検出と用量反応性評価。1個体あたり40〜50臓器の全身網羅的スクリーニング。病変は「微細な細胞変性・びまん性変化」であり、さらに加齢や動物系統による「自然発生背景病変」との厳密な鑑別が必須。データのほとんどが製薬企業・CROの社内秘（機密保持）であり、オープンデータが極端に少ない。

### 2. 「未開拓だが極めてねらい目の技術」の着眼点
- **着眼点A (基盤モデルの種差展開)**: ヒトの巨大学習済み病理基盤モデル（UNI/CONCH/Virchow等）を動物種（ラット/マウス/サル等）へどう効率的に転移・アライメントさせるか。
- **着眼点B (用量反応性MIL)**: 従来のスライド単位MILから、試験群・個体・臓器・スライド・パッチの階層グラフを扱い、用量相関性（単調増加性等）を正則化に組み込んだ Hierarchical Group-MIL。
- **着眼点C (差分・反事実異常検知)**: 対照群（Control）の正常＋背景病変分布をベースラインとし、投与群特有の形態変化のみを「差分」として抽出・可視化する Normative / Counterfactual Modeling。
- **着眼点D (INHAND準拠VLM & CBM)**: 毒性病理の国際基準INHANDの語彙・重症度スケールを組み込んだ解釈可能モデル。
- **着眼点E (Patho-Toxicogenomics)**: 組織画像からトキシコゲノミクス（発現変化・バイオマーカー）を予測、あるいは化合物の構造式から誘発病理像をシミュレーションする仮想毒性病理。
- **着眼点F (連邦学習コンソーシアム)**: MELLODDYプロジェクトの成功例を毒性病理画像に適用し、製薬企業間のWSI秘匿性を保ったまま超巨大毒性基盤モデルを共同訓練するアプローチ。
