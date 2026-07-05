# GenU RAG 評価用データセット v4

## 目的

AWS GenU + Bedrock Knowledge Bases で、S3 Vectors RAG と OpenSearch RAG の検索精度差を確認するためのデータです。

## v3での改善点

- ターゲット文書12件に加え、ノイズ文書72件を追加。
- v2で全ノイズ文書に入っていた共通キーワード備考を削除し、HYBRID検索への人工的な罰を避けた。
- 旧版/新版、近似語、似た数値、似たカテゴリを混ぜ、検索差が出やすくした。
- `effective_from` に加えて、数値比較用の `effective_from_num` を追加。
- Q19の対象文書本文から `doc_id` 文字列を外し、メタデータ検索テストとして意味が出るようにした。
- 評価は `S3 Vectors semantic`、`OpenSearch semantic`、`OpenSearch hybrid` の3系統で記録する想定。

## 注意

このデータセットでも本番規模とは異なります。差が出ない場合でも、本番想定データ量でフェーズ2検証が必要です。

## 推奨検索設定

- `numberOfResults`: 10
- embedding model: 日本語対応・多言語対応モデルを使用し、評価表にモデル名を記録
- filterなし → `status = active` → `effective_from_num >= 20260401` の順に比較

## ファイル構成

```text
docs/
  01_...md
  01_...md.metadata.json
  ...
  noise_*.md
  noise_*.md.metadata.json
eval_questions.csv
```


## ノイズ文書の役割

ノイズ文書は、主に検索指標用です。
hit_rank、retrieved_source_doc、citation_correctを見るために使います。

回答誤りを誘発する主な文書は、旧版の `11_near_duplicate_old_policy.md` です。
ノイズ文書はテンプレート量産のため、現実の文書群より見分けやすい限界があります。

## status と effective_from_num

一部のノイズ文書には、`effective_from_num = 20260401` かつ `status = old` のものがあります。
これは年度内の下書き、研修資料、廃止資料、参考資料を想定した意図的なデータです。

日付フィルタだけでは有効文書に絞れないため、`status = active` との組み合わせを確認してください。


## v4での応答時間ノイズ修正

v3では、全ノイズ文書に `応答時間` が残っていました。
v4では、Q15の意図的な検索衝突をAI基盤カテゴリ内に限定するため、AI基盤以外のノイズ文書ではカテゴリに応じた別表現へ置換しています。
