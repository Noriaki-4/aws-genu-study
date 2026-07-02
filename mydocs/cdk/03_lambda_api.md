# 第3章 Lambda・API Gateway・IAM

GenU のバックエンドの心臓部 `packages/cdk/lib/construct/api.ts`（約700行）を読めるようになる章です。登場する3要素 — Lambda（処理）、API Gateway（入口）、IAM（権限）— を順に見ます。

## 3.1 NodejsFunction — TypeScript Lambda の定番

`aws-cdk-lib/aws-lambda-nodejs` の `NodejsFunction` は、**TypeScript ソースを esbuild で自動バンドルして Lambda 化**する L2 Construct です。GenU の全 Lambda がこれです。

GenU の実物（`api.ts` より、ストリーミング応答用 Lambda）:

```ts
const predictStreamFunction = new NodejsFunction(this, 'PredictStream', {
  runtime: LAMBDA_RUNTIME_NODEJS,        // Node.js ランタイム（consts.ts で一元管理）
  entry: './lambda/predictStream.ts',    // ソースファイル。ここから import を辿って自動バンドル
  timeout: Duration.minutes(15),
  memorySize: 256,
  environment: {                         // Lambda の環境変数
    MODEL_REGION: modelRegion,
    MODEL_IDS: JSON.stringify(modelIds),
    BUCKET_NAME: fileBucket.bucketName,  // ← 他リソースの属性を渡す（重要パターン）
    // ...
  },
  bundling: bedrockSdkBundling,          // esbuild の細かい設定
});
```

押さえるべきパターン:

- **`entry` が Lambda の実装ファイル** — `packages/cdk/lambda/predictStream.ts` を開けば処理の中身が読める。CDK 定義（インフラ）と Lambda 実装（アプリ）はディレクトリで分かれているが、**entry で対応関係が分かる**
- **`environment` がインフラ→アプリの受け渡し口** — CDK が知っている値（テーブル名、バケット名、モデルID…）を環境変数として注入し、Lambda 実装側は `process.env.BUCKET_NAME` で読む。「Lambda がどの設定に依存しているか」は environment を見れば分かる
- `fileBucket.bucketName` のような**参照はデプロイ時に解決される** — synth 時点では実名が決まっていないため、CDK はトークン（プレースホルダ）を埋め込み、CloudFormation が実際の値に置換する

### GenU 固有の補足

GenU のメイン API (`apiHandler`) は「Express アプリを Lambda Web Adapter で動かすモノリス構成」という応用形です（`entry: './lambda/api/index.ts'`, `handler: 'run.sh'`）。個別エンドポイントの実装は `lambda/api/` 配下の Express ルーターにあります。一方 `predictStream.ts` など一部は独立した Lambda です。

## 3.2 API Gateway — REST API の定義

### RestApi 本体と Cognito 認可

`api.ts` の該当部分（実物・簡略化）:

```ts
// Cognito UserPool のトークンを検証する authorizer
const authorizer = new CognitoUserPoolsAuthorizer(this, 'Authorizer', {
  cognitoUserPools: [userPool],
});

const commonAuthorizerProps: Partial<MethodOptions> = {
  authorizationType: AuthorizationType.COGNITO,
  authorizer,
};

// Lambda との統合（proxy: true = リクエストを丸ごと Lambda に渡す）
const lambdaIntegration = new LambdaIntegration(apiHandler, { proxy: true, ... });

const api = new RestApi(this, 'Api', {
  defaultIntegration: lambdaIntegration,        // 全メソッドのデフォルトの処理先
  defaultMethodOptions: commonAuthorizerProps,  // 全メソッドにデフォルトで Cognito 認可
  defaultCorsPreflightOptions: {
    allowOrigins: Cors.ALL_ORIGINS,
    allowMethods: Cors.ALL_METHODS,
    allowCredentials: true,
  },
  ...
});
```

`defaultMethodOptions` に authorizer を設定しているため、**全 API が「Cognito ログイン済みユーザーのみ」に自動的に保護**されます。フロントエンド（React 教科書 第8章）で axios interceptor が付与していた `Authorization: <idToken>` ヘッダーを検証するのがこの authorizer です。フロントとバックが Cognito を介して繋がる場所です。

### リソースとメソッド = URL パスと HTTP メソッド

```ts
const predict = api.root.addResource('predict');   // POST /predict
predict.addMethod('POST');
predict.addResource('title').addMethod('POST');    // POST /predict/title

const chats = api.root.addResource('chats');
chats.addMethod('POST');                           // POST /chats（チャット作成）
chats.addMethod('GET');                            // GET  /chats（一覧）
const chat = chats.addResource('{chatId}');        // {} はパスパラメータ
chat.addMethod('GET');                             // GET  /chats/{chatId}
```

**フロントエンドの `useChatApi.ts` が呼んでいた URL は、ここで定義されています。** 「API を1本足したい」= ここに addResource / addMethod を足し、Lambda 側（`lambda/api/` の Express ルーター）に実装を足す、が基本セットです。

## 3.3 IAM — grant メソッドによる権限付与

AWS では「Lambda が DynamoDB を読む」ためにも IAM 権限が必要です。CDK の L2 Construct は **grant 系メソッド**でこれを1行にします。

```ts
// GenU の実コード（api.ts）
table.grantReadWriteData(apiHandler);        // apiHandler に table の読み書き権限
props.statsTable.grantReadWriteData(apiHandler);
fileBucket.grantReadWrite(apiHandler);       // S3 の読み書き
predictStreamFunction.grantInvoke(idPool.authenticatedRole);
// ↑「ログイン済みユーザー」に predictStream の直接呼び出しを許可
//   （ストリーミングは API Gateway を通さずフロントから Lambda を直接叩くため）
```

grant の向きは「**リソース.grantXxx(主体)** = このリソースへの Xxx 権限を、主体に与える」です。裏では主体（Lambda の実行ロールなど）に IAM ポリシーが自動で追記されます。

### PolicyStatement — grant で表現できない権限

Bedrock のようにリソース参照ではない権限は、素の `PolicyStatement` で書きます（実物・抜粋）:

```ts
const bedrockPolicy = new PolicyStatement({
  effect: Effect.ALLOW,
  resources: ['*'],
  actions: ['bedrock:*', 'logs:*', ...],
});
apiHandler.role?.addToPrincipalPolicy(bedrockPolicy);
predictStreamFunction.role?.addToPrincipalPolicy(bedrockPolicy);
```

「Lambda が Bedrock を呼べない（AccessDenied）」系のエラーはこの辺りを確認します。

## 3.4 この章の3要素の関係まとめ

```
[フロントエンド]
   │ Authorization ヘッダー付き HTTP
   ▼
[API Gateway RestApi]
   ├── CognitoUserPoolsAuthorizer … トークン検証（= 認証）
   ├── addResource/addMethod      … URL 定義
   └── LambdaIntegration          … 処理先の指定
        ▼
[NodejsFunction (apiHandler)]
   ├── entry: lambda/api/index.ts … 実装本体
   ├── environment                … インフラ値の注入
   └── 実行ロール ← grant / PolicyStatement で権限付与（= 認可）
        ▼
[DynamoDB / S3 / Bedrock]
```

---

## 練習問題 3

**問3-1.** `NodejsFunction` の `entry` と `environment` はそれぞれ何のためにありますか。「Lambda の実装コードが何のリソースに依存しているか」を調べたいとき、どちらを見ますか。

**問3-2.** 次の要件を CDK コードで書いてください。「`./lambda/hello.ts` を Lambda 化し、`GET /hello` で呼べる REST API を作る。認可は無しでよい」（RestApi, LambdaIntegration, addResource, addMethod を使用）

**問3-3.** `table.grantReadWriteData(fn)` は何をしていますか。これを書き忘れて Lambda が DynamoDB にアクセスするとどうなり、どんなエラーからどう気づけますか。

**問3-4.** GenU に `GET /chats/{chatId}/messages` というエンドポイントがあります。(1) `api.ts` でこのパスを定義している箇所を見つけ、(2) フロントエンドの `packages/web/src/hooks/useChatApi.ts` でこれを呼んでいる関数を見つけて、フロント→インフラ→実装の対応を確認してください。

**問3-5.** `predictStreamFunction.grantInvoke(idPool.authenticatedRole)` は、他の Lambda と違い「API Gateway 経由ではない」呼び出し経路を示唆しています。なぜチャットのストリーミング応答だけこの構成なのか、推測してください（ヒント: API Gateway REST API のレスポンスの性質）。

[解答例](./answers.md#第3章)
