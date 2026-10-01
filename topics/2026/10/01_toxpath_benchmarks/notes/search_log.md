# 調査メモ・検索ログ (Search & Investigation Log)

---

## 🧭 スコープ（ユーザー確認済み、2026-10-01）
- 対象：公開データセット、評価タスク・指標
- 動物範囲：非臨床毒性病理のみ（獣医臨床・ヒト病理は除外）
- 既存トピック（05 TG-GATEs、11 UNI動物転用）は参照・要約に留める
- 既存レポートのTRACE帰属の誤りは本レポートで明記するだけにし、既存ファイルは修正しない

---

## 🔍 検索クエリ履歴

調査は2つのサブエージェントで並行して行った（データセット担当、評価タスク・指標担当）。

| 日時 | ソース | 検索クエリ | ヒット件数 | 備考 |
|:---|:---|:---|:---|:---|
| 2026-10-01 | PubMed | `(toxicologic pathology OR nonclinical OR preclinical OR toxicity study) AND (deep learning OR AI OR ML) AND (rat OR rodent OR mouse OR dog OR monkey) AND (histopatholog* OR whole slide)`（2018–2026） | 178 | 主要な候補の抽出 |
| 2026-10-01 | PubMed | `(Toxicol Pathol OR J Toxicol Pathol OR Vet Pathol)[ta] AND (deep learning OR AI OR convolutional OR ML OR anomaly)` | 166 | 約35件を精読 |
| 2026-10-01 | PubMed | `TG-GATEs AND (deep learning OR neural network OR whole slide OR histopatholog*)` | 14 | |
| 2026-10-01 | PubMed | `(anomaly detection OR out-of-distribution) AND (toxicology OR preclinical OR nonclinical) AND (histopatholog* OR whole slide)` | 134 | 該当3件 |
| 2026-10-01 | Europe PMC | `"TG-GATEs" AND "whole slide"` | 11 | |
| 2026-10-01 | Europe PMC | (毒性病理 OR 非臨床) AND WSI AND (深層学習 OR AI) | 38 | |
| 2026-10-01 | Zenodo API | `TG-GATEs` / `toxicologic pathology whole slide` | ノイズ多数 | 関連は7541930、14617491、17514595 |
| 2026-10-01 | Web | TG-GATEs download / public rat tox WSI / NTP CEBS / DrugMatrix images / BigPicture access / MMO-Net / grand-challenge・Kaggle・IDR・STP challenge / monkey・dog・minipig tox WSI / 日本語・韓国語での検索 | 各約10件 | 公開WSIはTG-GATEs、MMO-Net、Graf、Zingmanのみ |
| 2026-10-01 | Web | Patho-Bench / eva / HEST の毒性タスク有無 | 各約10件 | いずれも毒性タスクなし |

**直接確認したページ**：dbarchive（画像ディレクトリの一覧を実際に取得）、Synapse API syn30282632、Zenodo各レコード、CEBS 15670、NTP Archives、bioRxiv TRACEの著者・所属ページ。

**取得できなかったもの**：Sage・ACSの全文（403）、OSFのページ（描画されない）、IHIのトピック草案PDF（SSLエラー）、PMCのPDF（ボット対策）。

---

## 🎯 選定・除外基準
- **採用**：毒性試験由来のWSIを使い、評価プロトコル（分割・参照標準・指標）が読み取れる一次研究。公開データセットの原典
- **除外**：MIDOG/CCMCTなど獣医腫瘍、ヒト病理、疾患モデルのみのデータ（隣接データとして別記）

---

## 💡 調査中の思考メモ
- 依頼時の前提（TRACE=Pfizer、Graf 2026=TRACE、MMO=Bayer）が一次情報と食い違った。2つのエージェントが独立に指摘し、TRACEについてはbioRxivの著者・所属ページで自分でも確認した。
- TG-GATEsのラベルは個体単位なので、スライド単位のタスクにはラベルノイズが入る（Slootwegが言及）。
