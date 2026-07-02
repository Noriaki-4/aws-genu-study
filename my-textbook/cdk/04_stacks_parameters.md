# 第4章 マルチスタック構成とパラメータ管理

## 4.1 GenU のスタック一覧

GenU は1つの巨大スタックではなく、**条件に応じて複数のスタックを組み立てる**構成です。`lib/create-stacks.ts` が組み立て役で、`cdk deploy --all` で必要なものが全部デプロイされます。

| スタック | 役割 | 作られる条件 | リージョン |
|----------|------|-------------|-----------|
| `GenerativeAiUseCasesStack` | 本体（Web, Auth, API, DB…） | 常に | 指定リージョン |
| `CloudFrontWafStack` | WAF（IP/地域制限） | IP 制限などが設定されたとき | **us-east-1 固定** |
| `RagKnowledgeBaseStack` | Bedrock Knowledge Base | `ragKnowledgeBaseEnabled` | modelRegion |
| `AgentStack` | Bedrock Agent | `agentEnabled` など | modelRegion |
| `GuardrailStack` | Bedrock Guardrail | `guardrailEnabled` | modelRegion |
| `DashboardStack` | CloudWatch ダッシュボード | `dashboard` | modelRegion |
| `AgentCoreStack` ほか | AgentCore 系 | 各フラグ | それぞれ |

## 4.2 なぜスタックを分けるのか

`create-stacks.ts` のコメントに理由が書かれています。代表例:

```ts
// WAF v2 is only deployable in us-east-1, so the Stack is separated
const cloudFrontWafStack = (params.allowedIpV4AddressRanges || ...) 
  ? new CloudFrontWafStack(app, `CloudFrontWafStack${params.env}`, {
      env: {
        account: params.account,
        region: 'us-east-1',        // ← スタックごとにリージョンを指定できる
      },
      params: params,
      crossRegionReferences: true,
    })
  : null;
```

分割の理由は主に3つ:

1. **リージョンの制約** — CloudFront 用 WAF は us-east-1 にしか置けない。**1スタック = 1リージョン**なので、リージョンが違うリソースはスタックを分けるしかない
2. **ライフサイクルの分離** — RAG や Agent はオプション機能。有効/無効の切り替えで本体スタックを作り直したくない
3. **条件付き生成** — `params.xxxEnabled ? new XxxStack(...) : null` の形で、設定によりスタックごと存在を切り替える（第9章のルート定義と同じ発想）

スタック名に `${params.env}` が付く点にも注目してください。`env: "dev"` なら `GenerativeAiUseCasesStackdev` になり、**同じアカウントに複数環境を共存**できます（4.4節）。

## 4.3 スタック間の値の受け渡し

スタックが分かれると「WAF スタックで作った WebACL の ARN を本体スタックで使う」といった**スタック間参照**が必要になります。GenU では2つの方法が登場します。

### (1) crossRegionReferences + props 渡し

TypeScript 上は単純に「スタック A のプロパティをスタック B のコンストラクタに渡す」だけです。

```ts
const generativeAiUseCasesStack = new GenerativeAiUseCasesStack(app, `...`, {
  ...
  webAclId: cloudFrontWafStack?.webAclArn,   // WAF スタックの成果物を渡す
  cert: cloudFrontWafStack?.cert,
  crossRegionReferences: true,               // リージョン跨ぎの参照を許可
});
```

裏では CloudFormation の Export/Import（+ リージョン跨ぎの場合はカスタムリソース）が自動生成されます。

### (2) RemoteOutputs（cdk-remote-stack）

Export/Import には「**参照されている Export は変更・削除できない**（デッドロック）」という弱点があります。GenU は一部でこれを避けるため、`cdk-remote-stack` の `RemoteOutputs`（デプロイ時に相手スタックの Outputs を API で読む方式）を使っています。

```ts
// generative-ai-use-cases-stack.ts（実物）
const agentRemoteOutputs = new RemoteOutputs(this, 'AgentRemoteOutputs', {
  stack: props.agentStack,
  alwaysUpdate: true,
  timeout: Duration.seconds(600),
});
agentsJson = agentRemoteOutputs.get(REMOTE_OUTPUT_KEYS.AGENTS);
```

初読では「Export/Import の副作用を避けるための、スタック間参照のもう1つの手段」と理解できれば十分です。

### CfnOutput — 外部への出力

`CfnOutput` はスタックの「Outputs」を定義します。GenU では大量にあります。

```ts
new CfnOutput(this, 'WebUrl', { value: ... });
new CfnOutput(this, 'ApiEndpoint', { value: api.api.url });
new CfnOutput(this, 'UserPoolId', { value: auth.userPool.userPoolId });
```

これが重要なのは、**フロントエンドのローカル開発（`npm run web:devw`）がこの Outputs を CloudFormation API から取得して環境変数（`VITE_APP_API_ENDPOINT` など）に変換している**からです。「CDK の Outputs → フロントの env」という接続を覚えておくと、フロント・インフラ間の設定齟齬を調査できます。

## 4.4 パラメータ管理: cdk.json → zod → 各スタック

GenU の設定値（50個以上）は次のパイプラインで流れます。

```
cdk.json の "context"（デフォルト設定ファイル）
  or parameter.ts の envs（環境ごとの上書き定義）
  or -c コマンドライン引数（一時的な上書き）
      │
      ▼
parameter.ts の getParams()
      │  stackInputSchema.parse(...)   ← zod でバリデーション + デフォルト値適用
      ▼
ProcessedStackInput（型安全な設定オブジェクト）
      │
      ▼
createStacks(app, params) → 各スタック → 各 Construct の props へ
```

### cdk.json の context

```json
"context": {
  "env": "",
  "ragEnabled": false,
  "selfSignUpEnabled": true,
  "modelRegion": "us-east-1",
  "modelIds": [ ... ],
  ...
}
```

**GenU のデプロイ設定変更 = 基本的にこのファイルの編集**です（公式ドキュメント `docs/ja/DEPLOY_OPTION.md` が全項目のリファレンス）。

### stack-input.ts の zod スキーマ

```ts
const baseStackInputSchema = z.object({
  account: z.string().default(process.env.CDK_DEFAULT_ACCOUNT ?? ''),
  region: z.string().default(process.env.CDK_DEFAULT_REGION ?? 'us-east-1'),
  env: z.string().default(''),
  selfSignUpEnabled: z.boolean().default(true),
  allowedSignUpEmailDomains: z.array(z.string()).nullish(),
  ...
});
```

zod により (1) 型の間違った設定は synth 時に即エラー、(2) 未指定項目にはデフォルト値、が保証されます。**「この設定項目はどんな型で何がデフォルトか」を知りたければ stack-input.ts を見る**のが最速です。

### parameter.ts の envs — 環境の切り替え

```ts
const envs: Record<string, Partial<StackInput>> = {
  dev:     { /* 開発環境のパラメータ */ },
  staging: { /* ... */ },
  prod:    { /* ... */ },
};
```

`-c env=dev` を付けてデプロイすると、cdk.json の代わりに `envs.dev` の内容が使われ、スタック名も `...Stackdev` になります。**dev / staging / prod を同一アカウントに並べられる**仕組みです。

```bash
npm run cdk:deploy -- -c env=dev
```

---

## 練習問題 4

**問4-1.** GenU で CloudFrontWafStack が本体スタックと分かれている理由を説明してください。

**問4-2.** 「同じ AWS アカウントに開発用と本番用の GenU を共存させたい」とき、GenU が用意している仕組みを使った手順を説明してください（どのファイルに何を書き、どうデプロイするか）。

**問4-3.** `cdk.json` の context に `"ragEnabled": "true"`（文字列）と書いてしまいました。GenU では何が起き、どの時点で気づけますか（stack-input.ts の仕組みを踏まえて）。

**問4-4.** フロントエンドのローカル開発時、`VITE_APP_API_ENDPOINT` のような環境変数はどこから来ますか。CDK 側のどのコードと繋がっているか、経路を説明してください。

**問4-5.** GenU の `lib/create-stacks.ts` を開き、(1) `RagKnowledgeBaseStack` が作られる条件、(2) そのスタックの成果物（何か）が本体スタックへどう渡されているか（`GenerativeAiUseCasesStack` の props を確認）、を調べてください。

[解答例](./answers.md#第4章)
