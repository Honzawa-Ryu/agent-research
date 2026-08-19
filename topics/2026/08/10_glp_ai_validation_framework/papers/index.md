# 論文・文献インデックス

PROP-09「GLP規制環境下でのAIバリデーション・信頼性保証フレームワーク」調査で精読した論文・規制ガイダンス文書一覧。本テーマは技術アーキテクチャではなく規制科学が中心のため、査読論文に加えFDA/EMA/PMDAの公式ガイダンス文書も対象に含む。

---

## 1. Pohlmeyer-Esch, G., Halsey, C., Boisclair, J., Ram, S., Kirschner-Kitz, S., Knight, B., Moulin, P., Frisk, A.-L. (2025)
**"Digital Pathology and Artificial Intelligence Applied to Nonclinical Toxicology Pathology—The Current State, Challenges, and Future Directions"**
*Toxicologic Pathology*, 53(6), 516–535. DOI: [10.1177/01926233251340622](https://doi.org/10.1177/01926233251340622) / [PMC12612283](https://pmc.ncbi.nlm.nih.gov/articles/PMC12612283/)

本調査の中核文献。GLP毒性病理へのAI適用における規制ギャップを包括的に整理。FDA/EMA/OECDガイダンスの現状、三段階同等性検証アプローチ（Box 1）、ISPE AIバリデーション成熟度モデル（6段階）、21 CFR Part 11/AlCOA要件、EU AI Act高リスク分類への言及を含む。「GLP毒性病理AIに特化した規制ガイダンスは現時点で存在しない」ことを明言する一次資料。

## 2. Thoolen, B., Bradley, A., Stathonikos, N., van Diest, P. J. (2025)
**"Toxicologic Pathology Forum: Opinion on the Future of Histopathology Using Whole Slide Images in Toxicologic Pathology of Preclinical Studies and Its Successful Implementation in Compliance With Good Laboratory Practice—Yes, We Are There!"**
*Toxicologic Pathology*, 53(7). DOI: [10.1177/01926233251366270](https://journals.sagepub.com/doi/10.1177/01926233251366270)

WSIの一次読影（primary read）がGLP環境で既に実運用レベルに達しているとする意見論文。デジタル病理単体（AI以前の段階）のGLP実装が到達点にあることを示し、AIバリデーションの前提となる「WSI自体の規制受容」が確立しつつあることを裏付ける。

## 3. FDA (2023)
**"Use of Whole Slide Imaging in Nonclinical Toxicology Studies: Questions and Answers" — Guidance for Industry**
最終版, 2023年5月. [PDF](https://www.fda.gov/media/168431/download) / [Federal Register](https://www.federalregister.gov/documents/2023/05/25/2023-11211/use-of-whole-slide-imaging-in-nonclinical-toxicology-studies-questions-and-answers-guidance-for)

GLP準拠の非臨床毒性試験におけるWSIの管理・文書化・使用に関するFDAの現行見解を示すQ&A形式ガイダンス。AIそのものへの言及はないが、AIトリアージが前提とするWSIデータ基盤の規制的位置づけを定める基礎文書。

## 4. FDA (2025, ドラフト)
**"Considerations for the Use of Artificial Intelligence To Support Regulatory Decision-Making for Drug and Biological Products" — Draft Guidance for Industry**
2025年1月公表（コメント募集中）. [Federal Register 2024-31542](https://www.federalregister.gov/documents/2025/01/07/2024-31542/considerations-for-the-use-of-artificial-intelligence-to-support-regulatory-decision-making-for-drug)

医薬品・生物製剤の安全性・有効性・品質に関する規制判断を支援するAIモデル全般に対する、リスクベースの信頼性評価枠組み（Context of Use: COUの定義から始まる段階的フレームワーク）を提示。非臨床データも対象範囲に含むが、病理画像AI・毒性病理トリアージへの個別言及はない。

## 5. FDA / Health Canada / UK MHRA（2021初版）、IMDRF（2025最終版）
**"Good Machine Learning Practice for Medical Device Development: Guiding Principles"**
[FDA PDF](https://www.fda.gov/media/153486/download)

AI/ML搭載医療機器開発のための10の指導原則（総製品ライフサイクルアプローチ、データ代表性、独立検証データセット、モデル限界の説明、市販後モニタリング等）。医療機器（SaMD）向けの枠組みであり、GLP前臨床試験プロセスへの直接適用は明記されていないが、本調査のコア問い1が転用可能性を検討する対象。

## 6. FDA（2024年最終）
**"Predetermined Change Control Plans for Machine Learning-Enabled Medical Devices: Guiding Principles"** および **"Marketing Submission Recommendations for a Predetermined Change Control Plan for Artificial Intelligence-Enabled Device Software Functions"**
[PCCP Guiding Principles PDF](https://www.fda.gov/media/173206/download)（2024年最終）、Marketing Submission Recommendations（2024年12月最終）

モデルの継続的更新を、変更のたびに再申請することなく事前承認された変更管理計画（PCCP）の範囲内で実施できる仕組み。GLP試験でAIモデルをバージョンアップする際の変更管理・監査証跡設計の参考枠組みとなりうる。

## 7. FDA（2024年最終）
**"Transparency for Machine Learning-Enabled Medical Devices: Guiding Principles"**
[PDF](https://www.fda.gov/media/179269/download)（2024年6月）

AI/MLモデルの透明性（意図された使用、学習・テストデータ、既知の限界の開示）に関する指導原則。Concept Bottleneck Model等の説明可能性技術（PROP-05）との接続点。

## 8. EMA（2024年9月、最終版）
**"Reflection Paper on the Use of Artificial Intelligence (AI) in the Medicinal Product Lifecycle"**
CHMP/CVMP採択, 2024年9月9日. [PDF](https://www.ema.europa.eu/en/documents/scientific-guideline/reflection-paper-use-artificial-intelligence-ai-medicinal-product-lifecycle_en.pdf)

医薬品ライフサイクル全体（創薬〜市販後）を対象としたAI利用に関するEMAの反省文書。人間主導のガバナンス、GxP原則を実装したSOP、リスク管理計画を推奨。動物試験（非臨床）段階のAI利用も対象範囲に含まれるが、毒性病理画像AIへの個別言及はない。PMDA（日本）はGLP毒性病理AIに特化したガイダンスを本調査時点で公表しておらず、一般的なプログラム医療機器審査の枠組み（J-STAGE解説論文等）に留まる。

## 9. Turner, O. C., Aeffner, F., Bangari, D. S., et al. (2020)
**"Society of Toxicologic Pathology Digital Pathology and Image Analysis Special Interest Group Article: Opinion on the Application of Artificial Intelligence and Machine Learning to Digital Toxicologic Pathology"**
*Toxicologic Pathology*, 48(6). DOI: [10.1177/0192623319881401](https://journals.sagepub.com/doi/full/10.1177/0192623319881401)

topics/01でも引用済みだが、本調査ではAI用語定義・バリデーション課題・規制ハードルの整理という観点で再照会。STPが業界として最初にAI/ML適用の論点を体系化した文書。

## 10. Tuomari, D. L., Kemp, R. K., Sellers, R., Yarrington, J. T., Geoly, F. J., Fouillet, X. L. M., Dybdal, N., Perry, R. (2007)
**"Society of Toxicologic Pathology Position Paper on Pathology Image Data: Compliance with 21 CFR Parts 58 and 11"**
*Toxicologic Pathology*, 35(3), 450–455. DOI: [10.1080/01926230701284509](https://doi.org/10.1080/01926230701284509) / [PubMed:17474067](https://pubmed.ncbi.nlm.nih.gov/17474067/)

病理画像データが「生データ」に該当する条件、21 CFR Part 58/11準拠の認証・アーカイブ要件を定めた基礎的な業界ポジションペーパー（AI以前）。AIトリアージ判定の監査証跡設計の出発点となる規制解釈を提供。

---

### 未収録・確認できなかった事項
- PMDA（日本）によるGLP毒性病理特化のAIバリデーションガイダンスは本調査時点で確認できなかった。
- 「AIによる陰性除外（Normal Triage）」に特化した感度/NPV基準の数値的な規制受容基準（閾値）を定めた一次文献は発見できなかった（Pohlmeyer-Esch 2025は「同等性・非劣性」を要求するのみで具体的閾値は示していない）。
