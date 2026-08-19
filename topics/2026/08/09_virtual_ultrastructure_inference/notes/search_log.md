# 調査メモ・検索ログ (Search & Investigation Log)

本調査における検索クエリ、情報ソース、選定・除外基準、思考ログを記録します。

対象テーマ: [PROP-08] 光顕WSIからの微小構造推定（Virtual Ultrastructure / Sub-resolution Toxicity Inference）の実現可能性調査

---

## 🔍 検索クエリ履歴

| 日時 | ソース | 検索クエリ | ヒット件数(概算) | 採用件数 | 備考 |
|:---|:---|:---|:---|:---|:---|
| 2026-08-19 | Web (bioRxiv/Nature/PubMed横断) | `label-free prediction fluorescence microscopy transmitted light deep learning Ounkomol Christiansen In Silico Labeling` | 10 | 2 | Label-free predictionの基礎2論文(Ounkomol 2018, Christiansen 2018)を特定 |
| 2026-08-19 | Web | `virtual staining light microscopy to electron microscopy deep learning prediction ultrastructure` | 10 | 1 | EMDiffuse(EM画像拡張、光顕入力なし)を発見 |
| 2026-08-19 | Web | `image-to-image translation H&E to electron microscopy generative model histology` | 10 | 0 | H&E→EMの直接変換研究は不在。H&E↔IHC等の光顕内変換のみヒット |
| 2026-08-19 | Web | `toxicologic pathology mitochondrial degeneration ultrastructure deep learning prediction light microscopy` | 9 | 0 | ミトコンドリア毒性のDL研究はCell Painting等の細胞アッセイ中心で、WSI由来ではない |
| 2026-08-19 | Web | `EMDiffuse diffusion model electron microscopy super-resolution volume EM prediction` | 10 | 1 | EMDiffuseの詳細確認、EM画像のみを扱うことを確認 |
| 2026-08-19 | Web | `correlative light and electron microscopy CLEM registration deep learning alignment same section` | 9 | 2 | DeepCLEM, CLEM-Regを発見。いずれも位置合わせ目的でEM→蛍光の粗い予測のみ |
| 2026-08-19 | Web | `predicting electron microscopy image from fluorescence light microscopy cross-modality deep learning generative` | 9 | 1 | Light My Cellsデータセット(ミトコンドリア含む4構造)を発見 |
| 2026-08-19 | Web | `virtual staining review histology deep learning Rivenson Ozcan digital pathology light microscopy` | 10 | 3 | Rivenson 2019(virtual H&E)、Kreiss 2023レビュー、Zhang 2025拡散モデル超解像を発見 |
| 2026-08-19 | Web | `predicting ultrastructure electron microscopy from histopathology light microscopy same tissue deep learning feasibility` | 8 | 0 | 「同一組織のHE→EM予測」の直接研究は不在を再確認 |
| 2026-08-19 | Web | `"Light My Cells" dataset transmitted light fluorescence prediction mitochondria organelle` | 9 | 0(既出情報の補強) | データセット規模・対象構造の詳細確認 |
| 2026-08-19 | Web | `phospholipidosis electron microscopy toxicologic pathology regulatory drug induced ultrastructural evaluation` | 7 | 1 | EMが実際に規制毒性評価で使われる代表例(薬剤性リン脂質症)を確認、Anderson & Borlak 2006を採用 |
| 2026-08-19 | Web | `Christiansen 2018 Cell In Silico Labeling predict electron microscopy input immunofluorescence prediction` | 10 | 0 | 検索要約が「EM入力からIF予測」と誤った内容を示したため、GitHub READMEで直接検証し否定（transmitted-lightのみが入力） |
| 2026-08-19 | Web | `retrospective electron microscopy formalin fixed paraffin embedded FFPE tissue toxicologic pathology limitation fixation` | 8 | 0(文脈情報として活用) | FFPEブロックからの遡及的EM再処理は可能だが元の固定品質に強く依存、という制約を確認 |
| 2026-08-19 | Web | `"virtual electron microscopy" OR "virtual EM" histopathology toxicology drug safety deep learning` | 8 | 0 | 「毒性病理×virtual EM」の直接研究は不在を再確認。TRACE等の既存毒性病理AIはEM非対応 |
| 2026-08-19 | Web | `paired hematoxylin eosin transmission electron microscopy dataset public toxicology histopathology correlative` | 8 | 0 | 公開HE-TEMペアデータセットは発見できず(オープンクエスチョンとして記録) |
| 2026-08-19 | Web | `super-resolution histopathology whole slide image deep learning nanoscale texture inference limit` | 9 | 0(文脈情報) | WSI超解像(ISTE等)は光顕内の解像度向上に留まり、EMスケールへの外挿ではないことを確認 |
| 2026-08-19 | WebFetch (一次資料確認) | Cell論文PDF直接アクセス→403のためGitHub README確認 | - | - | Christiansen 2018の入力モダリティを一次情報に近い形で確認 |
| 2026-08-19 | WebFetch (一次資料確認) | PMC (EMDiffuse, DeepCLEM, CLEM-Reg) | - | - | 著者・DOI・定量結果の一次確認 |

---

## 🎯 論文の選定・除外基準
- **採用基準**:
  - Label-free prediction / Virtual staining / EM超解像 / CLEM登録という4つの隣接技術領域それぞれの代表的・査読済み研究
  - WebSearch/WebFetchで一次情報（論文本文・公式GitHub・PMC）まで遡って内容を確認できたもの
  - 毒性病理におけるEMの実運用文脈（薬剤性リン脂質症等）を示す論文
- **除外基準**:
  - 検索エンジンの要約のみに基づく未検証の主張（例: 「Christiansen 2018がEM入力からIF予測を行う」という誤情報は一次情報で否定し不採用）
  - ペイウォールで本文確認ができず、書誌情報のみしか得られない周辺論文

---

## 💡 調査中の思考メモ・ブレインストーミング
- **核心的な否定的知見**: 「HE-WSI（光顕）→TEM相当の微細構造（光顕の回折限界を超える解像度）」を直接予測する研究は、本調査の範囲では一件も発見できなかった。存在するのはいずれも以下のいずれかである：
  1. EM画像同士の強化（デノイズ・超解像・等方化）＝ EMDiffuse等。入力は既にEMであり、光顕からの外挿ではない。
  2. CLEM位置合わせのための粗い相互予測（EM→蛍光、蛍光→EM）＝ DeepCLEM, CLEM-Reg。目的は位置合わせであり、微細構造の忠実な再構成ではない。
  3. 光顕内でのモダリティ変換（無染色→HE、HE→IHC等）＝ Rivenson 2019, Light My Cells等。解像度は光顕の回折限界内に留まる。
- 検索エンジンの自動要約に、実際の論文内容と異なる主張（Christiansen 2018がEM入力を扱うという誤り）が含まれていたため、必ずGitHub README等の一次情報で裏取りする方針を徹底した。他の論文についても同様のリスクがあることを念頭に置くべき。
- 毒性病理特有の論点として、FFPE（ホルマリン固定パラフィン包埋）ブロックは標準的なTEM前処理（グルタルアルデヒド/オスミウム固定）と異なるため、遡及的にEMを行っても品質が元の固定条件に強く依存するという制約がある。これは「HE-WSIとTEMのペアデータをどう収集するか」というコア問い2に直結する重要な制約。
