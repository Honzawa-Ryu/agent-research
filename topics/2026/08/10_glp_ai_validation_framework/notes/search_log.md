# 検索ログ・思考メモ

PROP-09「GLP規制環境下でのAIバリデーション・信頼性保証フレームワーク」調査の検索記録。

## 検索方針
本テーマはPROP-01〜08と異なり、査読論文よりも規制当局（FDA/PMDA/EMA）の公式ガイダンス文書が一次情報源として重要になる。WebSearch中心で該当ガイダンスのFDA.gov/EMA.europa.eu上の実URLを特定し、WebFetchで内容を精読する二段階アプローチを取った。

## 検索クエリと結果

| # | クエリ | ヒット数目安 | 採否・メモ |
|:---|:---|:---:|:---|
| 1 | FDA Good Machine Learning Practice GMLP guiding principles SaMD 2024 2025 | 9件 | 採用。IMDRF 2025最終版・FDA/HC/MHRA 2021初版を確認 |
| 2 | FDA Predetermined Change Control Plan PCCP artificial intelligence device | 8件 | 採用。2024年12月最終ガイダンスを確認 |
| 3 | artificial intelligence GLP toxicologic pathology validation FDA guidance | 7件 | 採用。Pohlmeyer-Esch et al. 2025を発見、本調査の中核文献に |
| 4 | Society of Toxicologic Pathology digital pathology AI validation whole slide imaging | 8件 | 採用。Thoolen et al. 2025、STP関連文書複数を確認 |
| 5 | PMDA 医薬品医療機器総合機構 AI 人工知能 プログラム医療機器 ガイドライン 承認審査 | 6件 | 参考。GLP毒性病理特化のPMDAガイダンスは不在と判断 |
| 6 | GLP electronic records audit trail AI negative triage normal slide screening pathology validation acceptance criteria | 9件 | 部分採用。陰性除外に特化した数値基準は未発見 |
| 7 | Rudmann Turner last mile toxicologic pathology AI validation regulatory acceptance NOAEL screening | 7件 | 採用（topics/01で既出文献の再確認） |
| 8 | Thoolen Bradley Stathonikos van Diest 2025 Toxicologic Pathology Forum whole slide images GLP compliance pubmed | 8件 | 採用。書誌情報を確定 |
| 9 | Society Toxicologic Pathology position paper pathology image data 21 CFR part 58 part 11 compliance whole slide imaging | 7件 | 採用。Tuomari et al. 2007を発見 |
| 10 | "Considerations for the Use of Artificial Intelligence" FDA guidance drug biological products PDF context of use credibility | 8件 | 採用。FDA 2025年1月ドラフトガイダンスの7段階信頼性評価枠組みを確認 |
| 11 | FDA GMLP 10 guiding principles machine learning medical device site:fda.gov | 9件 | 採用。fda.gov/media/153486, 173206, 179269の実PDFリンクを確定 |
| 12 | FDA "Use of Whole Slide Imaging in Nonclinical Toxicology Studies" guidance 2023 final PDF | 8件 | 採用。fda.gov/media/168431を確定（GLP・非臨床特化の数少ない確定ガイダンス） |
| 13 | EMA reflection paper artificial intelligence medicinal product lifecycle 2024 final | 8件 | 採用。2024年9月最終版を確認 |
| 14 | Tuomari Kemp "Society of Toxicologic Pathology Position Paper on Pathology Image Data" year 2007 | 7件 | 採用。著者・巻号・DOIを確定 |

## WebFetch実施ログ
- PMC12612283（Pohlmeyer-Esch 2025全文）: 成功。三段階同等性検証・ISPE成熟度モデル・21 CFR Part 11要件等を精読
- fda.gov guidance各種直リンク（GMLP/PCCP/Transparency）: 404エラーで直接WebFetch不可（FDA.govのbot対策と推測）。WebSearchのスニペットおよび複数の二次情報源の相互照合で内容を確認
- federalregister.gov（AI規制決定支援ドラフトガイダンス）: リダイレクトブロックのためWebFetch不可。WebSearchスニペットで代替確認

## 未解決・確認できなかった事項
- 「AIによる陰性除外（Normal Triage）」の具体的な感度/NPV数値基準を定めた規制文書・論文は発見できなかった
- PMDAのGLP毒性病理AI特化ガイダンスは不在と判断（一般的プログラム医療機器審査の枠組みのみ確認）
- 21 CFR Part 11の監査証跡要件とAIモデルバージョニングを明示的に統合したバリデーションプロトコルの実装事例（企業内SOP等）は非公開情報が多く一次文献では確認できなかった
