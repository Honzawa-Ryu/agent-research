# 調査メモ・検索ログ (Search & Investigation Log)

本調査（対照群Normativeモデルによる反事実的異常検知の生成AIアーキテクチャ深掘り、PROP-04）における検索クエリ、情報ソース、選定・除外基準、思考ログを記録します。

---

## 🔍 検索クエリ履歴

| 日時 | ソース | 検索クエリ | ヒット件数(概算) | 採用件数 | 備考 |
|:---|:---|:---|:---:|:---:|:---|
| 2026-08-19 | Web (arXiv/PubMed横断) | `counterfactual diffusion model anomaly detection brain MRI 2024 2025` | 10 | 2 | 反事実拡散モデルの主要系譜を特定 |
| 2026-08-19 | Web | `diffusion model unsupervised anomaly detection medical imaging survey 2025` | 6 | 1 | サーベイ論文(arXiv:2501.11430)を発見 |
| 2026-08-19 | Web | `normative modeling generative diffusion pathology whole slide image` | 8 | 0（病理生成AI全般の地図作りに利用、直接採用は次のクエリで） | WSI生成AI動向の概観 |
| 2026-08-19 | Web | `counterfactual synthesis toxicologic pathology background lesion control group generative` | 6 | 1 | **「toxicologic pathology」×「counterfactual」の直接ヒットは"What is Healthy?"論文のみ。毒性病理特化の反事実生成研究は本調査時点で皆無であることを確認** |
| 2026-08-19 | Web | `gigapixel whole slide image diffusion model computational cost patch-based generation histopathology` | 9 | 1 | WSIスケール拡張のための計算コスト対策技術（Harb et al.） |
| 2026-08-19 | Web | `rat strain spontaneous background lesion incidence chronic progressive nephropathy machine learning` | 8 | 1 | 系統別背景病変（CPN）のAI診断先行研究を発見。ただし生成/反事実ではなく検出・分割タスク |
| 2026-08-19 | Web | `Wolleb diffusion models for medical anomaly detection AnoDDPM MICCAI 2022` | 10 | 2 | 反事実拡散異常検知の2大源流論文を特定 |
| 2026-08-19 | Web | `vehicle control group deep learning anomaly detection preclinical toxicology histopathology 2025` | 8 | 0（新規）| Funk et al. 2025 / Graf et al. 2026等、topics/02で既収録の論文と重複確認 |
| 2026-08-19 | Web | `Pinaya normative modeling deep generative model brain covariate age diffusion autoencoder` | 9 | 0（背景理解用） | 条件付き正規モデル(cAE)の系譜を把握、個別採用は下記Bercea 2025に集約 |
| 2026-08-19 | Web | `f-AnoGAN normative model medical image unsupervised anomaly detection GAN Schlegl` | 10 | 0（背景理解用） | GAN系正規モデルの位置づけを比較表用に整理 |
| 2026-08-19 | Web | `ZoomLDM latent diffusion multi-scale gigapixel pathology generation efficient` | 9 | 1 | WSIスケール生成の計算効率化手法を特定 |
| 2026-08-19 | Web | `counterfactual explanation histopathology cancer generative model interpretability tumor` | 8 | 1 | **最重要発見: MoPaDi（Kather Lab, 2024）— 反事実拡散モデルを病理画像（ただし疾患病理）に直接適用した唯一の実証研究** |
| 2026-08-19 | Web (追加確認) | `MoPaDi morphing histopathology diffusion counterfactual explanation authors institution` / GitHub `KatherLab/mopadi` | - | - | 著者・手法詳細の確認 |
| 2026-08-19 | Web (追加確認) | `"Counterfactual MRI Generation" "Denoising Diffusion" Alzheimer's Disease Effect Detection arxiv authors` | 6 | 1 | 共変量（診断ラベル）条件付け生成の具体例として採用 |
| 2026-08-19 | Web (追加確認) | `"chronic progressive nephropathy" rat kidney AI deep learning YOLOv8 "Toxicological Research" 2024 Dice accuracy` | 8 | 1 | 系統特異的背景病変AIの定量結果を確認 |
| 2026-08-19 | Web (追加確認) | `Wyatt Leach Schmon Willcocks AnoDDPM simplex noise arxiv 2022` | 10 | 1 | AnoDDPMの書誌情報・OpenAccess PDF所在を確認 |
| 2026-08-19 | Web (追加確認) | `Evaluating normative representation learning generative AI anomaly detection brain imaging PMC` | 5 | 1 | 正規モデリングの評価指標という方法論的示唆を採用 |

---

## 🎯 論文の選定・除外基準
- **採用基準**:
  - 反事実拡散/GAN/正規モデリングの基礎・代表手法（査読済みトップ会議・ジャーナル）
  - 病理画像（WSI）またはギガピクセル画像への拡張性に直接言及する研究
  - 毒性病理・獣医病理の背景病変（Spontaneous Lesions）を扱う実証研究
- **除外基準**:
  - 反事実生成と無関係な単純な二値分類・セグメンテーションのみの論文（ただしCPN診断論文のみ「毒性病理の背景病変AIの現在地」を示す比較対象として例外的に採用）
  - 査読前・引用検証が困難な情報源のみに基づく主張

---

## 💡 調査中の思考メモ・ブレインストーミング
- **最大の発見**: 「毒性病理×反事実生成」で直接ヒットする論文は事実上ゼロ。一方、疾患病理では2024年にMoPaDi（KatherLab）が反事実拡散モデルを病理画像に実装済み。医用画像（脳MRI）の反事実異常検知は2022年のWollebらの研究を起点に成熟しつつある。つまり「技術の要素（反事実拡散 × 病理画像 × 対照群条件付け）」はすべて個別には存在するが、**3つを組み合わせた実装例は本調査時点で見つからなかった**。これはPROP-01（動物種横断FM）で確認されたのと同型の「未開拓」パターン。
- MoPaDiは疾患病理（TCGA等）を対象としており、"Control群"という概念そのものが存在しない（患者ごとの反事実であり、群比較ではない）。毒性病理の反事実生成が疾患病理と質的に異なる点は、真の「同一条件対照群」の実在データを条件付けに使える点。
- WSIスケールへの拡張（コア問い1）については、遺伝子疾患病理領域での「ギガピクセル生成」自体の研究（Harb et al., ZoomLDM）が計算コストの参考になるが、いずれも生成品質の忠実性評価が目的であり、異常検知の下流タスクでの検証はまだない。
- 系統別背景病変プロファイルの条件付け（コア問い2）については、CPN診断のような「検出・分類」レベルの研究はあるが、「系統をコンディションとして反事実生成に組み込む」実証は本調査では未発見。Dhinagar et al. (2024)の診断ラベル条件付けLDMが最も近い先行例（脳MRI・アルツハイマー病）。
