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

---

## 🏷 カテゴリ・タグ別クイックリンク

- **病理画像 / 創薬安全性 (Digital Pathology & Safety AI)**
  - [2026-08] [毒性病理画像と疾患病理画像の違い・共通点と未開拓な応用可能「ねらい目」技術](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
  - [2026-08] [動物種横断（Cross-Species）病理基盤モデルの構築手法と形態学的相同性アライメント技術](topics/2026/08/02_cross_species_pathology_fm/report.md)
  - [2026-08] [対照群Normativeモデルによる反事実的異常検知（Counterfactual Anomaly Detection）の生成AIアーキテクチャ](topics/2026/08/03_counterfactual_anomaly_detection/report.md)
- **基盤モデル / マルチモーダル (Foundation Models & Multimodal)**
  - [2026-08] [毒性病理画像と疾患病理画像の違い・共通点と未開拓な応用可能「ねらい目」技術](topics/2026/08/01_toxicology_vs_clinical_pathology/report.md)
  - [2026-08] [動物種横断（Cross-Species）病理基盤モデルの構築手法と形態学的相同性アライメント技術](topics/2026/08/02_cross_species_pathology_fm/report.md)
- **異常検知 / 生成AI (Anomaly Detection & Generative AI)**
  - [2026-08] [対照群Normativeモデルによる反事実的異常検知（Counterfactual Anomaly Detection）の生成AIアーキテクチャ](topics/2026/08/03_counterfactual_anomaly_detection/report.md)
