# POCチェックリスト

## 作成前

- [ ] 利用リージョンを決める
- [ ] embedding modelを決める
- [ ] embedding model名を評価表に記録する
- [ ] S3バケットを用意する
- [ ] GenU環境を確認する

## データ投入

- [ ] `docs/*.md` をS3へアップロード
- [ ] `docs/*.metadata.json` をS3へアップロード
- [ ] アップロード件数を確認
- [ ] target文書とnoise文書が両方入っていることを確認
- [ ] ノイズ文書に共通キーワード備考が残っていないことを確認

## KB作成

- [ ] S3 Vectors KBを作成
- [ ] OpenSearch KBを作成
- [ ] OpenSearchのindexはfaiss前提で作成
- [ ] 両KBで同じembedding modelを使用
- [ ] 両KBで同じchunking条件を使用
- [ ] 両KBを同期

## 評価

- [ ] S3 Vectors + SEMANTICを実行
- [ ] OpenSearch + SEMANTICを実行
- [ ] OpenSearch + HYBRIDを実行
- [ ] numberOfResults = 10で統一
- [ ] latency_msを記録
- [ ] hit_rankを記録
- [ ] citation_correctを記録
- [ ] answer_scoreを記録
- [ ] Q19はdoc_idフィルタありで実行
- [ ] Q15はAI基盤カテゴリ内の応答時間キーワード衝突テストとして解釈する

## 後始末

- [ ] 不要なKnowledge Baseを削除
- [ ] 不要なOpenSearch Serverless collectionを削除
- [ ] 不要なS3 Vector bucket / indexを削除
- [ ] S3評価データを削除
- [ ] Cost Explorerで費用を確認
