# GenU RAG 比較検証パッケージ v4

このパッケージは、AWS GenUで以下を比較検証するための資料とデータです。

- Bedrock Knowledge Bases + S3 Vectors + SEMANTIC
- Bedrock Knowledge Bases + OpenSearch Serverless + SEMANTIC
- Bedrock Knowledge Bases + OpenSearch Serverless + HYBRID

今回はTextractを使わず、テキスト化済みデータでRAG検索精度の違いを確認します。

## v4での主な改善と引き継ぎ

1. v3で削除したノイズ文書の共通キーワード備考を削除済み状態として維持。
2. v3で全ノイズ文書に残っていた `応答時間` をAI基盤系ノイズのみに限定。
3. ノイズ文書の役割を「検索指標用」、旧版文書の役割を「回答誤り誘発用」と明記。
4. OpenSearch HYBRID searchを評価系統に追加。
5. v2で本文一致から切り離したQ19を、メタデータdoc_idフィルタ検証として評価表に明記。
6. 評価テンプレートに latency / numberOfResults / embedding model / hit_rank を追加。
7. `effective_from_num` を追加し、日付比較の型ノイズを低減。
8. フィルタ検証用の追加行を評価テンプレートに追加。
9. POC後のOpenSearch Serverless削除チェックを追加。
10. 旧版文書の `dataset_role` を `old_version` として、正規ターゲット文書と区別。

## ファイル構成

```text
01_plan.md                         # 検証計画
02_understanding_points.md          # 理解のポイント
03_steps.md                         # 実行手順
04_references.md                    # 参考リンク
05_checklist.md                     # POCチェックリスト
data/
  evaluation_result_template.csv    # 評価記録テンプレート
  genu_rag_eval_dataset/
    README.md
    eval_questions.csv
    docs/
      target docs
      old version docs
      noise docs
      *.metadata.json
```

## 最初に読む順番

1. `01_plan.md`
2. `02_understanding_points.md`
3. `03_steps.md`
4. `05_checklist.md`
5. `04_references.md`
6. `data/genu_rag_eval_dataset/eval_questions.csv`
7. `data/evaluation_result_template.csv`

## 重要な解釈

このPOCで差が出ない場合でも、本番でS3 VectorsとOpenSearchが同等とは判断しない。
差が出なければ、フェーズ2として本番想定量に近いデータで検証する。
