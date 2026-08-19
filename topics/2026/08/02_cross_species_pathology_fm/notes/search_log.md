# 調査メモ・検索ログ (Search & Investigation Log)

本調査（動物種横断病理基盤モデル / Cross-Species Pathology Foundation Models）における検索クエリ、情報ソース、選定・除外基準、思考ログを記録します。

---

## 🔍 検索クエリ履歴

| 日時 | ソース (arXiv / PubMed / Google Scholar 等) | 検索クエリ | ヒット件数(概算) | 採用件数 | 備考 |
|:---|:---|:---|:---:|:---:|:---|
| 2026-08-19 | Web検索 (arXiv中心) | `cross-species foundation model digital pathology mouse rat whole slide image 2025 2026` | 9 | 0 | ヒト疾患病理FM一般の記事が中心。動物種横断特化のFMは未ヒット |
| 2026-08-19 | Web検索 | `veterinary pathology foundation model deep learning whole slide image self-supervised` | 10 | 0 | ヒト向けFM（TITAN, PathVQ等）中心。獣医特化FMは未発見 |
| 2026-08-19 | Web検索 | `domain adaptation histopathology species human mouse rat pathology foundation model fine-tuning` | 9 | 2 | **Lost in Translation (Arora 2026)**, PathoTune を発見 |
| 2026-08-19 | Web検索 | `toxicologic pathology foundation model rat NTP archive artificial intelligence 2025` | 8 | 1 | Pohlmeyer-Esch et al. 2025 (Toxicologic Pathology) レビューを発見 |
| 2026-08-19 | Web検索 | `species-invariant representation learning histology disentanglement` | 8 | 0 | 種横断ではなく染色・施設間ドメイン不変性の研究が中心（参考情報として保持） |
| 2026-08-19 | WebFetch | arXiv:2603.04405 (Lost in Translation) 詳細 | - | 1 | Semantic Anchoring手法・実験数値を精読 |
| 2026-08-19 | Web検索 | `"Lost in Translation" cross-species pathology language re-aligns vision arxiv` | 6 | - | 同論文の追加情報確認（著者はThomas Jefferson High School在籍の独立研究者、査読前プレプリント） |
| 2026-08-19 | Web検索 | `National Toxicology Program digitized slide archive artificial intelligence deep learning rodent histopathology` | 7 | 1 | PathologAI (Bussola et al. 2023) を発見 |
| 2026-08-19 | Web検索 | `STP Society Toxicologic Pathology AI special interest group cross-species 2025 2026` | 10 | 0 | STP学会の一般情報のみ。専用SIGの一次資料は未発見 |
| 2026-08-19 | Web検索 | `human pathology foundation model UNI Virchow domain shift animal tissue rodent evaluation benchmark` | 9 | 0 | ヒトFM一般のサーベイ中心。動物組織への直接ベンチマークは未発見（ギャップの証左） |
| 2026-08-19 | Web検索 | `PathoTune parameter efficient fine-tuning pathology foundation model prompt tuning` | 9 | 1 | PathoTune (MICCAI 2024) の詳細確認 |
| 2026-08-19 | Web検索 | `comparative pathology evolutionary conserved lesion morphology cross species deep learning veterinary` | 7 | 1 | **panspecies.ai (AbdulJabbar & Castillo et al. 2023, Nature Communications)** を発見。morphospace overlap指標 |
| 2026-08-19 | Web検索 | `INHAND ontology natural language processing text embedding toxicologic pathology lesion nomenclature` | 6 | 0 | INHAND自体の解説記事中心。テキスト埋め込みへの応用例は未発見（PROP-05で深掘り予定） |
| 2026-08-19 | Web検索 | `digital pathology AI dog canine comparative oncology cross-species tumor foundation model 2025` | 8 | 0 (重複) | panspecies.ai, Lost in Translation の再確認 |
| 2026-08-19 | WebFetch | PMC10133243 (panspecies.ai論文) 詳細 | - | 1 | アーキテクチャ・morphospace overlap計算法・精度要因を精読 |
| 2026-08-19 | WebFetch | PMC12612283 (Pohlmeyer-Esch 2025) 詳細 | - | 1 | 規制動向（FDA GMLPドラフト2025-01等）を精読 |
| 2026-08-19 | Web検索 | `rodent liver kidney whole slide image pretrained vision transformer self-supervised foundation model preclinical toxicology` | 10 | 3 | **kidney abnormality detection (Slootweg 2025)**, **Mahalanobis anomaly detection (Graf 2026)**, **attention-guided weak supervision (Sci Rep 2025)** を発見 |
| 2026-08-19 | Web検索 | `LoRA low-rank adaptation cross-species domain shift histopathology veterinary few-shot 2025 2026` | 7 | 0 | 種横断×LoRAの直接研究は未発見（未開拓であることの傍証） |
| 2026-08-19 | WebFetch | arXiv:2509.00131 (kidney SSL) 詳細 | - | 1 | UNI特徴量を用いたラット腎臓異常検知の手法・数値を確認 |
| 2026-08-19 | Web検索 | `"attention-guided" weak supervision rat liver whole slide image toxicologic pathology Scientific Reports 2025 pilot study` | 10 | 2 | Funk et al. 2025 (MIL vs ViT) を追加発見 |
| 2026-08-19 | Web検索 | `"class-aware Mahalanobis distance" preclinical histopathology toxicity anomaly detection` | 7 | 0 (重複) | Graf et al. 論文の別URL確認 |
| 2026-08-19 | WebFetch | arXiv:2602.02124v2 (Mahalanobis) 詳細 | - | 1 | DINOv2+LoRA手法・マウス肝臓データセット詳細を精読 |
| 2026-08-19 | Web検索 | `Funk Clement Togninalli visual transformer multiple instance learning rat liver histopathologic lesions Roche 2025` | 9 | 1 | Funk et al. 2025 の詳細（著者所属Roche/Genentech、データ規模）を確認 |
| 2026-08-19 | WebFetch | journals.sagepub.com (Funk et al. 2025) 詳細 | - | 1 | MIL vs ViT AUROC比較表を取得 |
| 2026-08-19 | Web検索 | `Banerjee Bertram "Reporting transparency in veterinary pathology deep learning" abstract species reproducibility` | 7 | 1 | 獣医病理DL研究の再現性系統的レビュー (2026) を発見 |

---

## 🎯 論文の選定・除外基準

- **採用基準**:
  1. 動物種横断（ヒト⇄動物、動物種間）の病理形態表現学習・ドメイン適応に直接関係する研究。
  2. ヒト病理基盤モデル（UNI, Virchow, DINOv2等）を動物・毒性病理データへ転移・ファインチューニングした実証研究（2025〜2026年の最新事例を優先）。
  3. 種横断病理AIの実現可能性を左右する周辺トピック（形態保存性の定量化、規制動向、再現性の課題）。
  4. 実在確認できたもの（arXiv/DOI/PMCで内容を精読できたもの）のみ採用し、要約のみで実体確認できない文献は採用しない。
- **除外基準**:
  1. ヒト単一種の疾患病理基盤モデル一般（UNI/CONCH/Virchow自体の詳細）は、topics/2026/08/01で既出のため重複掲載しない（本レポートでは背景情報としてのみ簡潔に言及）。
  2. 染色・施設間ドメインシフト（種横断ではない）に関する研究は、直接的な関連度が低いため参考情報に留め、papers/index.mdには非掲載。
  3. 放射線画像（X線等）ベースの種横断研究は、WSI/組織形態とモダリティが異なるため除外。

---

## 💡 調査中の思考メモ・ブレインストーミング

### 1. 「動物種横断病理基盤モデル」はまだ誰も本格的に作っていない
- 検索を重ねる中で明確になったのは、「ヒト病理基盤モデル（UNI/Virchow/CONCH）をそのまま動物病理に転移する」という直球のアプローチを扱った文献がほとんど存在しないという事実そのものが、PROP-01の提案（未開拓フロンティア）を裏付けている。
- 実際に見つかった毒性病理AI研究（Funk et al. 2025, Graf et al. 2026, 攻撃guided weak supervision Sci Rep 2025）は、いずれも **ImageNet事前学習**または**自然画像で学習したDINOv2**をバックボーンとしており、ヒト病理特化基盤モデル（UNI等）を出発点にした例は kidney abnormality detection (Slootweg et al. 2025) の1件のみで、AUC 0.62・NPVは89%と、限定的な性能に留まっている。これは「ヒト病理FMの動物への転移はまだ体系的に検証されていない」ことの直接的な証拠。

### 2. 「進化的相同性アライメント」の最有力な先行研究は、実は"がん"ではなく"腫瘍細胞形態"の汎種比較から来ている
- panspecies.ai (AbdulJabbar & Castillo et al. 2023) の "morphospace overlap" 指標は、フロンティア1で構想していた「種不変特徴 / 種特異的特徴の分離」を定量評価する具体的な枠組みとして極めて有用。哺乳類の上皮腫瘍で精度0.79〜0.94、円形細胞腫瘍（リンパ腫等）で0.57〜0.70に低下するという知見は、「病変タイプによって種横断転移の難易度が大きく異なる」ことを示しており、毒性病理（肝細胞肥大・壊死等の非腫瘍性病変が中心）への外挿時に要注意。

### 3. Semantic Anchoring（Lost in Translation, Arora 2026）は着想として面白いが、査読前・単著プレプリントである点に留意
- 著者は高校生の独立研究者（Thomas Jefferson High School）で、査読済み論文ではない。技術的着想（凍結視覚バックボーン＋テキストアンカーで「種による意味崩壊」を補正）は動物種横断病理FMのフロンティアと直結し、示唆に富むが、本調査ではこの限界を明記した上で「有望なアイデアの一つ」として扱う。

### 4. 次に深掘りすべき点
- INHAND用語体系のテキスト埋め込みと種横断アライメントの統合（PROP-05と接続）。
- panspecies.aiのmorphospace overlap指標を毒性病理の非腫瘍性病変（肥大・空胞変性・壊死等）に拡張できるか。
- LoRA/PEFTによる少量動物データでの効率的アライメント（Graf et al. 2026のLoRA実装は同一種内タスクだが、種横断への拡張余地は大きい）。
