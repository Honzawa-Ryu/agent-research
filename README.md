# Agent Research Repository

AI Agentに文献調査・技術動向リサーチ・論文収集を委託し、構造化されたレポートを蓄積するためのリポジトリです。

---

## 📂 ディレクトリ構成

「**1つのフォルダに並列して10個以上のアイテムを並べない**」設計により、長期的な運用でも散らからない構造になっています。

```
agent-research/
├── .agent/                    # Agent向け調査指示書・フォーマット定義
│   ├── system_instructions.md # 調査の進め方・品質ガイドライン
│   └── format_rules.md        # 命名規則・1フォルダ10個制限ルール
│
├── templates/                 # 調査時に複製して使う雛形
│   ├── topic/                 # 1テーマごとの基本セット
│   │   ├── report.md          # 総合調査レポート雛形
│   │   ├── papers/index.md    # 論文メタデータ・リンク・要約
│   │   └── notes/search_log.md# 検索クエリ履歴・思考メモ
│   └── proposal_template.md   # テーマ提案用フォーマット
│
├── topics/                    # 実際の調査ディレクトリ（YYYY/MM/NN_slug）
│   └── 2026/
│       └── 08/
│           └── 01_pathology_ai_vision/
│               ├── report.md
│               ├── papers/ (index.md, pdfs/)
│               ├── assets/
│               └── notes/
│
├── proposals/                 # 次期調査テーマの提案・バックログ
│   ├── backlog.md             # 提案中・承認待ちテーマ一覧
│   └── archive/               # 調査完了/アーカイブされた提案
│
├── INDEX.md                   # 全調査テーマの総合目次
└── README.md                  # 本ファイル
```

---

## 🤖 Agentへの依頼方法（プロンプト例）

### 1. 新しいテーマの調査を依頼する場合
```markdown
【調査依頼】
以下のテーマについて調査し、レポートを作成してください。

テーマ: 「病理画像分野における画像認識AIの活用と最新研究」
条件:
- .agent/system_instructions.md のワークフローに従うこと。
- templates/topic/ をコピーして topics/YYYY/MM/ 配下に新しいディレクトリを作成すること。
- 重要論文の書誌情報、リンク、要約を papers/index.md に整理し、可能ならPDFを保存すること。
- 調査完了後、リポジトリ直下の INDEX.md に追記すること。
```

### 2. 過去の調査を踏まえて次のテーマを自動提案してもらう場合
```markdown
【テーマ提案依頼】
これまでに調査したレポート（INDEX.md および各 report.md の「今後の展望・オープンクエスチョン」）を読み込み、
次に深掘り・調査すべき有望なテーマを3つ提案してください。
提案は templates/proposal_template.md のフォーマットに従い、proposals/backlog.md に追記してください。
```

### 3. 提案されたテーマから調査を開始する場合
```markdown
【提案テーマの着手】
proposals/backlog.md にある [PROP-01] のテーマについて調査を開始してください。
ステータスを「調査中」に更新し、新規トピックディレクトリを作成して調査を進めてください。
```

---

## 📌 運用ルール
- **1フォルダ10個制限**: 調査テーマは月単位（`topics/YYYY/MM/`）で分割し、1ディレクトリあたり10件以内に収めます。
- **再現性**: どんなクエリで文献を探したかを `notes/search_log.md` に残します。
- **インデックス更新**: レポート作成時は必ず `INDEX.md` を更新して一覧性を保ちます。
