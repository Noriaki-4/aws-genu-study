# 第1章 CDK の基本概念とライフサイクル

## 1.1 CDK とは

AWS CDK (Cloud Development Kit) は、**AWS インフラを TypeScript などのプログラミング言語で定義するツール**です。

CloudFormation（AWS 純正の IaC。YAML/JSON でリソースを定義）には次の課題がありました。

- YAML が巨大になる（GenU 規模なら数万行）
- ループ・条件分岐・再利用が書きにくい

CDK は「**TypeScript でリソースを定義 → CloudFormation テンプレートを自動生成 → CloudFormation がデプロイ**」という仕組みでこれを解決します。

```
TypeScript コード（packages/cdk/）
    │  cdk synth（合成）
    ▼
CloudFormation テンプレート（cdk.out/ に生成される JSON）
    │  cdk deploy
    ▼
AWS 上の実リソース（CloudFormation スタックとして管理）
```

重要な帰結:

- **CDK コードは「実行するとテンプレートを吐くプログラム」**であり、コード自体が AWS 上で動くわけではない
- デプロイ結果は AWS コンソールの CloudFormation 画面で確認できる（スタック一覧・イベント・エラー）
- `if` 文や三項演算子で「この設定のときだけリソースを作る」が自然に書ける（GenU が多用するパターン）

## 1.2 3つの基本単位: App / Stack / Construct

```
App（アプリ全体。デプロイの単位の集合）
└── Stack（CloudFormation スタック 1つに対応。デプロイ・削除の単位）
    └── Construct（リソースのまとまり。再利用可能な部品）
        └── リソース（S3 バケット、Lambda 関数など）
```

| 単位 | 役割 | GenU での例 |
|------|------|------------|
| `App` | ルート。全スタックを束ねる | `bin/generative-ai-use-cases.ts` の `new cdk.App()` |
| `Stack` | デプロイ単位。リソースの生死を共にする | `GenerativeAiUseCasesStack`, `RagKnowledgeBaseStack` など |
| `Construct` | 関連リソースをまとめた部品 | `Auth`（Cognito 一式）, `Api`（API Gateway + Lambda 一式） |

## 1.3 GenU のエントリーポイントを読む

CDK アプリの起点は `cdk.json` の `app` に書かれています。

```json
{ "app": "npx ts-node --prefer-ts-exts bin/generative-ai-use-cases.ts" }
```

つまり `cdk` コマンドは `bin/generative-ai-use-cases.ts` を実行します。その全文（実物）:

```ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { getParams } from '../parameter';
import { createStacks } from '../lib/create-stacks';
import { TAG_KEY } from '../consts';

const app = new cdk.App();                 // (1) App を作る
const params = getParams(app);             // (2) 設定値を読み込む（第4章）
if (params.tagValue) {
  const tagKey = params.tagKey || TAG_KEY;
  cdk.Tags.of(app).add(tagKey, params.tagValue, {   // (3) 全リソースにタグ付け
    excludeResourceTypes: ['AWS::OpenSearchServerless::Collection'],
  });
}
createStacks(app, params);                 // (4) スタック群を組み立てる（第4章）
```

流れは「App 生成 → 設定読み込み → スタック生成」だけです。実体は `lib/create-stacks.ts` と `lib/*-stack.ts` にあります。

`cdk.Tags.of(app).add(...)` は **Aspects** という仕組みで、「ツリー配下の全リソースに一括で何かを適用する」機能です（第5章で再登場します）。

## 1.4 CDK CLI の主要コマンド

GenU ではルートの `package.json` にラッパーコマンドが用意されています。

| コマンド | 素の CDK | 役割 |
|---------|----------|------|
| `npm run cdk:deploy` | `cdk deploy --all` | 全スタックをデプロイ |
| `npm run cdk:diff` | `cdk diff --all` | **今のコードとデプロイ済み環境の差分表示**。デプロイ前に必ず実行する習慣を |
| `npm run cdk:destroy` | `cdk destroy --all` | 全スタック削除 |
| （直接実行） | `cdk synth` | テンプレート生成のみ（`cdk.out/` に出力）。コードの検証に便利 |
| （初回のみ） | `cdk bootstrap` | CDK 用の作業リソース（S3 等）をアカウント/リージョンに準備 |

### bootstrap とは

CDK はデプロイ時に「Lambda のコード zip」「Docker イメージ」などの**アセット**を一時置き場（S3）にアップロードしてから CloudFormation に渡します。この置き場を作るのが `cdk bootstrap` で、**アカウント × リージョンごとに初回1回だけ**必要です。GenU は us-east-1（WAF 用）と モデルリージョンも使うため、複数リージョンで bootstrap が必要になることがあります。

## 1.5 デプロイの実行フロー（全体像）

`npm run cdk:deploy` の裏で起きること:

```
1. bin/generative-ai-use-cases.ts を実行（= TypeScript として評価）
   → App / Stack / Construct のツリーがメモリ上に組み上がる
2. synth: ツリーから CloudFormation テンプレートを生成（cdk.out/）
   このとき GenU では packages/web のフロントエンドビルドも仕込まれる（第5章）
3. アセットを S3 にアップロード（Lambda コードは esbuild でバンドルされる）
4. CloudFormation がテンプレートを適用（作成・更新・削除）
5. Outputs（WebUrl, ApiEndpoint など）が表示される
```

**エラーが出たときの切り分け**: 手順1〜2で失敗（型エラー、synth 時の throw）なら TypeScript の問題でローカルで完結。手順4で失敗ならリソース作成の問題（権限、上限、名前重複など）で、CloudFormation コンソールのイベントタブに原因が出ます。

---

## 練習問題 1

**問1-1.** CDK コードを変更して `npm run cdk:deploy` したとき、AWS に反映されるまでに CDK コードはどんな形に変換されますか。また、その中間生成物はどのディレクトリに出力されますか。

**問1-2.** App / Stack / Construct の3つを「デプロイの単位」という観点で説明してください。「あるリソースだけ削除したい」とき、どの単位で考える必要がありますか。

**問1-3.** `cdk deploy` の前に `cdk diff` を実行すべき理由を、CloudFormation の「更新」の挙動を踏まえて説明してください（ヒント: リソースの設定変更には「その場で更新」と「作り直し（replacement）」の2種類がある）。

**問1-4.** GenU の `packages/cdk/cdk.json` を開き、(1) `app` に指定されたエントリーポイント、(2) `context` に定義されている設定項目を5つ、確認してください。`ragEnabled` の初期値は何ですか。

[解答例](./answers.md#第1章)
