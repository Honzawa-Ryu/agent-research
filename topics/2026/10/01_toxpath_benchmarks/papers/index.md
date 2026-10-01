# 収集論文・リソース一覧

毒性病理WSIの公開データセットと、評価タスク・指標・分割プロトコルの把握に使った論文・リソースです。詳細な比較は [report.md](../report.md) の §1・§3 を参照してください。

---

## 📚 論文メタデータ一覧

| No. | タイトル / 名称 | 著者 / 年 / 掲載 | リンク | ローカルPDF | 位置づけ | 重要度 |
|:---|:---|:---|:---|:---|:---|:---:|
| 1 | Open TG-GATEs: a large-scale toxicogenomics database | Igarashi et al. 2015, NAR 43:D921 | [DOI](https://doi.org/10.1093/nar/gku955) / [PMC4384023](https://pmc.ncbi.nlm.nih.gov/articles/PMC4384023/) | — | 主要公開データセットの原典 | ★★★ |
| 2 | Deep Learning-based Modeling for Preclinical Drug Safety Assessment (TRACE) | Jaume, de Brot, ..., Mahmood 2024, bioRxiv | [DOI](https://doi.org/10.1101/2024.07.20.604430) | [PDF](pdfs/2024_Jaume_TRACE_ToxFoundation.pdf) | TG-GATEs最大規模の利用。試験単位分割 | ★★★ |
| 3 | Transcriptomics-guided Slide Representation Learning (TANGLE) | Jaume et al. 2024, CVPR | [arXiv:2405.11618](https://arxiv.org/abs/2405.11618) | [PDF](pdfs/2024_Jaume_TANGLE.pdf) | TG-GATEs肝のfew-shot評価 | ★★☆ |
| 4 | Self-supervised large-scale kidney abnormality detection in drug safety assessment studies | Slootweg et al. 2025, arXiv | [arXiv:2509.00131](https://arxiv.org/abs/2509.00131) | [PDF](pdfs/2025_Slootweg_SSL_KidneyAnomaly.pdf) | 化合物単位分割の異常検知。NPVを報告 | ★★★ |
| 5 | Mahalanobis型 異常セグメンテーション・OOD検出（マウス肝） | Graf et al. 2026, Sci Rep (BI/Tübingen) | [DOI](https://doi.org/10.1038/s41598-026-56510-9) / [arXiv:2602.02124](https://arxiv.org/abs/2602.02124) | [PDF](pdfs/2026_Graf_MahalanobisLiverOOD.pdf) | 公開データ（OSF）付き。試験単位分割 | ★★★ |
| 6 | PathologAI（ラット肝壊死、自然発生か処置関連か） | Bussola et al. 2023, Chem Res Toxicol (FDA NCTR) | [DOI](https://doi.org/10.1021/acs.chemrestox.3c00058) / [PMC10445282](https://pmc.ncbi.nlm.nih.gov/articles/PMC10445282/) | — | 処置関連性の区別を評価 | ★★☆ |
| 7 | MMO-Net: 多臓器同定 | Gámez Serna et al. 2022, J Pathol Inform 13:100126 (Roche) | [DOI](https://doi.org/10.1016/j.jpi.2022.100126) / [PMC9577048](https://pmc.ncbi.nlm.nih.gov/articles/PMC9577048/) | — | 唯一のleave-one-lab-out検証。公開データ | ★★☆ |
| 8 | Kuklyte et al.（ラット多臓器 病変セグメンテーション） | Kuklyte et al. 2021, Toxicol Pathol (Deciphex) | [DOI](https://doi.org/10.1177/0192623320986423) / [PMC8091423](https://pmc.ncbi.nlm.nih.gov/articles/PMC8091423/) | — | タイル単位分割によるリークの例 | ★★☆ |
| 9 | Bigpicture 非臨床AIの総説（D5.06） | Pohlmeyer-Esch et al. 2025 | [PMC12612283](https://pmc.ncbi.nlm.nih.gov/articles/PMC12612283/) | — | 分割・CI・fit-for-purpose検証の推奨 | ★★☆ |
| 10 | Zehnder et al.（ラット肝 所見分類） | Zehnder et al. 2025, Sci Rep (Genentech) | [DOI](https://doi.org/10.1038/s41598-025-86615-6) / [PMC11794696](https://pmc.ncbi.nlm.nih.gov/articles/PMC11794696) | — | 企業内データの所見別AUROC | ★☆☆ |
| 11 | VIPER（ラット毒性病理ROIのVLM-QAベンチマーク） | Weishaupt et al. 2026, arXiv | [arXiv:2608.26382](https://arxiv.org/abs/2608.26382) | — | 「ベンチマーク」を名乗る唯一の例（WSIではない） | ★★☆ |

PMC掲載の論文（No.1, 6–10）は、ボット対策のため自動ダウンロードできなかった。リンクのみ記載。

---

## 📝 補足：抄録のみ確認した文献

| 文献 | 内容 | 確認状況 |
|:---|:---|:---|
| Funk et al. 2025, Toxicol Pathol (Roche/Visium), [DOI](https://doi.org/10.1177/01926233251339653) | ラット肝58試験のMIL、腎への転移学習 | AUROCの数値は未確認 |
| Forest et al. 2026, Toxicol Pathol (Merck/AIRA), [DOI](https://doi.org/10.1177/01926233261475696) | one-class異常検知。感度 95.9%、特異度 70.8% | 分割方法は未確認 |
| Babburi et al. 2026, Vet Pathol (AbbVie), [DOI](https://doi.org/10.1177/03009858261462565) | BiGANによる異常検知・重症度。AUC 0.77 | 分割方法は未確認 |
| Li et al. 2025, MICCAI (GraphTox, Merck) | グラフ型異常検知。AUC 0.80 | 詳細は未確認 |
| Tokarz et al. 2021 / Steinbach et al. 2024, Toxicol Pathol | ラット心筋症のセグメンテーション、病理医間一致 | PMC8262119 / PMC11412787 で全文あり |
| Shimazaki et al. 2022, J Toxicol Pathol, [DOI](https://doi.org/10.1293/tox.2021-0053) | ラット肝7所見 | PMC9018404 で全文あり |
| Tomikawa et al. 2025, J Toxicol Pathol（製薬協AI病理TF） | 2017年以降の44報のレビュー | PMC12208865 |
