# AWS GenUで S3 Vectors RAG / OpenSearch RAG を比較検証する計画 v4

## 0. 今回の前提

- 今回は **Textractなし**。
- 原本はテキスト化済みの Markdown / テキストファイルを使う。
- 比較対象は以下の3系統。
  1. Bedrock Knowledge Bases + S3 Vectors + semantic search
  2. Bedrock Knowledge Bases + OpenSearch Serverless + semantic search
  3. Bedrock Knowledge Bases + OpenSearch Serverless + hybrid search
- GenUは大きく改造せず、まずはKnowledge Baseを切り替えてRAGの違いを確認する。
- ただし、GenU標準UIで検索タイプやメタデータフィルタを自由に指定できない場合があるため、厳密な比較はBedrock Console/APIで行う。

## 1. 最重要な注意: 小規模検証の限界とノイズ設計

### 1.1 小規模検証の限界

今回の評価データはv4でノイズ文書を追加・調整したが、それでも本番想定の文書量とは大きく異なる。

そのため、以下を明記する。

```text
このPOCで差が出ない場合でも、S3 VectorsとOpenSearchが本番で同等とは判断しない。
差が出ない場合は、フェーズ2として本番想定量に近いデータ、または数千〜数万チャンク規模のノイズデータで再検証する。
```

小規模データでは、semantic searchだけでも正解チャンクが上位に入りやすい。
このため、S3 VectorsとOpenSearchの差が見えにくい。

### 1.2 ノイズ文書の役割

v4では、ノイズ文書から `RAG`、`Knowledge Base`、`OpenSearch`、`S3 Vectors`、`ハイブリッド検索`、`doc_id` などを一律に含める備考文を削除した。
これにより、Q18やQ19でOpenSearch HYBRIDだけが不自然に不利になるキーワード汚染を避ける。

ノイズ文書の主な役割は **検索指標用** である。
つまり、hit_rank、retrieved_source_doc、citation_correctを見るための検索汚染を作る。

一方、回答誤り誘発の主役は `11_near_duplicate_old_policy.md` のような旧版・近似文書である。
ノイズ文書はテンプレート量産であり現実文書より見分けやすいため、回答誤り誘発力は限定的と解釈する。

### 1.3 status と effective_from_num の意図

一部のノイズ文書には `effective_from_num = 20260401` かつ `status = old` のものがある。
これは年度内に作られた下書き、研修資料、廃止資料、参考資料を想定した意図的なデータである。

このため、`effective_from_num >= 20260401` だけでは有効文書に絞れない。
本番では `status = active` と日付条件を組み合わせる必要がある。

## 2. 目的

今回の目的は、AWS GenUでRAGを提供する前提で、S3 VectorsとOpenSearch Serverlessのどちらが業務要件に合うかを短時間で見極めること。

特に確認したい観点は以下。

| 観点 | 確認内容 |
|---|---|
| 回答精度 | 質問に対して正しい答えが返るか |
| 根拠精度 | 正しい文書・チャンクを根拠にしているか |
| 旧版/新版の切り分け | 古い規程に引っ張られないか |
| 数値・日付・金額 | 金額、日付、期限、件数を正確に拾えるか |
| 文書ID・固有語 | doc_idや制度名などを正確に拾えるか |
| メタデータフィルタ | status/category/org/doc_id等で絞り込めるか |
| hit@k | 正解チャンクが検索結果の何位に出るか |
| レイテンシ | 検索・回答の遅延 |
| コスト/運用性 | 低コスト重視か、本番検索性能重視か |

## 3. 全体構成

```text
評価用S3バケット
  └─ docs/
      ├─ target docs
      ├─ old version docs
      └─ noise docs
          ｜
          ├─ Bedrock Knowledge Base A
          │    └─ Vector store: S3 Vectors
          │        └─ search type: SEMANTIC
          │
          └─ Bedrock Knowledge Base B
               └─ Vector store: OpenSearch Serverless
                   ├─ search type: SEMANTIC
                   └─ search type: HYBRID

GenU RAG Chat
  ├─ KB-Aを指定して質問
  └─ KB-Bを指定して同じ質問
```

## 4. 比較対象の位置づけ

| 項目 | S3 Vectors | OpenSearch Serverless |
|---|---|---|
| 基本思想 | 低コスト・大量ベクトル保管 | 本番向け検索基盤 |
| 検索方式 | semantic search中心 | semantic / hybridを試しやすい |
| ハイブリッド検索 | 非対応 | 検証対象に必ず含める |
| 固有名詞・番号・金額 | やや弱点が出やすい | 強くしやすい |
| メタデータフィルタ | 使えるが制約に注意 | 使いやすい |
| チューニング余地 | 小さめ | 大きい |
| コスト | 低く抑えやすい | 常時費用が重くなりやすい |
| 今回の見るべき点 | 低コスト構成でどこまで耐えられるか | semantic/hybridでどこまで精度差が出るか |

## 5. 推奨検証ステップ

### Step 1: 評価データをS3にアップロード

`data/genu_rag_eval_dataset/docs/` 配下をS3へアップロードする。

例:

```text
s3://<your-bucket>/genu-rag-eval/docs/
```

アップロード対象は `.md` と `.metadata.json` の両方。

### Step 2: S3 Vectors用Knowledge Baseを作成

| 項目 | 設定 |
|---|---|
| Data source | S3 |
| S3 path | `s3://<your-bucket>/genu-rag-eval/docs/` |
| Vector store | S3 Vectors |
| Search type | SEMANTIC |
| Embedding model | 日本語対応・多言語対応モデル |
| Chunking | まずはデフォルトまたは固定サイズ |
| Metadata | `.metadata.json` を利用 |

### Step 3: OpenSearch用Knowledge Baseを作成

| 項目 | 設定 |
|---|---|
| Data source | S3 |
| S3 path | `s3://<your-bucket>/genu-rag-eval/docs/` |
| Vector store | OpenSearch Serverless |
| Index | Bedrockに自動作成させる |
| Engine | faiss |
| Embedding model | S3 Vectors側と同じ |
| Chunking | S3 Vectors側と同じ |
| Search type | SEMANTIC と HYBRID の両方を評価 |

既存OpenSearch indexを使わず、Bedrockに自動作成させるのが最短。

### Step 4: 両方のKnowledge Baseを同期

Bedrock ConsoleまたはAPIでData source syncを実行する。

### Step 5: Bedrock Console/APIで3系統を比較

同じ質問を以下3系統で実行する。

| 系統 | Vector store | Search type |
|---|---|---|
| A | S3 Vectors | SEMANTIC |
| B | OpenSearch Serverless | SEMANTIC |
| C | OpenSearch Serverless | HYBRID |

`numberOfResults` は **10固定** を推奨する。
結果記録にも必ず残す。

### Step 6: フィルタなし / フィルタありを比較

1. フィルタなし
2. `status = active`
3. `effective_from_num >= 20260401`
4. 必要に応じて `category` や `doc_id`

### Step 7: GenU RAG Chatで確認

Bedrock Console/APIで差分が見えたら、GenU側でKnowledge Baseを切り替えて同じ質問を流す。

GenUでは主に以下を見る。

- ユーザー体験としての回答差
- 引用表示
- 体感レイテンシ
- 回答安定性

## 6. メタデータ設計

今回の評価データには以下を付ける。

| メタデータ | 用途 | 例 |
|---|---|---|
| `doc_id` | 文書ID指定 | `DOC-TELEWORK-2026` |
| `org` | 組織・部門 | `共通` |
| `category` | 文書分類 | `勤務制度` |
| `effective_from` | 適用開始日文字列 | `2026-04-01` |
| `effective_from_num` | 適用開始日数値 | `20260401` |
| `status` | 有効/旧版 | `active`, `old` |
| `dataset_role` | target/noise | `target`, `noise` |
| `topic_tags` | 関連語 | `["RAG", "SLA"]` |

`effective_from` はISO形式文字列でも辞書順が日付順と一致しやすいが、POCで型起因のノイズを避けるため、数値の `effective_from_num` を併記する。

## 7. 試すべきフィルタ

### `status = active`

```json
{
  "equals": {
    "key": "status",
    "value": "active"
  }
}
```

### `effective_from_num >= 20260401`

```json
{
  "greaterThanOrEquals": {
    "key": "effective_from_num",
    "value": 20260401
  }
}
```

### `category = AI基盤`

```json
{
  "equals": {
    "key": "category",
    "value": "AI基盤"
  }
}
```

### `doc_id = DOC-AI-SLA-2026`

```json
{
  "equals": {
    "key": "doc_id",
    "value": "DOC-AI-SLA-2026"
  }
}
```

Q19はこのフィルタを付けて評価する。

### `andAll`

```json
{
  "andAll": [
    {
      "equals": {
        "key": "status",
        "value": "active"
      }
    },
    {
      "greaterThanOrEquals": {
        "key": "effective_from_num",
        "value": 20260401
      }
    }
  ]
}
```

## 8. 評価方法

### 8.1 基本評価

全20問を以下3系統で評価する。

- S3 Vectors + SEMANTIC
- OpenSearch + SEMANTIC
- OpenSearch + HYBRID

### 8.2 フィルタ検証

全問×全フィルタは組み合わせが多すぎるため、v3では以下に絞って評価テンプレートに追加行を用意する。

| 質問 | フィルタ |
|---|---|
| Q01/Q02/Q20 | `status = active` |
| Q01/Q02/Q20 | `effective_from_num >= 20260401` |
| Q14/Q15 | `category = AI基盤` |
| Q19 | `doc_id = DOC-AI-SLA-2026` |

### 8.3 記録項目

`data/evaluation_result_template.csv` を使い、以下を記録する。

- embedding_model
- number_of_results
- search_type
- filter_name
- latency_ms
- hit_rank
- citation_correct
- answer_score

検索と生成を分離するため、hit@kを必ず見る。
正解文書が検索結果10件の何位に出たかを `hit_rank` に記録する。

採点基準:

| 点 | 意味 |
|---:|---|
| 2 | 回答・根拠とも正しい |
| 1 | 回答は概ね正しいが、根拠や詳細に不安あり |
| 0 | 誤答、または根拠文書が違う |

## 9. 差が出やすい質問

| 質問ID | 狙い |
|---|---|
| Q01 | 旧版/新版の取り違え |
| Q02 | 申請期限の取り違え |
| Q05 | 通常期限と例外期限の識別 |
| Q07 | 金額範囲の識別 |
| Q16 | Markdown表の列対応 |
| Q17 | Markdown表の複数列対応 |
| Q18 | 概念・同義語 |
| Q19 | メタデータdoc_id指定 |
| Q20 | 旧版/新版の明示的比較 |

## 10. 判断基準

| 結果 | 判断 |
|---|---|
| S3 Vectorsで大半が正答 | 低コスト構成の候補にできる |
| S3 Vectorsが数値・IDで弱い | OpenSearch優先 |
| OpenSearch HYBRIDだけ改善 | ハイブリッド検索が必要 |
| 両方とも旧版に引っ張られる | メタデータ設計またはフィルタ適用が不足 |
| 表質問で両方弱い | 前処理またはTextract/Advanced Parsingを別途検討 |
| 差が出ない | 本番想定量でフェーズ2検証。小規模同等性を本番同等性と解釈しない |

## 11. POC後の後始末

OpenSearch Serverlessは最低OCU課金が発生する可能性があるため、POC終了後に以下を確認する。

- 不要なKnowledge Baseを削除
- 不要なOpenSearch Serverless collectionを削除
- 不要なS3 Vector bucket / indexを削除
- S3評価データを削除
- CloudWatch Logsや一時IAM Roleを確認
- コストエクスプローラーで当日〜翌日の費用を確認

## 12. 最初の結論仮説

- S3 Vectorsは、低コストで大量データを持つ候補として有望。
- ただし、自治体・庁内規程のように、文書ID、年度、金額、日付、旧版/新版が重要な場合はOpenSearchのほうが安定しやすい可能性がある。
- 本番を見据えるなら、OpenSearch Serverless + メタデータフィルタ + HYBRID検索を本命として検証する。
- 低コスト要件が強い場合は、S3 Vectorsでどこまで耐えられるかを今回のデータで確認する。


## 13. v4での微修正

v3では、`応答時間` が全72件のノイズ文書に残っていた。
Q15の意図的なキーワード衝突テストとしては成立するが、勤務制度・補助金・防災などの参考資料まで応答時間に言及するのは不自然だった。

v4では、`応答時間` をAI基盤カテゴリのノイズ文書にのみ残し、その他カテゴリでは `処理期間`、`審査日数`、`公開準備期間` などカテゴリに合う表現へ置換した。
これにより、Q15は「AI基盤カテゴリ内の紛らわしいSLA系文書から正規SLA文書を拾えるか」という、より自然なテストになった。
