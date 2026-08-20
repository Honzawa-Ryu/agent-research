# 収集論文・リソース一覧

本調査で参照・精読した論文および関連リソースの書誌情報と要約です。
UNI/UNI2(UNI2-h)を動物(非ヒト)病理組織画像に適用した一次研究を中心に収集しています。

---

## 📚 論文メタデータ一覧

| No. | タイトル | 著者 / 年 / 会議 | リンク (arXiv/DOI) | 対象動物種 | 重要度 |
|:---|:---|:---|:---|:---|:---:|
| 1 | [Is Self-Supervision Enough? Benchmarking Foundation Models Against End-to-End Training for Mitotic Figure Classification](#paper-1) | Ganz, Ammeling, Rosbach, Lausser, Bertram, Breininger, Aubreville (2024) | [arXiv:2412.06365](https://arxiv.org/abs/2412.06365) | イヌ(肺癌・リンパ腫・肥満細胞腫) | ★★★ |
| 2 | [Benchmarking Foundation Models for Mitotic Figure Classification](#paper-2) | Ammeling, Ganz, Rosbach, Lausser, Bertram, Breininger, Aubreville (2026, MELBA) | [arXiv:2508.04441](https://arxiv.org/abs/2508.04441) / [DOI](https://doi.org/10.59275/j.melba.2026-a3eb) | イヌ(肥満細胞腫・多腫瘍種) | ★★★ |
| 3 | [Beyond Classification: Pathology Foundation Models as Detection Encoders for Mitotic Figures](#paper-3) | Banerjee, Teimoury, Porsche, Stoll, Weiss, Hargarter, Ammeling, Conrad, Stroblberger, Kaltnecker, Klopfleisch, Bertram, Breininger, Aubreville (2026) | [arXiv:2607.28007](https://arxiv.org/abs/2607.28007) | イヌ(肺癌・リンパ腫・肥満細胞腫) | ★★★ |
| 4 | [Parameter-efficient fine-tuning (PEFT) of Vision Foundation Models for Atypical Mitotic Figure Classification](#paper-4) | Ramchandani, Deotale, Das (2025, Aira Matrix) | [arXiv:2509.16935](https://arxiv.org/abs/2509.16935) | イヌ・ネコ(MIDOG2025) | ★★☆ |
| 5 | [Self-supervised large-scale kidney abnormality detection in drug safety assessment studies](#paper-5) | Slootweg, García-De-La-Puente, Litjens, Dammak (2025) | [arXiv:2509.00131](https://arxiv.org/abs/2509.00131) | ラット(腎臓・毒性病理) | ★★★ |
| 6 | [Mitosis Detection in the Wild: Multi-Tumor and Context-Aware Generalization in the MIDOG 2025 Challenge](#paper-6) | MIDOG 2025 Challenge運営チーム (2026) | [arXiv:2606.07368](https://arxiv.org/abs/2606.07368) | ヒト・イヌ・ネコ(12腫瘍種、365症例) | ★★☆ |
| 7 | [Beyond the Failures: Rethinking Foundation Models in Pathology](#paper-7) | (2025-2026) | [arXiv:2510.23807](https://arxiv.org/abs/2510.23807) | (直接適用なし、批評論文) | ★★☆ |
| 8 | [Reporting transparency in veterinary pathology deep learning](#paper-8) | Banerjee, Bertram, Weiss, et al. (2026) | [DOI:10.1177/03009858261459452](https://doi.org/10.1177/03009858261459452) | (獣医病理DL全般のレビュー、既存topic02/10で既収載) | ★☆☆ |

---

## 📝 論文詳細サマリー

### <a id="paper-1"></a> [1] Is Self-Supervision Enough? Benchmarking Foundation Models Against End-to-End Training for Mitotic Figure Classification
- **著者**: Jonathan Ganz, Jonas Ammeling, Emely Rosbach, Ludwig Lausser, Christof A. Bertram, Katharina Breininger, Marc Aubreville
- **掲載**: arXivプレプリント (2024-12-09投稿, v2: 2024-12-16)。BVM2025向け短縮版あり。
- **リンク**: [arXiv:2412.06365](https://arxiv.org/abs/2412.06365)

#### 概要
UNIを含む複数の病理基盤モデルを凍結特徴抽出器として使い、線形分類器のみを学習させて有糸分裂像(mitotic figure)分類タスクを解いた場合の性能を、End-to-endで学習したResNet50と比較した研究。データセットはMIDOG(ドメイン2:イヌ肺癌、ドメイン3:イヌリンパ腫、ドメイン4:CCMCT=イヌ肥満細胞腫を含む)とCCMCT(32枚のイヌ肥満細胞腫WSI)。

#### 手法のポイント
- UNI(ViT-L)を含むFMを凍結し、抽出した特徴量に線形プローブを学習。
- In-domain(学習・評価が同一ドメイン)とOut-of-domain(未見ドメインへの汎化)を分けて評価。

#### 結果・貢献
- UNIのAUROC: **In-domain 0.76±0.03 / Out-of-domain 0.64±0.05**(H-optimus-0はOOD 0.79±0.04で優位)。
- **限界**: 「FMベースの分類器は、我々のEnd-to-end学習ResNet50モデルよりドメインシフトに頑健というわけではなかった("FM-based classifiers didn't turn out to be more robust to domain shifts than one of our end-to-end trained ResNet50 model")」— ヒト病理で学習したFMが動物データを含むドメインシフトに対して自動的に優位性を持つわけではないという重要な反証。
- **改善への言及**: 「特定タスクに対してFMをファインチューニングすればより良い結果が得られる可能性がある("better results might be achievable if the FMs were fine-tuned for the specific task")」との推測に留まり、本論文内では未実装。

---

### <a id="paper-2"></a> [2] Benchmarking Foundation Models for Mitotic Figure Classification
- **著者**: Jonas Ammeling, Jonathan Ganz, Emely Rosbach, Ludwig Lausser, Christof A. Bertram, Katharina Breininger, Marc Aubreville
- **所属**: Technische Hochschule Ingolstadt, MIRA Vision Microscopy GmbH, University of Veterinary Medicine Vienna 他
- **掲載**: *Machine Learning for Biomedical Imaging* (MELBA), vol.3, MELBA–BVM 2025 Special Issue, pp.38–55 (2026)
- **リンク**: [arXiv:2508.04441](https://arxiv.org/abs/2508.04441) / [DOI:10.59275/j.melba.2026-a3eb](https://doi.org/10.59275/j.melba.2026-a3eb)

#### 概要
Phikon, UNI, Virchow, Virchow2, H-optimus-0, Prov-GigaPathの6基盤モデルを、CCMCT(イヌ肥満細胞腫、4万以上の有糸分裂像アノテーション)とMIDOG2022(多腫瘍・多施設・多種)の2データセットでベンチマーク。線形プロービングとLoRAの両方を比較した、論文1の直接の発展版。

#### 手法のポイント
- 線形プロービング(FM凍結)とLoRA(パラメータ効率的ファインチューニング、内部Attentionを調整)を比較。
- ドメイン内/ドメイン外(クロスドメイン)の両方を評価。
- UNI(ViT-L, 304Mパラメータ)のみを評価対象とし、UNI2は本論文の評価対象外。

#### 結果・貢献
- UNI AUC: CCMCT **0.83±0.00(線形プローブ)→ 0.87±0.01(LoRA)**、MIDOG2022 **0.83±0.00(線形プローブ)→ 0.85±0.07(LoRA)**。
- クロスドメイン: ドメイン内平均AUC 0.86±0.05(LoRA)、ドメイン外平均AUC 0.80±0.06(LoRA)。特に「ドメインE(イヌ肺癌)」が難易度が高いと指摘。
- **限界**: 「施設間の染色プロトコルとスキャナーの違いが組織の見え方を変え、頑健な手法の開発・展開を複雑にする("differences in staining protocols and scanning devices across institutions can alter tissue appearance, complicating the development and deployment of robust methods")」。
- **改善への言及(最重要)**: LoRA適用によりOut-of-domain AUCが**線形プローブ時0.61-0.66 → LoRA適用時0.85-0.88**へと劇的に改善。また、LoRA適用モデルはわずか10%のデータで100%データ使用時に近い性能に到達(特にH-optimus-0で顕著)。「基盤モデル+LoRA」という組み合わせが、動物種を跨ぐドメインシフトへの最も具体的で再現性のある改善策として示された。

---

### <a id="paper-3"></a> [3] Beyond Classification: Pathology Foundation Models as Detection Encoders for Mitotic Figures
- **著者**: Sweta Banerjee, Alireza Teimoury, Nils Porsche, Alexandra K. Stoll, Viktoria Weiss, Niklas Hargarter, Jonas Ammeling, Thomas Conrad, Christoph Stroblberger, Christopher Kaltnecker, Robert Klopfleisch, Christof A. Bertram, Katharina Breininger, Marc Aubreville
- **掲載**: arXivプレプリント (2026-07-30投稿)
- **リンク**: [arXiv:2607.28007](https://arxiv.org/abs/2607.28007)

#### 概要
論文1・2と同一研究クラスタ(Banerjeeは[topic 02](../../02_cross_species_pathology_fm/report.md)・[topic 10](../../10_glp_ai_validation_framework/report.md)で既出の獣医病理DLレビュー著者)による発展研究。UNIとUNI2-hを含む複数FMを「分類器の特徴抽出器」としてではなく、物体検出器(Faster R-CNN, RetinaNet)の**バックボーンエンコーダ**として使う設計で、MIDOG++(イヌ肺癌・イヌリンパ腫・CCMCT等を含む)とTUPAC16(ドメイン外評価)で検証。

#### 手法のポイント
- UNI/UNI2-hの特徴マップをFaster R-CNN/RetinaNetのバックボーンとして統合。
- MIDOG++(ドメイン内)とTUPAC16(ドメイン外、乳がん有糸分裂検出データセット)で汎化性能を評価。

#### 結果・貢献
- MIDOG++テストF1: **UNI+Faster R-CNN 0.4513 / UNI+RetinaNet 0.6614** vs **UNI2-h+Faster R-CNN 0.7559 / UNI2-h+RetinaNet 0.7178**。UNI2世代への更新で顕著な性能向上。
- TUPAC16(OOD)F1: **UNI+Faster R-CNN 0.3952 → UNI2-h+Faster R-CNN 0.7043**。世代間の頑健性向上がより顕著。
- **限界**: UNI・UNI2-hはH-optimus-0やVirchowと比較すると検出エンコーダとしての性能面で見劣りすると指摘。
- **改善への言及**: 「パラメータ効率的ファインチューニング、特にLoRAの適用が次の自然なステップである」と提言(未実装・今後の課題)。

---

### <a id="paper-4"></a> [4] Parameter-efficient fine-tuning (PEFT) of Vision Foundation Models for Atypical Mitotic Figure Classification
- **著者**: Lavish Ramchandani, Gunjan Deotale, Dev Kumar Das
- **所属**: Aira Matrix Private Limited (ムンバイ)
- **掲載**: arXivプレプリント (2025-09-21投稿, v2: 2026-01-10)。MIDOG2025 Challenge Track2投稿手法。
- **リンク**: [arXiv:2509.16935](https://arxiv.org/abs/2509.16935)

#### 概要
Virchow, Virchow2, UNI, UNI2-hにLoRAを適用し、MIDOG2025の非定型有糸分裂像(atypical mitotic figure)分類に取り組んだ研究。本文中で「ヒト対イヌ腫瘍、異なるスキャナー・施設("human vs. canine tumors, different scanners and labs")」と明記されており、UNI2-hが直接イヌ組織画像に適用されている。

#### 手法のポイント
- 8グループ分割(施設・種・腫瘍タイプ等を跨ぐグループ交差検証と推測される)によるロバスト性評価。
- LoRAによるパラメータ効率的ファインチューニング。

#### 結果・貢献
- UNI(8グループ分割): 平均検証Balanced Accuracy **0.8037**、Preliminary Test Set **0.7847**。
- **限界**: 「グループ/ドメイン分割によってより大きなばらつきが露呈し、ドメインシフトの課題が浮き彫りになった("group/domain splits exposed greater variability, underscoring the challenge of domain shift")」。
- **改善への言及**: LoRAによるファインチューニングが線形プロービングに対して優位(定性的言及。具体的な線形プローブ比較数値は本文簡約版からは未取得)。

---

### <a id="paper-5"></a> [5] Self-supervised large-scale kidney abnormality detection in drug safety assessment studies
- **著者**: I. Slootweg, N. P. García-De-La-Puente, G. Litjens, S. Dammak
- **掲載**: arXivプレプリント (2025)
- **リンク**: [arXiv:2509.00131](https://arxiv.org/abs/2509.00131)

#### 概要
[topic 02](../../02_cross_species_pathology_fm/report.md)で既に発見済みの論文。UNIの特徴量を用いてラット腎臓の自己教師あり異常検知を行った、**毒性病理(創薬安全性評価)領域でUNIを動物組織に直接適用した唯一の事例**。今回の深掘り調査でも、これに追加・代替する毒性病理特化の事例は発見できなかった。

#### 結果・貢献
- AUC 0.62、NPV 89%。「実用に耐えるが完全ではない」水準。

---

### <a id="paper-6"></a> [6] Mitosis Detection in the Wild: Multi-Tumor and Context-Aware Generalization in the MIDOG 2025 Challenge
- **著者**: MIDOG 2025 Challenge運営チーム
- **掲載**: arXivプレプリント (2026)
- **リンク**: [arXiv:2606.07368](https://arxiv.org/abs/2606.07368)

#### 概要
MIDOG2025チャレンジ自体の総括論文。テストセットは**365症例、ヒト・イヌ・ネコの12腫瘍種**にまたがり、複数スキャナーでデジタイズされている。上位チームは検出F1 0.740、非定型分類Balanced Accuracy 0.908を達成したが、希少・多形性の高い腫瘍型では検出精度が一貫して低下したと報告。UNI/UNI2-hを含む基盤モデルが主要参加手法の共通要素になっている(論文1-4参照)。

---

### <a id="paper-7"></a> [7] Beyond the Failures: Rethinking Foundation Models in Pathology
- **掲載**: arXivプレプリント (2025-2026, v4まで改訂)
- **リンク**: [arXiv:2510.23807](https://arxiv.org/abs/2510.23807)

#### 概要
UNIを含む病理基盤モデル全般に対する批評論文。UNIを動物データへ直接適用してはいないが、「がん種・動物種(species)・教師あり度を同時に振ったクロスドメイン挙動の研究はほぼ手付かずのまま("The behavior of foundation models under cross-domain conditions, such as different cancer types or species, remains largely unexplored")」と明記しており、本調査の主要発見(=有糸分裂像分類という狭いニッチ以外では系統的な動物種検証がほぼ存在しない)を裏付ける傍証として引用。

---

### <a id="paper-8"></a> [8] Reporting transparency in veterinary pathology deep learning: A systematic review of reproducibility-critical details
- **著者**: Sweta Banerjee, Christof A. Bertram, Viktoria Weiss, et al.
- **掲載**: *Veterinary Pathology* (2026) [DOI:10.1177/03009858261459452](https://doi.org/10.1177/03009858261459452)
- **備考**: [topic 02](../../02_cross_species_pathology_fm/report.md)・[topic 10](../../10_glp_ai_validation_framework/report.md)で既収載の論文。本調査で発見した論文1〜3の著者(Banerjee, Bertram, Aubreville)と同一クラスタであることを確認する目的で再掲。獣医病理DL研究全体の再現性課題(コード公開率3%等)を指摘。
