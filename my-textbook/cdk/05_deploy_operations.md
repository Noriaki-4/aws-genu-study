# 第5章 デプロイ運用と GenU CDK の歩き方

## 5.1 Web 配信の仕組み — フロントエンドはどうデプロイされるのか

`packages/cdk/lib/construct/web.ts` は、React アプリ（packages/web）を配信する部分です。2つの部品が中心です。

### CloudFrontToS3（L3 / AWS Solutions Constructs）

「S3 に置いた静的ファイルを CloudFront (CDN) で配信する」定番構成を1つの Construct にしたものです。バケット、ディストリビューション、アクセス制御（ユーザーは S3 に直接アクセスできず CloudFront 経由のみ）などがまとめて構築されます。

### NodejsBuild（deploy-time build）

**「デプロイのタイミングでフロントエンドをビルドする」**仕掛けです（実物・抜粋）:

```ts
const build = new NodejsBuild(this, 'BuildWeb', {
  nodejsVersion: 22,
  assets: [{ path: '../../', exclude: [...] }],   // リポジトリ全体をアセット化
  buildCommands: ['npm ci', 'npm run web:build'],  // ビルドコマンド
  buildEnvironment: {
    // ← CDK が知っている値を Vite の環境変数として注入
    VITE_APP_API_ENDPOINT: props.apiEndpointUrl,
    VITE_APP_USER_POOL_ID: props.userPoolId,
    VITE_APP_RAG_ENABLED: props.ragEnabled.toString(),
    ...
  },
  destinationBucket: ...,   // ビルド成果物を Web 配信用バケットへ
  distribution: ...,        // デプロイ後に CloudFront キャッシュを無効化
});
```

これが重要な理由:

1. **React 教科書第9章の機能フラグ（`VITE_APP_XXX_ENABLED`）の供給源はここ**。cdk.json の設定 → CDK props → `buildEnvironment` → Vite ビルド → フロントの `import.meta.env` という一本道が繋がる
2. フロントだけ直したいときも、環境変数を変えたいときも、**デプロイし直せばビルドからやり直される**（手元でビルドして S3 に置く作業は不要）

## 5.2 RemovalPolicy と Aspects

### RemovalPolicy — スタック削除時にリソースをどうするか

ステートフルなリソース（S3, DynamoDB など）は、既定では**スタックを消してもリソースが残る**（RETAIN）ことがあります。データ保護のためです。

GenU は PoC・検証用途を想定し、`create-stacks.ts` で**全リソースを DESTROY（スタック削除と共に削除）に一括設定**しています:

```ts
class DeletionPolicySetter implements cdk.IAspect {
  constructor(private readonly policy: cdk.RemovalPolicy) {}
  visit(node: IConstruct): void {
    if (node instanceof cdk.CfnResource) {
      node.applyRemovalPolicy(this.policy);
    }
  }
}

cdk.Aspects.of(generativeAiUseCasesStack).add(
  new DeletionPolicySetter(cdk.RemovalPolicy.DESTROY)
);
```

### Aspects — ツリー全体への一括適用

`Aspects` は「Construct ツリーの全ノードを訪問して処理を適用する」仕組みです（Visitor パターン）。第1章で見た `cdk.Tags.of(app).add(...)` も同じ仕組みで、「全リソースにタグを付ける」「全リソースの削除ポリシーを変える」のような横断的な設定に使われます。

**運用上の含意**: GenU をそのまま本番運用する場合、`cdk destroy` で**チャット履歴の DynamoDB もファイルの S3 も消えます**。`docs/ja/DESTROY.md` を必ず読んでから destroy してください。

## 5.3 デプロイ運用コマンド集

```bash
# 通常デプロイ（全スタック）
npm run cdk:deploy

# 高速デプロイ（開発中の反復用: 承認スキップ・並列化）
npm run cdk:deploy:quick

# 差分確認（habit にする）
npm run cdk:diff

# 環境指定
npm run cdk:deploy -- -c env=dev

# 全削除
npm run cdk:destroy
```

デプロイには数分〜数十分かかります（初回は特に。NodejsBuild のフロントビルドや CloudFront の反映が長い）。

### よくあるトラブルと見る場所

| 症状 | 見る場所 |
|------|---------|
| synth 段階でエラー（deploy 前に落ちる） | TypeScript のエラーメッセージ。設定値の型違いなら zod のエラー（stack-input.ts） |
| CREATE_FAILED / UPDATE_FAILED | CloudFormation コンソール → 該当スタック → イベントタブ |
| デプロイは成功したが画面で機能が出ない | cdk.json の該当フラグ → `web.ts` の `buildEnvironment` に渡っているか → フロントの `import.meta.env` |
| Lambda 実行時エラー（AccessDenied 等） | CloudWatch Logs の該当関数ロググループ → 権限なら `api.ts` の grant / PolicyStatement |
| bootstrap 系エラー | 対象リージョンで `cdk bootstrap` 済みか（us-east-1 と modelRegion に注意） |

## 5.4 GenU CDK ディレクトリの歩き方

```
packages/cdk/
├── bin/generative-ai-use-cases.ts   # エントリーポイント（第1章）
├── cdk.json                         # デプロイ設定（context）★一番よく触る
├── parameter.ts                     # 環境別設定（envs）
├── consts.ts                        # 定数（Lambda ランタイムなど）
├── lib/
│   ├── create-stacks.ts             # スタックの組み立て（第4章）
│   ├── generative-ai-use-cases-stack.ts  # 本体スタック（Construct の配線係）
│   ├── *-stack.ts                   # オプション機能のスタック群
│   ├── stack-input.ts               # 設定の zod スキーマ（型とデフォルト値）
│   └── construct/                   # 部品の実装（第2〜3章）
│       ├── auth.ts                  #   Cognito
│       ├── api.ts                   #   API Gateway + Lambda 群 ★最重要
│       ├── database.ts              #   DynamoDB
│       ├── web.ts                   #   CloudFront + S3 + フロントビルド
│       ├── rag.ts / agent.ts ...    #   オプション機能
│       └── index.ts                 #   再エクスポート
├── lambda/                          # Lambda の実装（アプリケーションコード）
│   ├── api/                         #   メイン API（Express モノリス）
│   ├── predictStream.ts             #   ストリーミング推論
│   └── ...
├── rag-docs/                        # RAG のサンプルドキュメント
└── test/                            # スナップショットテスト
```

### 目的別・コードの探し方

| やりたいこと | 見るファイル |
|--------------|-------------|
| 機能の ON/OFF・モデル変更などデプロイ設定 | `cdk.json` の context（リファレンスは `docs/ja/DEPLOY_OPTION.md`） |
| 設定項目の型・デフォルト値を知る | `lib/stack-input.ts` |
| どのスタックがいつ作られるか | `lib/create-stacks.ts` |
| API エンドポイントの定義 | `lib/construct/api.ts`（addResource / addMethod） |
| API の処理の中身 | `lambda/api/` ほか `lambda/` 配下（NodejsFunction の entry で対応を確認） |
| Lambda の権限エラー | `lib/construct/api.ts` の grant / PolicyStatement |
| 認証まわり（パスワードポリシー、サインアップ制限） | `lib/construct/auth.ts` |
| フロントに渡る環境変数 | `lib/construct/web.ts` の `buildEnvironment` |
| DynamoDB のキー設計 | `lib/construct/database.ts` |

## 5.5 総仕上げ: 設定1つが反映されるまでの旅

`cdk.json` で `"ragEnabled": true` にしたときの流れを追うと、この教科書の全章が繋がります。

```
cdk.json: "ragEnabled": true
  → parameter.ts getParams() が読み込み、stack-input.ts の zod で検証   （第4章）
  → generative-ai-use-cases-stack.ts: params.ragEnabled が true なので
      new Rag(this, 'Rag', {...}) が実行される（条件付き Construct 生成） （第2章）
      Kendra インデックス等のリソースが定義される
  → api.ts: RAG 用のエンドポイントと Lambda に権限が配線される           （第3章）
  → web.ts: buildEnvironment に VITE_APP_RAG_ENABLED: 'true' が渡る      （第5章）
  → デプロイ時に NodejsBuild がフロントを再ビルド
  → フロント main.tsx: ragEnabled が true なので /rag ルートが生成され、
      サイドバーに「RAG チャット」が現れる                    （React 教科書 第9章）
```

---

## 練習問題 5

**問5-1.** GenU では「フロントエンドのビルド」はいつ・どこで実行されますか。ローカルで `npm run web:build` して S3 にアップロードする作業が不要な理由を説明してください。

**問5-2.** `cdk destroy` すると GenU のチャット履歴はどうなりますか。その挙動を決めているコードを `create-stacks.ts` から特定してください。

**問5-3.** 「cdk.json で `guardrailEnabled: true` にしてデプロイしたのに Guardrail が効いていない気がする」— 調査すべき箇所を、この章の「よくあるトラブルと見る場所」と第4章の知識を使って3つ挙げてください。

**問5-4.**（読解総合）`cdk.json` の `selfSignUpEnabled` を `false` にしたとき、(1) どのファイルの (2) どのコードに影響し、(3) ユーザーにはどう見えるか。`stack-input.ts` → スタック → `auth.ts` → フロントの経路で説明してください（フロント側は `VITE_APP_SELF_SIGN_UP_ENABLED` を grep）。

**問5-5.**（改修演習・手を動かす）GenU に「新しい API エンドポイント `GET /health`（認可あり、'ok' を返すだけ）」を追加するとしたら、変更が必要なファイルをすべて列挙し、それぞれ何を書くか概要を説明してください。実装まではしなくてOK。

[解答例](./answers.md#第5章)
