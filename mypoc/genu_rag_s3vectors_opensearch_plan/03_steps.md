# 実行手順書: GenUで S3 Vectors RAG / OpenSearch RAG を試す v4

## 1. 事前準備

### 1.1 用意するもの

- AWSアカウント
- Bedrockを利用するリージョン
- 利用するembedding modelの有効化
- S3バケット
- GenU環境
- 評価データ一式

### 1.2 推奨設定

| 項目 | 推奨 |
|---|---|
| `numberOfResults` | 10 |
| embedding model | 日本語対応・多言語対応モデル |
| chunking | まずは両KBで同一 |
| filter | なし → status → effective_from_num → category/doc_id |
| 評価系統 | S3 semantic / OpenSearch semantic / OpenSearch hybrid |

embedding modelは必ず評価表に記録する。

### 1.3 データを確認

このZIP内の以下を使う。

```text
data/genu_rag_eval_dataset/docs/
data/genu_rag_eval_dataset/eval_questions.csv
```

`docs/` には `.md` と `.metadata.json` が入っている。

## 2. S3へ評価データをアップロード

例:

```bash
aws s3 sync ./data/genu_rag_eval_dataset/docs/ s3://<your-bucket>/genu-rag-eval/docs/
```

確認:

```bash
aws s3 ls s3://<your-bucket>/genu-rag-eval/docs/
```

`.metadata.json` が一緒にアップロードされていることを確認する。

## 3. S3 Vectors用Knowledge Baseを作る

### 3.1 Bedrock Consoleで作成

1. Amazon Bedrock Consoleを開く
2. Knowledge Basesを開く
3. Create knowledge base
4. Data sourceにS3を選択
5. S3 URIに以下を指定

```text
s3://<your-bucket>/genu-rag-eval/docs/
```

6. Embedding modelを選択
7. Vector storeにS3 Vectorsを選択
8. 作成
9. Data source syncを実行

### 3.2 テスト

Bedrock Consoleで以下を質問。

```text
テレワークの通常時上限と、特別な事情がある場合の上限は？
```

まずフィルタなしで回答・引用元を見る。

## 4. OpenSearch用Knowledge Baseを作る

### 4.1 Bedrock Consoleで作成

1. Amazon Bedrock Consoleを開く
2. Knowledge Basesを開く
3. Create knowledge base
4. Data sourceにS3を選択
5. 同じS3 URIを指定

```text
s3://<your-bucket>/genu-rag-eval/docs/
```

6. Embedding modelをS3 Vectors側と同じにする
7. Vector storeにOpenSearch Serverlessを選択
8. OpenSearchのvector indexはBedrockに自動作成させる
9. 作成
10. Data source syncを実行

### 4.2 faiss確認

既存indexを使っていなければ、まずは深く気にしなくてよい。
既存indexを使う場合は、engineが `faiss` か確認する。

## 5. 3系統で検索する

以下の3系統で同じ質問を実行する。

| 系統 | KB | Search type |
|---|---|---|
| A | S3 Vectors KB | SEMANTIC |
| B | OpenSearch KB | SEMANTIC |
| C | OpenSearch KB | HYBRID |

OpenSearch側は必ず **SEMANTICとHYBRIDの両方** を実行する。
これをしないとOpenSearchの優位性を測れない。

`numberOfResults` は10に固定する。

## 6. メタデータフィルタを試す

### 6.1 フィルタなし

旧版文書やノイズ文書が混ざるかを見る。

対象質問:

```text
テレワークの通常時上限と、特別な事情がある場合の上限は？
テレワーク申請は何営業日前まで？
2026年度版で、テレワークの上限は週1日か週2日か？根拠も示して。
```

### 6.2 `status = active`

```json
{
  "equals": {
    "key": "status",
    "value": "active"
  }
}
```

旧版を除外できるかを見る。

### 6.3 `effective_from_num >= 20260401`

```json
{
  "greaterThanOrEquals": {
    "key": "effective_from_num",
    "value": 20260401
  }
}
```

日付比較は文字列ではなく数値フィールドを使う。

### 6.4 `category = AI基盤`

```json
{
  "equals": {
    "key": "category",
    "value": "AI基盤"
  }
}
```

対象質問:

```text
庁内AI・RAGサービスの標準利用時間と夜間の優先処理は？
RAG検索を含む回答の応答時間目標は？
```

### 6.5 `doc_id = DOC-AI-SLA-2026`

```json
{
  "equals": {
    "key": "doc_id",
    "value": "DOC-AI-SLA-2026"
  }
}
```

対象質問:

```text
doc_idがDOC-AI-SLA-2026の文書に書かれた障害通知ルールを答えて。
```

## 7. GenUで確認

### 7.1 KBを切り替える

GenUのRAG Chatで参照するKnowledge Baseを以下のように切り替える。

- S3 Vectors KB
- OpenSearch KB

同じ質問を流し、回答と引用元を見る。

### 7.2 注意点

GenU標準UIでメタデータフィルタやHYBRID searchを自由に指定できない場合がある。
その場合、厳密評価はBedrock Console/APIで行う。

GenUではまず以下を見る。

- ユーザー体験としての回答差
- 引用表示の見え方
- レイテンシ
- 回答の安定性

## 8. 評価記録

### 8.1 基本評価行

評価テンプレートには、20問 × 3系統の基本評価行がある。

### 8.2 フィルタ評価行

さらに、以下のフィルタ評価行を追加済み。

| 質問 | フィルタ |
|---|---|
| Q01/Q02/Q20 | `status = active` |
| Q01/Q02/Q20 | `effective_from_num >= 20260401` |
| Q14/Q15 | `category = AI基盤` |
| Q19 | `doc_id = DOC-AI-SLA-2026` |

全質問×全フィルタは実施しない。
記録漏れを防ぐため、まずはテンプレートにある行だけ実行する。

### 8.3 記録項目

`evaluation_result_template.csv` を使う。

記録する主な項目:

| 項目 | 意味 |
|---|---|
| `system` | s3_vectors_semantic / opensearch_semantic / opensearch_hybrid |
| `embedding_model` | 使用したembedding model |
| `number_of_results` | 10固定推奨 |
| `filter_name` | none / status_active / effective_from_num / doc_id など |
| `latency_ms` | 検索または回答の所要時間 |
| `hit_rank` | 正解文書・チャンクが何位に出たか |
| `citation_correct` | 引用が正しいか |
| `answer_score` | 0/1/2 |

採点:

| 点 | 意味 |
|---:|---|
| 2 | 正しい |
| 1 | 一部正しい |
| 0 | 誤り |

## 9. 最終判断

### S3 Vectorsでよいケース

- 20問中、多くが正答
- 旧版/新版をフィルタで十分制御できる
- 表・金額・日付も許容範囲
- コスト重視
- 高度なハイブリッド検索が不要

### OpenSearchを選ぶべきケース

- S3 Vectorsが文書ID・金額・日付で弱い
- 旧版/新版の誤引用が目立つ
- OpenSearch HYBRIDで明確に改善する
- メタデータフィルタを多用する
- ハイブリッド検索が必要
- 組織・権限・文書分類で厳密に絞る必要がある

## 10. 後始末チェックリスト

POC終了後、不要リソースを削除する。

- [ ] 不要なKnowledge Baseを削除
- [ ] 不要なOpenSearch Serverless collectionを削除
- [ ] 不要なS3 Vector bucket / indexを削除
- [ ] S3評価データを削除
- [ ] CloudWatch Logsを確認
- [ ] 一時IAM Role / Policyを確認
- [ ] Cost Explorerで費用を確認

## 11. CSVの注意

CSVはExcelで開きやすいようにBOM付きUTF-8、CRLFで作成している。
スクリプトで読む場合は `utf-8-sig` を使う。
