# 論文インデックス

本調査（PROP-06: MELLODDY型・製薬企業間連合学習コンソーシアムの毒性病理領域への応用可能性）で厳選した11件の書誌情報・要約。

---

### 1. MELLODDY: Cross-pharma Federated Learning at Unprecedented Scale Unlocks Benefits in QSAR without Compromising Proprietary Information
- **著者**: Heyndrickx, W., Mervin, L., Morawietz, T., et al.（MELLODDYコンソーシアム）
- **出典**: *Journal of Chemical Information and Modeling*, 63(7), 1832–1839 (2023)
- **リンク**: https://pubs.acs.org/doi/10.1021/acs.jcim.3c00799
- **概要**: MELLODDYプロジェクトの中核成果論文。10製薬企業が保有する2,700万化合物・4千万活性値を、生データを一切共有せず連合学習でQSARモデルを共同構築し、単独企業モデルを上回る予測性能を達成したことを報告。ただし対象データは全て**分子構造（SMILES）と活性値のテーブルデータ**であり、画像データは一切扱っていない点が本調査の核心的論点。

### 2. Industry-Scale Orchestrated Federated Learning for Drug Discovery
- **著者**: MELLODDYコンソーシアム（Owkin主導）
- **出典**: arXiv:2210.08871 (2022) / AAAI-23 IAAI Track
- **リンク**: https://arxiv.org/abs/2210.08871
- **概要**: MELLODDYのシステムアーキテクチャ論文。10社が並行して勾配を計算し、中央サーバがSecure Aggregation（後述#11参照）で集約する非同期オーケストレーション方式、Substra基盤上のブロックチェーン型監査ログ、参加権限管理の設計を解説。「暗号化された勾配のみを共有」という抽象的な説明にとどまり、具体的な暗号方式の詳細はブログ・技術記事側（#11）でしか確認できなかった。

### 3. A review on federated learning in computational pathology
- **著者**: Schoenpflug, L. A., Nie, Y., Sheikhzadeh, F., Koelzer, V. H.
- **出典**: *Computational and Structural Biotechnology Journal* (2024) / DOI:10.1016/j.csbj.2024.10.037
- **リンク**: https://pmc.ncbi.nlm.nih.gov/articles/PMC11584763/
- **概要**: 2021年8月〜2024年7月に発表された連合学習×計算病理学の43件を網羅したシステマティックレビュー。染色・スキャナ差異による深刻なデータ異質性を主課題と特定し、GANベース染色正規化併用で10%以上の性能改善を報告。差分プライバシー/セキュアマルチパーティ計算（SMC）併用時の性能低下は0.2〜6%と定量化。**前臨床・毒性病理への言及は皆無**（臨床がん診断・治療反応予測・Gleason分類のみを対象）。

### 4. Navigating real-world challenges: A case study on federated learning in computational pathology
- **著者**: Schoenpflug, L. A., et al.（Univ. Zurich, Roche Diagnostics, Univ. Basel, ETH Zurich）
- **出典**: *Journal of Pathology Informatics* (2025) / DOI:10.1016/j.jpi.2025.100464
- **リンク**: https://pmc.ncbi.nlm.nih.gov/articles/PMC12357140/
- **概要**: 転移性黒色腫のIHC画像を用いた実運用連合学習の実装知見。NVIDIA FLAREでの構築時に(1)統合モデルが必ずしも全施設で局所モデルを上回らない、(2)ギガピクセル画像＋異種ハードウェアで実験サイクルが長期化、(3)病院・企業ネットワーク制限がインフラ設計を阻害、(4)フレームワーク設定の複雑さ、という4つの実運用障壁を報告。**Roche Diagnosticsが臨床病理領域で連合学習WSI基盤を既に検証している**ことを示す一次証拠だが、対象は臨床（ヒト腫瘍）でありPROP-06が対象とする前臨床毒性病理ではない。

### 5. HistoFS: Non-IID Histopathologic Whole Slide Image Classification via Federated Style Transfer with RoI-Preserving
- **著者**: Raswa, F. H., Lu, C.-S., Wang, J.-C.
- **出典**: CVPR 2025
- **リンク**: https://openaccess.thecvf.com/content/CVPR2025/html/Raswa_HistoFS_Non-IID_Histopathologic_Whole_Slide_Image_Classification_via_Federated_Style_CVPR_2025_paper.html
- **概要**: WSI内に複数の形態構造・染色スタイルが混在する状況で、施設間の染色・スキャナ差異（非IID特徴シフト）をクライアント間スタイル転送で吸収しつつ、病変領域（RoI）をAuthenticity Moduleで保護する手法。RQ2（スキャナ・染色プロトコル差異の吸収）に対する2025年時点で最も具体的な技術的回答。3つの臨床データセットでSOTA達成を報告。

### 6. Nationwide federated learning for histopathology: secure deployment across Germany behind firewalls（TheOden）
- **著者**: 未確認（PubMed抄録のみ取得、詳細な著者リストは本文有料部分）
- **出典**: PubMed:42509385 (2026年)
- **リンク**: https://pubmed.ncbi.nlm.nih.gov/42509385/
- **概要**: エッセン・ケルン・ライプツィヒの3大学病院とダルムシュタットのサーバ間で、クライアント側ポート開放不要のリバースプロキシ方式により病院ファイアウォール越しの連合学習を実現したTheOdenフレームワーク。大腸癌・乳癌のセグメンテーションでDiceスコア0.75前後を達成。連合学習の「ネットワークインフラ」側の実運用課題（本調査4.のSchoenpflug論文と同じ課題領域）に対する具体解。製薬企業・CRO間ネットワークという毒性病理特有の閉域網環境への応用可能性を検討する際の技術的参照点。

### 7. Federated Learning in Computational Toxicology: An Industrial Perspective on the Effiris Hackathon
- **出典**: *Chemical Research in Toxicology*, 36(9), 1381–1390 (2023) / DOI:10.1021/acs.chemrestox.3c00137
- **リンク**: https://pmc.ncbi.nlm.nih.gov/articles/PMC10523574/
- **概要**: Lhasa Limitedが開発した連合学習プラットフォーム「Effiris」を用い、Roche含む7製薬企業が参加して実施されたハッカソンの報告。SMILES構造式からのオンターゲット活性予測という**計算毒性学（QSAR）**領域での連合学習の産業実証例。**画像（WSI）ではなく分子記述子ベース**である点がMELLODDY同様の限界。「製薬企業間の連合学習は毒性領域で既に組織的合意形成の実績がある」ことを示す最重要の間接的precedent。

### 8. Federated Learning: An approach with Hybrid Homomorphic Encryption
- **著者**: Correia, P., Silva, I., Amorim, I., Maia, E., Praça, I.
- **出典**: arXiv:2509.03427 (2025)
- **リンク**: https://arxiv.org/abs/2509.03427
- **概要**: 対称暗号PASTAとBFV完全準同型暗号を組み合わせたハイブリッド方式。クライアント上りバンド幅を2,000倍超削減、クライアント実行時間をBFV単独方式比30%削減し、精度低下は平文比1.3%（97.6%）に抑制。ただし**サーバ側計算コストは約15,621倍に増加**しており、WSIパッチのような大規模特徴量への適用は依然として重い計算コストという代償を伴う。RQ1（準同型暗号のWSI連合学習への転用コスト）に対する最も具体的な定量データ。

### 9. The eTOX Data-Sharing Project to Advance in Silico Drug-Induced Toxicity Prediction
- **著者**: Cases, M., Briggs, K., Steger-Hartmann, T., et al.
- **出典**: *International Journal of Molecular Sciences*, 15(11), 21136–21154 (2014) / DOI:10.3390/ijms151121136
- **リンク**: https://pmc.ncbi.nlm.nih.gov/articles/PMC4264217/
- **概要**: IMI（Innovative Medicines Initiative）による2010年開始の毒性データ共有コンソーシアム。13製薬企業が過去の毒性試験データ（4,000試験超）を「オネストブローカー」（Lhasa Limited）経由で段階的機密度別に共有する**中央集約型**の枠組み。MELLODDY（非集約・連合学習型）とは対照的なガバナンスモデルであり、「毒性領域での企業間データ共有は組織的には10年以上前から実績がある」ことを示す一方、対象は構造化データ（試験メタデータ・化学構造）のみで画像データは扱っていない。

### 10. Benchmarking Federated Learning Frameworks for Medical Imaging Deployment: A Comparative Study of NVIDIA FLARE, Flower, and Owkin Substra
- **著者**: Gupta, R., Chowdhury, A., Nalawade, S.
- **出典**: arXiv:2511.00037 (2025)
- **リンク**: https://arxiv.org/abs/2511.00037
- **概要**: 医用画像向け主要連合学習フレームワーク3種の比較研究。NVIDIA FLAREは本番運用のスケーラビリティ、Flowerは研究・プロトタイピングの柔軟性、Owkin Substra（MELLODDYの技術的後継オープンソース基盤）はプライバシー・コンプライアンス機能に強みがあると結論。毒性病理連合学習基盤を選定する際の技術選定の出発点。

### 11. The MELLODDY Project from a Privacy Point of View（技術ブログ、参考情報源）
- **出典**: CrySyS Lab Blog (2021)
- **リンク**: https://blog.crysys.hu/2021/05/melloddy/
- **概要**: MELLODDYが採用した秘匿化手法の技術解説。**準同型暗号ではなくペアワイズマスキングに基づくSecure Aggregation（SMPCベース）が主機構**であり、個々の勾配ではなく集約後の勾配のみがサーバから見える設計であることを説明。査読論文ではない技術ブログだが、Owkin公式発表と整合しており、本調査で最も重要な誤り訂正（後述）の裏付けとして採用。

---

## 論文選定から見えた全体傾向・重要な訂正

- **既存レポート（topics/01, フロンティア6）の記述への訂正**: topics/01のレポートは「MELLODDYが準同型暗号・差分プライバシー・Secure Aggregationを採用」と記述しているが、一次情報源（#2, #11）を確認した結果、**MELLODDYの主機構はSecure Aggregation（SMPCベースのペアワイズマスキング）であり、準同型暗号は採用されていない**。差分プライバシーもMulti-task learning構造自体が持つ暗黙の防御として言及される程度で、標準的なDP-SGDのような明示的機構ではない。準同型暗号はむしろ学術研究側（#8）で「MELLODDY型のFLに追加適用しうる補完技術」として提案されている段階であり、両者を混同しないことが重要。
- **「毒性病理×連合学習」の直接実装例は2026年8月時点で一次文献上に存在しない**。連合学習×WSIの実例（#3, #4, #5, #6）はすべて臨床（ヒト腫瘍・IHC）領域に限定され、連合学習×毒性学の実例（#7, #9）はすべて画像を扱わない構造化データ（QSAR・試験メタデータ）に限定される。両者の交差点（毒性病理WSI×連合学習）は真に未着手。
- **製薬企業間のデータ共有は組織的には既に2つの異なるガバナンスモデルで実績がある**：eTOX型（#9, 中央集約＋オネストブローカー, 2010年〜）とMELLODDY型（#1/#2, 非集約＋連合学習, 2019年〜）。毒性病理WSIへの応用を検討する際、どちらのモデルを踏襲すべきかは技術選定以前の組織設計上の論点になる。
