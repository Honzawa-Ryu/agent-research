# 調査メモ・検索ログ (Search & Investigation Log)

本調査における検索クエリ、情報ソース、選定・除外基準、思考ログを記録します。

対象範囲: UNI (Chen et al. 2024, Nature Medicine) および UNI2/UNI2-h (MahmoodLab, 2025公開)
を「ヒト以外の動物組織画像」に適用した一次研究。[02_cross_species_pathology_fm](../../02_cross_species_pathology_fm/report.md)
はテーマを「動物種横断病理基盤モデル」全般に広く取ったが、本調査は **UNI/UNI2という具体的なモデル名**
に絞り込み、02では拾いきれなかった隣接クラスタ（有糸分裂像分類コミュニティ）を掘り下げる。

---

## 🔍 検索クエリ履歴

| 日時 | ソース | 検索クエリ | ヒット件数(概算) | 採用件数 | 備考 |
|:---|:---|:---|:---|:---|:---|
| 2026-08-20 | Web検索 | `"UNI" pathology foundation model veterinary animal histopathology zero-shot` | 10 | 0 | 直接ヒットなし。獣医病理レビューの存在のみ確認 |
| 2026-08-20 | Web検索 | `UNI2-h foundation model animal histopathology 2025 2026` | 8 | 1 | UNI2-hの技術仕様(1536次元embedding)を確認 |
| 2026-08-20 | Web検索 | `Chen 2024 "UNI" pathology foundation model cited rodent OR canine OR "non-human primate" domain shift` | 10+10+10 | 0 | UNI原著論文自体には動物種への直接言及なし(想定通り) |
| 2026-08-20 | Web検索 | `pathology foundation model UNI Virchow CONCH toxicologic pathology rat mouse benchmark` | 9 | 0 | ヒトがん病理ベンチマークが中心、毒性病理特化のものは未発見 |
| 2026-08-20 | Web検索 | `"Benchmarking Foundation Models for Mitotic Figure Classification" UNI canine` | 9 | 1 | **主要発見**: CCMCT(イヌ肥満細胞腫)データセットでUNI等をベンチマーク |
| 2026-08-20 | Web検索 | `MIDOG challenge foundation model UNI embeddings canine domain generalization mitosis` | 10 | 2 | MIDOGデータセットがヒト・イヌ・ネコ混在の多種データセットと判明 |
| 2026-08-20 | Web検索 | `canine mammary tumor OR "canine cutaneous mast cell tumor" pathology foundation model UNI feature extraction` | 9 | 0 | CMT分野は既存CNN中心、UNI直接使用例はこのクエリでは未発見 |
| 2026-08-20 | Web検索 | `Aubreville Bertram Klopfleisch foundation model pathology veterinary UNI Virchow` | 6 | 0(既知論文確認) | 獣医病理DL研究の中心人物クラスタを特定、既知の系統的レビュー論文を再確認 |
| 2026-08-20 | WebFetch | arxiv.org/abs/2508.04441 (概要ページ) | - | - | 抄録のみ、詳細は本文取得が必要と判明 |
| 2026-08-20 | WebFetch | arxiv.org/abs/2509.16935 (概要ページ) | - | - | 同上 |
| 2026-08-20 | WebFetch | arxiv.org/html/2412.06365 | - | 1 | **主要発見**: UNIのCCMCT/MIDOGでのIn-domain/Out-of-domain AUROC数値を取得 |
| 2026-08-20 | WebFetch | arxiv.org/pdf/2508.04441 | - | 1 | UNI linear probe vs LoRA数値(CCMCT/MIDOG2022)を取得 |
| 2026-08-20 | WebFetch | arxiv.org/pdf/2506.21444 | - | 0(本文からは数値抽出不可) | Banerjee et al. 2026 MELBA、犬データセット使用は確認したがUNI固有数値は本文断片から抽出できず |
| 2026-08-20 | WebFetch | arxiv.org/html/2607.28007v1 | - | 1 | **主要発見**: UNI/UNI2-hを検出エンコーダとして使用、MIDOG++/TUPAC16数値取得 |
| 2026-08-20 | Web検索 | `"A Comprehensive Benchmark of Histopathology Foundation Models for Kidney" arXiv 2603.15967 rat animal` | 8 | 0 | ヒトの非がん性腎疾患(CKD)ベンチマークでスコープ外(動物データ不使用) |
| 2026-08-20 | Web検索 | `pathology foundation model UNI feline equine wildlife histopathology application` | 9 | 0 | ネコ・ウマ・野生動物への単独適用例は未発見(MIDOG2025でネコは混在データとして間接的に含まれる) |
| 2026-08-20 | Web検索 | `"Ensemble of Pathology Foundation Models for MIDOG 2025" UNI2-h atypical mitosis canine feline` | 7 | 1(参考) | MIDOG2025 Track2のPFM+LoRAアンサンブル手法を確認(UNI2-h使用は示唆だが本文未確認のため参考扱い) |
| 2026-08-20 | Web検索 | `"Foundation Model-Driven Classification of Atypical Mitotic Figures with Domain-Aware Training Strategies" UNI` | 7 | 0(直接UNIではなくH-optimus-0中心と判明) | スコープ外として除外 |
| 2026-08-20 | WebFetch | arxiv.org/html/2509.16935v2 | - | 1 | UNI2-hのMIDOG2025数値・「human vs. canine tumors」の明記を取得 |
| 2026-08-20 | Web検索 | `MIDOG 2025 challenge dataset species human canine feline tumor types atypical mitosis track` | 10 | 1 | **確認**: MIDOG2025テストセットは365症例・ヒト/イヌ/ネコ12腫瘍型 |
| 2026-08-20 | Web検索 | `"UNI" foundation model embeddings toxicologic pathology drug safety rodent liver kidney histopathology` | 9 | 0 | 毒性病理特化でのUNI使用例は追加発見なし(Slootweg 2025が唯一のまま) |
| 2026-08-20 | Web検索 | `pathology foundation model UNI wildlife zoo veterinary diagnostic histopathology 2025` | 10 | 0 | 野生動物・動物園病理への適用例は未発見 |
| 2026-08-20 | Web検索 | `"Beyond the Failures" "Rethinking Foundation Models in Pathology" arXiv 2510.23807 domain shift species` | 8 | 1(参考) | 「がん種×動物種×教師あり度を同時に振った研究は存在しない」という業界的自己認識を確認 |
| 2026-08-20 | WebFetch | arxiv.org/abs/2607.22861 | - | 0 | Robustifying pathology FM via fine-tuning、動物種への言及なしを確認しスコープ外 |
| 2026-08-20 | Web検索(日本語) | `UNI病理基盤モデル 動物 マウス ラット イヌ 適用 限界 論文` | 7+10 | 0 | 日本語文献での直接報告は未発見(英語論文が情報源の中心) |
| 2026-08-20 | WebFetch | arxiv.org/abs/2412.06365, /abs/2607.28007, /abs/2508.04441, /abs/2509.16935 | - | - | 書誌情報(著者・所属・投稿日・DOI)の確認用 |

---

## 🎯 論文の選定・除外基準

- **採用基準**:
  - UNIまたはUNI2/UNI2-hというモデル名を明示的に使用し、かつ対象データにヒト以外の動物(イヌ・ネコ・ラット等)由来の病理組織画像が含まれることが本文で確認できるもの。
  - 査読誌掲載(MELBA等)、または著名な国際チャレンジ(MIDOG 2022/2025)関連のarXivプレプリント。
- **除外基準**:
  - UNI/UNI2という名称を使わず、他の基盤モデル(CPath-CLIP, DINOv2素のもの等)のみを動物データに適用した研究 → [02_cross_species_pathology_fm](../../02_cross_species_pathology_fm/report.md)側で既に整理済みのためここでは参照のみに留める。
  - ヒトの非腫瘍性疾患(例: ヒトCKD)ベンチマークで動物データを含まないもの。
  - UNI原著論文(Chen et al. 2024)自体・UNI2のモデルカード等、一次研究ではなく基盤モデルそのものの記述。

---

## 💡 調査中の思考メモ・ブレインストーミング

- **最大の気付き**: [02_cross_species_pathology_fm](../../02_cross_species_pathology_fm/report.md)は「UNIを動物に直接適用した実証研究はSlootweg et al. 2025(ラット腎臓)の1件のみ」と結論していたが、これは調査スコープが広すぎて特定の研究コミュニティを見落としていたことが判明。**有糸分裂像(mitotic figure)分類・検出**というニッチなタスクにおいて、MIDOG/CCMCTという「ヒト・イヌ・ネコ混在」の公開ベンチマークを使う研究コミュニティ(Aubreville, Bertram, Ganz, Ammeling, Banerjee ら、独Ingolstadt工科大学・ウィーン獣医大学が中心)が2024年末以降、UNI/UNI2-hを継続的に評価対象に含めている。これは腫瘍病理(がん)領域における「量的評価に強いモデル」という位置づけであり、毒性病理(非腫瘍性・びまん性病変)とは病変の性質が異なる点に注意。
- MIDOGデータセット自体は2021年設計だがUNIは2024年公開のため、UNIを使った再解析は2024年末以降の論文群に限られる。
- UNI2-hはUNI(v1)より動物データでの性能が明確に高い傾向(例: Banerjee 2026のMIDOG++/TUPAC16検出タスクでF1が大幅改善)。ただし直接比較したのはこの1本のみで、他のFM(H-optimus-0, Virchow2)がUNI系列を上回るとの報告も複数あり、「UNI/UNI2が動物データで最良」というわけではない。
- LoRA(パラメータ効率的ファインチューニング)は複数の独立した論文で共通して「ドメイン外(=種を跨ぐ場合を含む)性能を大幅に改善する」との一致した結果が出ている。これは[02](../../02_cross_species_pathology_fm/report.md)で整理した「PEFT系統」の主張と整合的。
- 一方で、Ganz et al. 2024の「FMベース分類器はEnd-to-end学習したResNet50より頑健というわけではなかった」という指摘は、基盤モデル信奉に対する重要な反証であり、報告書で強調する価値がある。
- 毒性病理(非腫瘍性病変)特化でのUNI利用はSlootweg 2025のみで変わらず。今回の深掘りでも新規発見はなかった。この点は02の結論を覆さない。
