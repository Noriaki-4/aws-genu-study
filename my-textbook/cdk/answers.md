# 練習問題 解答例

必ず自分で解いてから読んでください。行番号はいずれも 2026年7月時点の main ブランチのものです。

## 第1章

**問1-1.** CDK コード（TypeScript）は synth によって CloudFormation テンプレート（JSON）に変換され、`packages/cdk/cdk.out/` に出力される。AWS への反映はこのテンプレートを CloudFormation サービスが適用することで行われる。

**問1-2.**
- App: デプロイ対象全体の入れ物。App 自体はデプロイ単位ではない
- Stack: **デプロイ・削除の単位**。`cdk deploy StackName` / `cdk destroy StackName` で個別に操作できる
- Construct: Stack 内の部品。単独ではデプロイ・削除できない

「あるリソースだけ削除したい」場合、リソース単位の削除コマンドは無い。**コードからそのリソースの定義を消して deploy する**（差分としての削除）か、リソースが属する Stack ごと destroy するかになる。

**問1-3.** CloudFormation の更新には「in-place 更新」と「replacement（新規作成→旧削除）」がある。replacement が起きるとリソースが作り直され、DynamoDB や S3 なら**データ消失**、リソース名変更による参照切れなどが発生しうる。`cdk diff` は変更対象と種別（`[~]` 更新、`[-]/[+]` 作り直し・削除）を事前に表示するため、意図しない破壊的変更に deploy 前に気づける。

**問1-4.** (1) `npx ts-node --prefer-ts-exts bin/generative-ai-use-cases.ts`。(2) 例: `env`, `ragEnabled`, `ragKnowledgeBaseEnabled`, `selfSignUpEnabled`, `samlAuthEnabled`, `modelRegion`, `modelIds` など。`ragEnabled` の初期値は `false`。

## 第2章

**問2-1.**
- 第1引数 `this`（scope）: 親 Construct。このリソースがツリー上のどこに属するかを決める
- 第2引数 `'MyBucket'`（id）: 親の中で一意な識別子。CloudFormation 論理IDの元になる
- 第3引数 props: バケットの設定（暗号化、バージョニングなど）

**問2-2.** CloudFormation は論理IDでリソースを同一視するため、id 変更は「旧 `Table` の削除 + 新 `MainTable` の新規作成」になる。DynamoDB テーブルは作り直され、**保存されていたデータがすべて失われる**（またはポリシーによっては削除に失敗してエラーになる）。

**問2-3.** 複数リソースの定番構成の再利用は**自作 Construct**（または既製の L3）が適切。

```ts
import { Construct } from 'constructs';
import { Bucket, BucketEncryption, BlockPublicAccess } from 'aws-cdk-lib/aws-s3';
import { RemovalPolicy } from 'aws-cdk-lib';

export interface CleanBucketProps {
  readonly versioned?: boolean;
}

export class CleanBucket extends Construct {
  public readonly bucket: Bucket;

  constructor(scope: Construct, id: string, props: CleanBucketProps = {}) {
    super(scope, id);
    this.bucket = new Bucket(this, 'Bucket', {
      encryption: BucketEncryption.S3_MANAGED,
      blockPublicAccess: BlockPublicAccess.BLOCK_ALL,
      removalPolicy: RemovalPolicy.DESTROY,
      autoDeleteObjects: true,     // 「中身を空にしてから削除」
      versioned: props.versioned,
    });
  }
}
```

なお GenU の `transcribe.ts` の `AudioBucket` がまさにこの設定パターンの実例。

**問2-4.**
1. S3 バケット2つ（`AudioBucket` = 録音ファイル置き場、`TranscriptBucket` = 文字起こし結果置き場。どちらも暗号化・SSL 強制・パブリックアクセス遮断・DESTROY 設定）と、ログインユーザーに Transcribe/Translate 操作を許可する IAM ポリシー
2. `public readonly audioBucket` と `public readonly transcriptBucket`
3. `generative-ai-use-cases-stack.ts` の 192行目付近で `new Transcribe(this, 'Transcribe', {...})` が作られ、243〜244行目付近で `audioBucket` / `transcriptBucket` が `Api` construct の props に渡されている（API の Lambda が音声ファイルにアクセスするため）。622行目付近では CfnOutput にも出力されている

**問2-5.** SAML 認証（企業の IdP でログイン）を使う場合、誰でもメールアドレスで登録できるセルフサインアップが併存すると、IdP の統制を迂回してアカウントを作れてしまいセキュリティホールになる。この組み合わせ禁止を「設定者の注意」に頼らず**コードで強制**することで、cdk.json でどう設定されても危険な構成がデプロイされないことを保証できる（実際 auth.ts に「Be aware of security」のコメントがある）。

## 第3章

**問3-1.**
- `entry`: Lambda の実装ソースファイルの場所。esbuild がここを起点にバンドルする
- `environment`: Lambda 実行時の環境変数。CDK が決めた値（バケット名・テーブル名等）をアプリコードへ渡す唯一の受け渡し口

依存リソースを調べたいときは **environment を見る**（実装側は `process.env.XXX` で読むため、environment に列挙されたものがほぼ依存の一覧になる）。

**問3-2.**

```ts
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { RestApi, LambdaIntegration } from 'aws-cdk-lib/aws-apigateway';

const helloFunction = new NodejsFunction(this, 'Hello', {
  entry: './lambda/hello.ts',
});

const api = new RestApi(this, 'Api');
const hello = api.root.addResource('hello');
hello.addMethod('GET', new LambdaIntegration(helloFunction));
```

**問3-3.** `fn` の実行ロールに、`table` に対する読み書き（GetItem, PutItem, Query など）を許可する IAM ポリシーを自動追加する。書き忘れると Lambda 実行時に **AccessDeniedException**（`... is not authorized to perform: dynamodb:GetItem on resource ...`）が発生する。気づき方: フロントには 500 エラーが返る → CloudWatch Logs で該当 Lambda のログを見ると AccessDenied とアクション名・リソース名が出ているので、api.ts に grant を追加する。

**問3-4.**
1. `api.ts` 689〜691行目付近: `const messages = chat.addResource('messages'); messages.addMethod('GET'); messages.addMethod('POST');`（`chat` は `chats.addResource('{chatId}')`）
2. `useChatApi.ts` の `listMessages`（92行目付近）が `http.get<ListMessagesResponse>('chats/${chatId}/messages')` を呼んでいる。フロントのフック → API Gateway のリソース定義 → apiHandler（`lambda/api/` の Express ルーター）という対応

**問3-5.** API Gateway REST API はレスポンスをバッファリングするため、LLM の「生成しながら1文字ずつ返す」ストリーミング応答に向かない（当時はストリーミング非対応、タイムアウト 29 秒制限もある）。そこでストリーミングだけは Lambda の**関数 URL / SDK 経由の直接呼び出し（InvokeWithResponseStream）**を使う。API Gateway を通らないため Cognito authorizer で守れず、代わりに「ログイン済みユーザーの IAM ロール（idPool.authenticatedRole）に grantInvoke」して認可している。

## 第4章

**問4-1.** CloudFront に紐づける WAF (WAFv2 の CLOUDFRONT スコープ) は **us-east-1 リージョンにしか作成できない**。1スタックは1リージョンにしかデプロイできないため、本体スタック（任意リージョン）とは別スタックにし、`crossRegionReferences: true` で WebACL の ARN を本体へ渡している。

**問4-2.** `packages/cdk/parameter.ts` の `envs` に環境ごとの設定を書く（`dev: {...}, prod: {...}`）。デプロイ時に `npm run cdk:deploy -- -c env=dev` のように `-c env=環境名` を指定すると、その環境の設定が使われ、スタック名にも `dev` が付く（`GenerativeAiUseCasesStackdev`）。名前が違うので同一アカウント・同一リージョンでも衝突せず共存できる。

**問4-3.** `stack-input.ts` で `ragEnabled` は `z.boolean()` と定義されているため、文字列 `"true"` は **synth 時（deploy コマンド実行直後）に zod のバリデーションエラー**で落ちる。AWS に何かが中途半端に反映される前、ローカルで気づける。これが zod を挟む利点。

**問4-4.** デプロイ時に各スタックが `CfnOutput`（ApiEndpoint, UserPoolId など）を出力している。ローカル開発コマンド `npm run web:devw` が実行する `setup-env.sh` が、CloudFormation の API（describe-stacks）でこの Outputs を取得し、`VITE_APP_API_ENDPOINT` などの環境変数に変換してから Vite を起動する。つまり CDK の `new CfnOutput(this, 'ApiEndpoint', ...)`（generative-ai-use-cases-stack.ts）がフロントのローカル開発の設定源。

**問4-5.**
1. 条件: `params.ragKnowledgeBaseEnabled && !params.ragKnowledgeBaseId`（Knowledge Base を有効にしていて、かつ既存の KB ID を指定していないとき新規作成する）
2. `create-stacks.ts` 168行目付近で、`knowledgeBaseId: ragKnowledgeBaseStack?.knowledgeBaseId` と `knowledgeBaseDataSourceBucketName` が `GenerativeAiUseCasesStack` の props に渡され、本体スタック内で RAG 用 API（`api.ts` 経由で Lambda の環境変数 `KNOWLEDGE_BASE_ID`）に配線される

## 第5章

**問5-1.** デプロイ時に、`web.ts` の `NodejsBuild` が（CodeBuild 上で）`npm ci && npm run web:build` を実行してビルドし、成果物を Web 配信用 S3 バケットへ配置、CloudFront キャッシュも無効化する。API エンドポイントや機能フラグなどの環境変数（`buildEnvironment`）は CDK がその場で注入するため、**「正しい設定でビルドして正しい場所に置く」作業全体がデプロイに内包**されており、手動ビルド・手動アップロードは不要。

**問5-2.** チャット履歴（DynamoDB テーブル）も**削除される**。`create-stacks.ts` の `DeletionPolicySetter`（IAspect 実装）が `cdk.Aspects.of(generativeAiUseCasesStack).add(new DeletionPolicySetter(cdk.RemovalPolicy.DESTROY))` により全リソースの RemovalPolicy を DESTROY に設定しているため。

**問5-3.** 例:
1. **設定の伝播**: `create-stacks.ts` で GuardrailStack が生成される条件と、`guardrailIdentifier` / `guardrailVersion` が本体スタックの props に渡っているかを確認
2. **Lambda への注入**: `api.ts` で該当の環境変数（guardrail 関連）が predictStream などの `environment` に入っているか確認
3. **実行時**: CloudWatch Logs で推論 Lambda のログを確認（Guardrail 適用時のエラーや、環境変数が空になっていないか）。加えてそもそも `cdk diff` で GuardrailStack がデプロイ対象になっていたか、モデルリージョンの一致も確認ポイント

**問5-4.**
1. `stack-input.ts` の `selfSignUpEnabled: z.boolean().default(true)` で受け取られ、
2. `auth.ts` の `UserPool` 定義（`selfSignUpEnabled: props.samlAuthEnabled ? false : props.selfSignUpEnabled`）で Cognito のセルフサインアップが無効になる。また `web.ts` の `buildEnvironment` で `VITE_APP_SELF_SIGN_UP_ENABLED: 'false'` としてフロントに渡り、
3. フロントの `AuthWithUserpool.tsx` がこれを読んでログイン画面から「アカウント作成」タブを非表示にする。ユーザーには「サインアップできない（管理者にアカウントを作ってもらう運用）」として見える。インフラ側（Cognito）と UI 側の両方が同じ設定から制御されている点がポイント。

**問5-5.** 変更が必要なファイル:
1. **`packages/cdk/lib/construct/api.ts`** — `api.root.addResource('health').addMethod('GET');` を追加（defaultIntegration と defaultMethodOptions により、自動的に apiHandler 行き + Cognito 認可付きになる）
2. **`packages/cdk/lambda/api/` 配下（Express ルーター）** — `GET /health` に対して `{ status: 'ok' }` を返すハンドラを追加（ルーティング定義ファイルに1ルート追加）
3. （フロントから呼ぶ場合のみ）**`packages/web/src/hooks/useHttp.ts` を使う API フック** — `http.get('health')` を呼ぶ関数を追加

デプロイは `npm run cdk:deploy`。認可付きなので、確認はフロント経由か、Cognito の idToken を Authorization ヘッダーに付けた curl で行う。
