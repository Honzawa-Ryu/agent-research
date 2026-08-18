# フォーマット & 命名規則ガイド (Format Rules)

本リポジトリで作成されるすべてのファイル・ディレクトリは以下のルールに従います。

---

## 📁 命名規則 (Naming Conventions)

### 1. 調査ディレクトリ名
* パス: `topics/YYYY/MM/NN_slug/`
* `YYYY`: 4桁西暦（例: `2026`）
* `MM`: 2桁月（例: `08`）
* `NN`: 月内の2桁連番（`01`, `02`, ...）
* `slug`: 半角小文字英数とアンダースコア（スネークケース、最大30〜40文字程度）
* **例**: `topics/2026/08/01_pathology_ai_vision/`

### 2. PDFファイル名
* `papers/pdfs/YYYY_FirstAuthor_ShortTitle.pdf`
* **例**: `2024_Chen_HistoPathFM.pdf`, `2025_Smith_WSI_Segmentation.pdf`

### 3. 画像・アセット名
* `assets/figNN_description.png` または `assets/diagram_description.svg`
* **例**: `assets/fig01_wsi_pipeline.png`

---

## 🚫 1フォルダ10アイテム制限の維持ルール

1. **`topics/` 配下の階層化**:
   - `topics/` 直下は `YYYY/` のみ。
   - `topics/YYYY/` 直下は `MM/` (01〜12) のみ。※1年で最大12個
   - `topics/YYYY/MM/` 直下に各テーマが最大10個程度格納される。
2. **テーマフォルダ内の構造固定**:
   - 1つのテーマフォルダ直下には以下のファイル・フォルダのみを配置（合計4〜6個程度）:
     - `report.md`
     - `papers/`
     - `assets/`
     - `notes/`
3. **`papers/pdfs/` 内のファイル数**:
   - 1テーマあたりダウンロードする主要PDFは5〜10件程度にとどめる。
   - それ以上の大量の文献を扱う場合は `papers/index.md` でリンク管理するか、サブフォルダに分割する。

---

## 📝 Markdown 記法ルール

- **見出し**: `#`（H1）は文書のタイトルのみ1つ使用。以降は `##`（H2）、`###`（H3）を適切にネストする。
- **リンク**: ローカルファイルへのリンクは相対パスで正確に記載する。
- **テーブル**: 比較表や論文一覧は Markdown Table を使用して見通しを良くする。
- **ダイアグラム**: パイプラインやアーキテクチャは Mermaid 記法（```mermaid ... ```）を活用する。
- **コード・コマンド**: シンタックスハイライト付きのコードブロックを使用する。
