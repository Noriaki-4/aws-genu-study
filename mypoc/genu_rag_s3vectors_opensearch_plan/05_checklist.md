# POCチェックリスト

## 作成前

- [ ] 利用リージョンを決める
- [ ] embedding modelを決める
- [ ] embedding model名を評価表に記録する
- [ ] 評価文書を置く通常のS3バケットを用意する
- [ ] S3 Vectors KBはBedrock ConsoleのQuick createで作成する前提を確認する
- [ ] GenU環境を確認する

## データ投入

- [ ] `docs/*.md` をS3へアップロード
- [ ] `docs/*.metadata.json` をS3へアップロード
- [ ] アップロード件数を確認
- [ ] `.md` と `.metadata.json` の件数が一致していることを確認
- [ ] metadata JSONがパースできることを確認
- [ ] metadata keysと最大サイズがS3 Vectors + Bedrock KBの制限内であることを確認
- [ ] target文書、old_version文書、noise文書が入っていることを確認
- [ ] ノイズ文書に共通キーワード備考が残っていないことを確認

## KB作成

- [ ] S3 Vectors KBをQuick createで作成
- [ ] Quick createで作成されたS3 vector bucket / index名を記録
- [ ] S3 Vectors KB IDを記録
- [ ] OpenSearch KBを作成
- [ ] OpenSearchのcollection / indexはBedrockに自動作成させる
- [ ] OpenSearch KB IDを記録
- [ ] 両KBで同じembedding modelを使用
- [ ] 両KBで同じchunking条件を使用
- [ ] 両KBを同期

## 評価

- [ ] S3 Vectors + SEMANTICを実行
- [ ] S3 VectorsではHYBRIDを実行しない
- [ ] OpenSearch + SEMANTICを実行
- [ ] OpenSearch + HYBRIDを実行
- [ ] numberOfResults = 10で統一
- [ ] search_typeを評価表に記録
- [ ] kb_id_or_nameを評価表に記録
- [ ] latency_msを記録
- [ ] hit_rankを記録
- [ ] citation_correctを記録
- [ ] answer_scoreを記録
- [ ] Q19はdoc_idフィルタありで実行
- [ ] Q15はAI基盤カテゴリ内の応答時間キーワード衝突テストとして解釈する
- [ ] 厳密評価はBedrock Console/APIの結果を正とする

## GenU確認

- [ ] S3 Vectors KB用とOpenSearch KB用のGenU環境を分ける、または`ragKnowledgeBaseId`を差し替えて再デプロイする
- [ ] GenUでは回答UX、引用表示、体感レイテンシを確認する
- [ ] GenU UIでフィルタを使う場合は`packages/common/src/custom/rag-knowledge-base.ts`をPOC metadata用に変更する
- [ ] GenU標準UIの結果を、検索タイプやnumberOfResults固定の厳密採点に使わない

## 後始末

- [ ] 不要なKnowledge Baseを削除
- [ ] Knowledge Base削除時のデータ削除ポリシーがDeleteかRetainか確認
- [ ] Quick createで作成されたS3 vector bucket / indexを削除
- [ ] 不要なOpenSearch Serverless collectionを削除
- [ ] S3評価データを削除
- [ ] 一時IAM Role / Policyを確認
- [ ] CloudWatch Logsを確認
- [ ] Cost Explorerで費用を確認
