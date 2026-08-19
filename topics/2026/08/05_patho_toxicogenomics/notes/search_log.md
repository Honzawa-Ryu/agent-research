# 調査メモ・検索ログ (Search & Investigation Log)

本調査における検索クエリ、情報ソース、選定・除外基準、思考ログを記録します。

対象テーマ: [PROP-03] Open TG-GATEsを活用したトキシコゲノミクス×毒性病理WSIマルチモーダルAI（Patho-TGx）の先行研究とベンチマーク調査

---

## 🔍 検索クエリ履歴

| 日時 | ソース | 検索クエリ | ヒット件数（概算） | 採用件数 | 備考 |
|:---|:---|:---|:---|:---|:---|
| 2026-08-19 | Web検索 | `Open TG-GATEs deep learning histopathology gene expression prediction` | 8 | 0（背景理解） | Open TG-GATEsの基本構造、hist2RNA等一般病理領域のWSI→遺伝子発現予測の存在を確認 |
| 2026-08-19 | Web検索 | `"Open TG-GATEs" machine learning whole slide image` | 8 | 1 | PathologAI (Bussola et al. 2023) を特定 |
| 2026-08-19 | Web検索 | `PathologAI toxicogenomics deep learning liver kidney` | 7 | 1 | Att-RethinkNet (Su et al. 2022) を発見（遺伝子発現→多臓器病理所見の逆方向モデル） |
| 2026-08-19 | Web検索 | `virtual histopathology generation compound structure SMILES toxicity prediction` | 9 | 0（ギャップ確認） | SMILES→毒性スコア予測（QSAR系）は多数あるが、SMILES→病理組織像シミュレーションの実装は本調査時点で確認できず |
| 2026-08-19 | WebFetch | PMC10445282 フルテキスト精読 | - | 1 | PathologAIの詳細（アーキテクチャ、816WSI内訳、精度、トキシコゲノミクス統合の有無）を抽出 |
| 2026-08-19 | Web検索 | `multimodal deep learning integrate transcriptomics histopathology drug-induced liver injury` | 8 | 1 | ChemBioHepatox等のDILI予測モデル（画像非使用）、Mask R-CNN多所見検出モデルを確認 |
| 2026-08-19 | Web検索 | `"Att-RethinkNet" toxicogenomics multi-organ pathology prediction` | 8 | 1 | Att-RethinkNetの詳細（GitHub実装あり）を確認 |
| 2026-08-19 | Web検索 | `spatial transcriptomics toxicology drug safety histology` | 6 | 1 | Golfinos-Owens et al. 2026 (Frontiers in Toxicology) レビューを特定 |
| 2026-08-19 | WebFetch | Frontiers in Toxicology 2026 (10.3389/ftox.2026.1817521) 精読 | - | 1 | 空間トランスクリプトミクス×WSI統合の実装例（Cao 2024, Xin 2023, Nault 2023, Onoda 2022, Wijaya 2025等）を抽出 |
| 2026-08-19 | Web検索 | `Open TG-GATEs benchmark necrosis detection deep learning 2024 2025` | 8 | 1 | INSIGHT (Zhao et al. 2024) の存在を確認（Frontiers総説経由の二次情報） |
| 2026-08-19 | WebFetch | arXiv:2302.01653 (PDF) | - | 1 | バイナリ化け読み取り不能。arXiv abstractページから著者・概要を再取得 |
| 2026-08-19 | WebFetch | arXiv.org/abs/2302.01653 | - | 1 | Bertolini et al. (説明可能性フレームワーク, 前臨床病理MIL) の著者・概要を確認 |
| 2026-08-19 | WebFetch | Frontiers in Pharmacology 2026 (10.3389/fphar.2026.1907884) 精読 | - | 2 | INSIGHT詳細、および **GEESE (Jaume et al. 2024b)** の存在を発見（DOI 10.1101/2024.07.19.604355、macro-AUC 98.9%の言及） |
| 2026-08-19 | Web検索 | `Bussola Tong digital pathology animal study 2024 2025 follow-up toxicogenomics image` | 10 | 2 | FDA NCTR「AI4TOX」プログラム（AnimalGAN, SafetAI, BERTox, PathologAI）の全体像、TransTox論文の存在を確認 |
| 2026-08-19 | Web検索 | `"10.1101/2024.07.19.604355"` | 9 | 1 | GEESE論文の正式タイトル「AI-driven Discovery of Morphomolecular Signatures in Toxicology」(Jaume et al. 2024) を特定。PMC11291055あり |
| 2026-08-19 | Web検索 | `"GEESE" gene expression histology liver toxicology multiple instance learning` | 4 | - | GEESEの技術要旨（1,536遺伝子ターゲット回帰、TG-GATEs全156試験・10,234ペア）を確認 |
| 2026-08-19 | WebFetch | PMC11291055 フルテキスト精読 | - | 1 | GEESEの詳細（アーキテクチャ、データ分割、Pearson相関、病変分類AUC、限界）を抽出 |
| 2026-08-19 | WebFetch | biorxiv.org/content/10.1101/2024.07.19.604355v1 | - | - | 403 Forbidden（bot対策）。PMC版・検索要約で代替 |
| 2026-08-19 | Web検索 | `ToxGAN gene expression spatial pathology PathologAI Tong FDA` | 9 | - | PathologAI論文が将来構想として「ToxGAN×PathologAI統合による空間的遺伝子発現マッピング」を明記しているが未実装であることを確認（逆方向モデルの不在の直接証拠） |
| 2026-08-19 | Web検索 | `generative model predict histopathology image from compound structure drug toxicity diffusion` | 9 | 0（ギャップ確認） | 化合物構造→組織像生成の直接実装は毒性病理領域では未発見。High-content imaging分野のpDIFF/MorphDiff等は細胞形態（顕微鏡画像）が対象でWSI組織像ではない |
| 2026-08-19 | Web検索 | `"Bridging organ transcriptomics" Li Chen Tong npj Digital Medicine 2024 generative AI multiple organ toxicity` | 9 | 1 | TransTox (Li, Chen, Tong 2024, npj Digital Medicine) の詳細を確認 |
| 2026-08-19 | Web検索 | `AnimalGAN FDA generative adversarial network clinical pathology Nature Communications 2023` | 9 | 1 | AnimalGAN（血液生化学パラメータ生成、WSIではなく臨床病理検査値が対象）を関連文献として採用 |
| 2026-08-19 | Web検索 | `Open TG-GATEs whole slide image gene expression multimodal 2025 2026 foundation model toxicology` | 8 | 1 | TANGLE (Jaume et al. 2024, CVPR) の存在と、Frontiers総説による「toxicologic pathologyへの応用可能性」の指摘を確認 |
| 2026-08-19 | Web検索 | `TANGLE transcriptomics-guided slide representation learning Jaume CVPR 2024` | 9 | 1 | TANGLE公式情報（GitHub, CVPR2024, arXiv:2405.11618）を確認 |
| 2026-08-19 | WebFetch | arXiv.org/abs/2405.11618 | - | - | 事前学習データがヒト(Homo sapiens)とラット(Rattus norvegicus)の肝臓ペア(n=6,597)であることを確認。フロンティア1（動物種横断）との接続点として重要 |
| 2026-08-19 | Web検索 | `digital twin virtual toxicology pathology drug safety screening 2025` | 10 | 0（背景理解） | FDAのNAMs推進・AI規制ガイダンス動向を確認（PROP-09と重複するため本文では最小限の言及に留める） |
| 2026-08-19 | Bash/curl | オープンアクセスPDF取得試行（6件） | - | 4 | Su(Att-RethinkNet), Jaume(TANGLE), Jaume(GEESE), Li(TransTox) は取得成功。Bussola(PathologAI, PMCボット対策で失敗)とDing(AnimalGAN, 同様に失敗)はリンクのみで管理 |

---

## 🎯 論文の選定・除外基準
- **採用基準**:
  - 実際にWebSearch/WebFetchで内容を確認できたもの（書誌情報のみのハルシネーションは排除）
  - Open TG-GATEsまたは同種の毒性病理データを用いた実装、もしくはPatho-TGx（WSI⇔遺伝子発現/化合物構造の相互予測）に直接関係する主要理論・隣接技術
  - INSIGHT (Zhao et al. 2024) は一次ソース（原論文）に到達できなかったため、Frontiers総説を経由した二次情報として明示的に区別して引用
- **除外基準**:
  - タイトルのみでフルテキスト未確認のもの
  - 病理画像を一切使わない純粋なケモインフォマティクス/QSARモデル（ProTox, MolToxPred等）は背景文脈確認に留め、本文の主要引用からは除外

---

## 💡 調査中の思考メモ・ブレインストーミング
- **最大の発見**: 「順方向（WSI→遺伝子発現）」は **GEESE (Jaume et al. 2024)** によってTG-GATEs全156試験・10,234ペアという大規模スケールで既に実装され、病変分類macro-AUC 98.9%という高精度を達成している。フロンティア5で報告書01が「未開拓」と位置づけた技術は、2024年半ばの時点で（同一のMahmood研究室により）相当程度実現済みだったことが判明。ただし遺伝子発現「値」そのものの回帰精度は全遺伝子平均Pearson r=0.29と低く、「病変分類」と「連続値の遺伝子発現予測」の精度には大きな差がある点を報告書で強調する必要がある。
- **「逆方向」（化合物構造/TGx→仮想病理像生成）は依然として完全に未開拓**。PathologAI原著論文（Bussola et al. 2023）自身が将来構想として「ToxGANとの統合による空間的遺伝子発現マッピング」を明記しているが、2026年8月時点で後続の実装論文は確認できなかった。TransTox (Li et al. 2024) は臓器間（肝臓⇔腎臓）の遺伝子発現分布を生成AIで相互翻訳する点で「仮想毒性病理」の思想的先行例だが、出力はあくまで遺伝子発現ベクトルであり、画像（WSI）は生成しない。SMILES構造式を直接の入力とする病理画像生成モデルは、High-content imaging分野（pDIFF, MorphDiff等）には存在するが、対象は細胞形態（顕微鏡アッセイ画像）であり、WSIスケールの組織構造ではない。
- **意外な発見**: TANGLE (Jaume et al. 2024, CVPR) は肝臓の事前学習データとしてヒトとラットの両方（n=6,597ペア、2種）を用いており、意図せずして「フロンティア1（動物種横断）」と「フロンティア5（トキシコゲノミクス統合）」の両方を同時に部分的に満たす先行事例になっている。ただし論文自体は毒性病理を明示的な応用先として記述しておらず、Frontiers総説側がその可能性を指摘しているに留まる。PROP-01の既存レポートではこの論文は言及されておらず、本調査独自の追加発見。
- INSIGHT (Zhao et al. 2024) は原論文（PubMed/journal）に直接到達できず、Frontiers総説の引用表からの二次情報にとどまる。235遺伝子シグネチャ・AUC≈0.71という数値はTable記載値であり、原論文での前後文脈（検証方法の詳細等）は未確認。この限界を報告書に明記する。
