# 📚 調査トピック総合インデックス (Research Index)

本リポジトリで実施されたすべての調査レポートの目次です。  
新規調査が完了した際、Agentがこのテーブルに1行追記します。

---

## 📑 調査レポート一覧

| 実施年月 | テーマ / タイトル | カテゴリ / タグ | レポート | 主な結論・ハイライト | 収集論文数 |
|:---:|:---|:---|:---:|:---|:---:|
| 2026-08 | 毒性病理画像と疾患病理画像の違い・共通点と未開拓な応用可能「ねらい目」技術 | `#毒性病理` `#疾患病理` `#基盤モデル` `#弱教師あり学習` `#トキシコゲノミクス` | [Report](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md) | 個別診断(N=1)の疾患病理に対し、毒性病理は群間比較・用量反応・40+臓器スクリーニングが本質。動物種横断基盤モデル、用量反応MIL、背景病変差分異常検知、INHAND準拠VLM、Patho-TGx、連合学習の6大ねらい目技術を体系化。 | 10件 (PDF 6件) |
| 2026-08 | 動物種横断（Cross-Species）病理基盤モデルの構築手法と形態学的相同性アライメント技術 | `#基盤モデル` `#CrossSpecies` `#ドメイン適応` `#毒性病理` `#LoRA` `#VisionLanguage` | [Report](topics/2026/08/02_cross_species_pathology_fm/report.md) | ヒト病理基盤モデル(UNI等)を動物毒性病理へ直接転用した実証はUNI→ラット腎臓異常検知(AUC 0.62)の1件のみ。現行の毒性病理AIは概ねImageNet/DINOv2ベースでヒト病理FM未活用。形態空間(morphospace overlap)と意味空間(Semantic Anchoring)という2つのアライメント座標系を整理し、非腫瘍性病変への拡張とヒト病理FMの系統的ゼロショット評価が今後の核心課題と特定。 | 11件 (PDF 3件) |
| 2026-08 | 対照群Normativeモデルによる反事実的異常検知（Counterfactual Anomaly Detection）の生成AIアーキテクチャ | `#異常検知` `#拡散モデル` `#反事実生成` `#背景病変` `#NormativeModeling` `#毒性病理` | [Report](topics/2026/08/03_counterfactual_anomaly_detection/report.md) | 「毒性病理×反事実生成」の直接実装例は皆無。脳MRIの反事実拡散異常検知(Wolleb 2022等)と疾患病理WSIへの実装(MoPaDi 2024)は個体内反事実に留まり、Control群の集団分布を正規モデルとする「群レベル正規分布反事実」の設計思想が根本的に未着手であることを特定。WSIギガピクセル生成の計算コストは実用域に近づきつつあるが異常検知への統合検証は不在。 | 10件 (PDF 6件) |
| 2026-08 | 毒性試験における用量反応性（Dose-Response）モデリングと階層的弱教師あり学習（Hierarchical Group-MIL）の数理・実装設計 | `#弱教師あり学習` `#MIL` `#用量反応性` `#DoseResponse` `#PfizerTRACE` `#単調性制約` | [Report](topics/2026/08/04_dose_response_hierarchical_mil/report.md) | 実運用SOTA(TRACE, Graf et al. 2026)でも用量反応は「群平均+log2FC正規化」の統計的後処理に留まり、単調性正則化・NOAEL/BMD自動算出は未実装。「解剖学的階層MIL」と「実験デザイン階層(群-個体-臓器-スライド-パッチ)」が異なる概念であることを整理し、単調性制約ニューラルネット理論(CORAL, UMNN等)をMILのバッグ集約に応用する具体的実装ルートを提示。 | 11件 (PDF 4件) |
| 2026-08 | Open TG-GATEsを活用したトキシコゲノミクス×毒性病理WSIマルチモーダルAI（Patho-TGx）の先行研究とベンチマーク調査 | `#トキシコゲノミクス` `#OpenTGGATEs` `#マルチモーダル` `#VirtualPathology` `#GEESE` `#TANGLE` | [Report](topics/2026/08/05_patho_toxicogenomics/report.md) | 順方向(WSI→遺伝子発現)はGEESE(Jaume 2024)がTG-GATEs全156試験規模で実装済み、病変分類macro-AUC 98.9%だが発現量回帰はr=0.29と精度に非対称性。逆方向(化合物構造→仮想病理像生成)はPathologAI原著者が構想を明記するも2026年8月時点で未実装のまま。TANGLEがヒト+ラット肝臓データで動物種横断とトキシコゲノミクス統合を意図せず橋渡ししている点を新規発見。 | 10件 (PDF 4件) |
| 2026-08 | INHAND国際オントロジー準拠のVision-Language基盤モデル＆Concept Bottleneck Model設計の先行事例調査 | `#VisionLanguage` `#ConceptBottleneck` `#INHAND` `#説明可能AI` `#GLP規制` `#ConceptCLIP` | [Report](topics/2026/08/06_inhand_vlm_concept_bottleneck/report.md) | 「INHAND×VLM/CBM」の直接統合研究は一次文献上不在(獣医病理NLP最新レビューStimmer 2025も同分野を「ほぼ未開拓」と明言)。要素技術(Label-free Concept MIL, Clinical-Knowledge CBM, ConceptCLIPのUMLS概念源)は疾患病理側で成熟済みで転用は射程内だが、INHAND定義文を機械可読な概念ラベル集合へ構造化する工程が最大のボトルネックと特定。現行毒性病理SOTA(TRACE, PathologAI)は精度は高いが概念レベルの説明可能性を持たない。 | 10件 |
| 2026-08 | MELLODDY型・製薬企業間連合学習コンソーシアムの毒性病理領域への応用可能性と実装障壁 | `#連合学習` `#MELLODDY` `#プライバシー保護` `#業界コンソーシアム` `#SecureAggregation` `#毒性病理` | [Report](topics/2026/08/07_federated_learning_toxpath/report.md) | 既存レポート(01)のMELLODDY記述を訂正：主機構はSecure Aggregation(SMPC)で準同型暗号は不採用。「毒性病理WSI×連合学習」の直接実装例は皆無だが、隣接2領域(連合学習×臨床WSI、連合学習×毒性QSAR)はそれぞれ実績あり。Rocheが臨床病理WSIで連合学習を実運用検証済み(Schoenpflug 2025)。準同型暗号のWSIスケール適用はサーバコスト15,621倍増と依然重い。 | 10件 |

---

## 🏷 カテゴリ・タグ別クイックリンク

- **病理画像 / 創薬安全性 (Digital Pathology & Safety AI)**
  - [2026-08] [毒性病理画像と疾患病理画像の違い・共通点と未開拓な応用可能「ねらい目」技術](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
  - [2026-08] [動物種横断（Cross-Species）病理基盤モデルの構築手法と形態学的相同性アライメント技術](topics/2026/08/02_cross_species_pathology_fm/report.md)
  - [2026-08] [対照群Normativeモデルによる反事実的異常検知（Counterfactual Anomaly Detection）の生成AIアーキテクチャ](topics/2026/08/03_counterfactual_anomaly_detection/report.md)
  - [2026-08] [毒性試験における用量反応性（Dose-Response）モデリングと階層的弱教師あり学習（Hierarchical Group-MIL）の数理・実装設計](topics/2026/08/04_dose_response_hierarchical_mil/report.md)
  - [2026-08] [Open TG-GATEsを活用したトキシコゲノミクス×毒性病理WSIマルチモーダルAI（Patho-TGx）の先行研究とベンチマーク調査](topics/2026/08/05_patho_toxicogenomics/report.md)
  - [2026-08] [INHAND国際オントロジー準拠のVision-Language基盤モデル＆Concept Bottleneck Model設計の先行事例調査](topics/2026/08/06_inhand_vlm_concept_bottleneck/report.md)
  - [2026-08] [MELLODDY型・製薬企業間連合学習コンソーシアムの毒性病理領域への応用可能性と実装障壁](topics/2026/08/07_federated_learning_toxpath/report.md)
- **基盤モデル / マルチモーダル (Foundation Models & Multimodal)**
  - [2026-08] [毒性病理画像と疾患病理画像の違い・共通点と未開拓な応用可能「ねらい目」技術](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
  - [2026-08] [動物種横断（Cross-Species）病理基盤モデルの構築手法と形態学的相同性アライメント技術](topics/2026/08/02_cross_species_pathology_fm/report.md)
  - [2026-08] [Open TG-GATEsを活用したトキシコゲノミクス×毒性病理WSIマルチモーダルAI（Patho-TGx）の先行研究とベンチマーク調査](topics/2026/08/05_patho_toxicogenomics/report.md)
  - [2026-08] [INHAND国際オントロジー準拠のVision-Language基盤モデル＆Concept Bottleneck Model設計の先行事例調査](topics/2026/08/06_inhand_vlm_concept_bottleneck/report.md)
- **異常検知 / 生成AI (Anomaly Detection & Generative AI)**
  - [2026-08] [対照群Normativeモデルによる反事実的異常検知（Counterfactual Anomaly Detection）の生成AIアーキテクチャ](topics/2026/08/03_counterfactual_anomaly_detection/report.md)
- **弱教師あり学習 / MIL (Weakly Supervised Learning & MIL)**
  - [2026-08] [毒性試験における用量反応性（Dose-Response）モデリングと階層的弱教師あり学習（Hierarchical Group-MIL）の数理・実装設計](topics/2026/08/04_dose_response_hierarchical_mil/report.md)
- **トキシコゲノミクス / マルチモーダル統合 (Toxicogenomics & Multimodal Integration)**
  - [2026-08] [Open TG-GATEsを活用したトキシコゲノミクス×毒性病理WSIマルチモーダルAI（Patho-TGx）の先行研究とベンチマーク調査](topics/2026/08/05_patho_toxicogenomics/report.md)
- **説明可能AI / 規制対応 (Explainable AI & Regulatory Science)**
  - [2026-08] [INHAND国際オントロジー準拠のVision-Language基盤モデル＆Concept Bottleneck Model設計の先行事例調査](topics/2026/08/06_inhand_vlm_concept_bottleneck/report.md)
- **連合学習 / プライバシー保護 (Federated Learning & Privacy)**
  - [2026-08] [MELLODDY型・製薬企業間連合学習コンソーシアムの毒性病理領域への応用可能性と実装障壁](topics/2026/08/07_federated_learning_toxpath/report.md)
