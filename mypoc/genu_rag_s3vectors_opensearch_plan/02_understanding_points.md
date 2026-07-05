# Bedrock KB / GenU RAG 比較検証の理解ポイント v4

## 1. 今回の比較で絶対に外してはいけないこと

OpenSearchの強みを測るには、OpenSearch側で **SEMANTIC** と **HYBRID** の両方を試す必要がある。

S3 VectorsとOpenSearchを両方semantic-onlyで比較すると、OpenSearchの本来の優位性である以下を測れない。

- キーワード一致
- doc_idや固有名詞
- 金額・日付・番号
- ハイブリッド検索
- メタデータフィルタと検索タイプの組み合わせ

そのため、今回の評価は3系統にする。

```text
A. S3 Vectors + SEMANTIC
B. OpenSearch + SEMANTIC
C. OpenSearch + HYBRID
```

## 2. 小規模データの限界とノイズ設計

データが少ないと、semantic searchだけでも正解チャンクが上位に出やすい。
その結果、S3 VectorsとOpenSearchで差が出ないことがある。

この場合の解釈は以下。

```text
差が出なかった = 本番でも同等
ではない。

差が出なかった = この小規模データでは差が観測できなかった
である。
```

本番判断には、フェーズ2として本番想定に近いデータ量・チャンク数で再検証する。

v3では、全ノイズ文書に共通キーワードを大量投入する設計をやめた。
これは、HYBRID検索だけが不自然に不利になる人工的なバイアスを避けるためである。

ノイズ文書の役割は検索結果の混入、hit_rank、citation_correctの確認である。
回答レベルの誤答誘発は、主に旧版文書や近似文書で確認する。

## 3. Bedrock Knowledge Basesとは

Bedrock Knowledge Basesは、S3などのデータソースを取り込み、チャンク化、埋め込み生成、ベクトルストアへの保存、検索、回答生成までを扱うマネージド機能。

GenUから使う場合は、GenUのRAG Chatの情報源としてKnowledge Baseを指定する形になる。

## 4. GenUで見るべきこと

GenUはRAG ChatのUIとして使う。
ただし、最初からGenUだけで検証すると、以下が混ざる。

- Knowledge Baseの検索性能
- GenU側の検索呼び出し設定
- LLMの回答生成
- UI側の引用表示
- メタデータフィルタの渡し方
- search typeがSEMANTICかHYBRIDか

そのため、最初はBedrock Console/APIでKB単体を確認し、その後GenUで確認するのがよい。

## 5. S3 Vectorsの理解

S3 Vectorsは、S3上でベクトルを保管・検索する仕組み。
Bedrock Knowledge Basesのベクトルストアとして利用できる。

重要な理解:

- semantic search中心。
- hybrid searchは使えない。
- 低コスト・大量データ向き。
- カスタムメタデータ量・キー数に制約がある。
- 本番の厳密検索ではOpenSearchより弱点が出る可能性がある。

向く用途:

- 大量文書の低コストRAG
- 意味検索中心のFAQ
- 厳密なID検索や高度な絞り込みが少ない用途

注意する用途:

- 規程・法令・調達文書
- 年度差分が重要な文書
- 金額・日付・番号が重要な文書
- 組織/権限による厳密なフィルタが必要な用途

## 6. OpenSearch Serverlessの理解

OpenSearch Serverlessは、ベクトル検索だけでなく、キーワード検索やハイブリッド検索を組み合わせやすい検索基盤。

重要な理解:

- 本番寄りの検索チューニングがしやすい。
- 固有名詞、文書ID、日付、金額、番号に強くしやすい。
- メタデータフィルタを活用しやすい。
- S3 Vectorsよりコストは重くなりやすい。
- POC後にcollection削除を忘れると費用が残る可能性がある。

向く用途:

- 自治体規程
- 調達文書
- 文書ID検索
- 旧版/新版管理
- 組織別権限制御
- ハイブリッド検索が必要な業務

## 7. faiss / nmslib の理解

OpenSearch ServerlessをBedrock KBのvector storeとして使う場合、vector indexは `faiss` engineである必要がある。

今回のPOCでは、既存indexを使わず、Bedrock KB作成時にOpenSearch Serverless indexを自動作成させればよい。

気にするべきケース:

- 既存のOpenSearch Serverless collection/indexを流用する
- 過去に作ったindexをそのままKBに接続する
- engineがnmslibのindexを使っている

この場合は、新しいKBを作ってBedrockに自動作成させるか、faiss engineの別indexを作る。

## 8. メタデータフィルタの理解

メタデータフィルタは2段階で考える。

### 8.1 演算子はBedrock KB側にある

例:

- `equals`
- `greaterThan`
- `greaterThanOrEquals`
- `lessThan`
- `lessThanOrEquals`
- `in`
- `notIn`
- `stringContains`
- `listContains`
- `andAll`
- `orAll`

これらはBedrock KBの検索設定で使う。

### 8.2 メタデータ項目は自分で用意する

`status`、`category`、`org`、`doc_id` などは自分で設計して、S3上に `.metadata.json` として置く。

例:

```text
sample.md
sample.md.metadata.json
```

```json
{
  "metadataAttributes": {
    "doc_id": "DOC-SAMPLE-2026",
    "category": "勤務制度",
    "status": "active",
    "effective_from_num": 20260401
  }
}
```

## 9. effective_from の型

`effective_from = "2026-04-01"` のようなISO文字列は、人間には読みやすい。
ただし、比較演算子で型起因のノイズが出る可能性を避けるため、POCでは数値型も併記する。

```json
{
  "effective_from": "2026-04-01",
  "effective_from_num": 20260401
}
```

比較には `effective_from_num` を使う。

## 10. Q19の扱い

v1では本文冒頭に `doc_id` が含まれていたため、Q19はメタデータ検索ではなく本文一致検索になっていた。
v2では対象文書本文から `doc_id` 文字列を外した。

Q19は以下のフィルタと組み合わせて評価する。

```json
{
  "equals": {
    "key": "doc_id",
    "value": "DOC-AI-SLA-2026"
  }
}
```

## 11. 評価で混ぜてはいけないもの

今回の検証では以下を混ぜない。

- Textract
- PDF画像抽出
- 表画像抽出
- 複雑なチャンク戦略
- リランキング
- クエリ分解
- Agent
- 複数KBルーティング

最初は単純に比較する。

```text
同じデータ
同じ質問
同じembedding
同じchunking
違うのはvector storeとsearch type
```

## 12. 検索と生成を分離する

回答が間違ったとき、原因は2種類ある。

| 原因 | 見る指標 |
|---|---|
| 検索で正解チャンクが取れていない | hit_rank, retrieved_source |
| 検索では取れているが回答生成で間違えた | citation_correct, answer_score |

そのため、評価表には `hit_rank` を入れる。

## 13. POC完了条件

最低限、以下ができれば今回のPOCは完了。

- S3 Vectors KBを作成できた
- OpenSearch KBを作成できた
- 同じ評価データを同期できた
- 20問を3系統に投げられた
- SEMANTIC/HYBRIDの差を記録できた
- 正答/誤答/根拠文書/hit_rank/latencyを記録できた
- `status = active` フィルタあり/なしの違いを確認できた
- `doc_id` フィルタをQ19で確認できた
- S3 Vectorsで十分か、OpenSearchが必要かの初期判断ができた


## 14. v3で直したキーワード汚染

v2では、全ノイズ文書の備考に `生成AI`、`RAG`、`Knowledge Base`、`OpenSearch`、`S3 Vectors`、`ハイブリッド検索`、`メタデータ`、`doc_id`、`status`、`effective_from` が入っていた。

これは、Q18、Q19、Q15のようなOpenSearch HYBRIDの強みを見る質問に対して、HYBRID検索だけを不自然に不利にする可能性があった。

v3ではこの備考欄を削除した。
`応答時間` については、Q15の検索衝突を見るためにAI基盤系ノイズ文書内にのみ残している。
そのため、Q15は「AI基盤カテゴリ内で応答時間という語が複数文書に出る中で、正規SLA文書を拾えるか」を見る問題として解釈する。
