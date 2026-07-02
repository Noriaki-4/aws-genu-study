# AWS CDK 教科書 — GenU の CDK を読むための最短コース

GenU (generative-ai-use-cases) の `packages/cdk/` を読める・設定変更や軽微な改修ができるようになることをゴールにした教科書です。

TypeScript の文法は既知として扱います（必要なら [React 教科書 第2章](../react/02_jsx_typescript.md) を参照）。

## GenU のインフラ全体像

GenU をデプロイすると、CDK が以下の AWS リソースを自動構築します。

```
ユーザー
  │
  ▼
CloudFront + S3          ← フロントエンド配信（packages/web のビルド成果物）
  │                         + WAF（IP制限、us-east-1）
  ▼
Cognito                  ← 認証（UserPool / IdentityPool）
  │
  ▼
API Gateway (REST)       ← バックエンド API
  │
  ▼
Lambda (Node.js)         ← packages/cdk/lambda/ の各関数
  │
  ├── DynamoDB           ← チャット履歴・統計
  ├── S3                 ← ファイルアップロード
  └── Amazon Bedrock     ← LLM 呼び出し（別リージョン可）
       ├── Knowledge Base ← RAG（オプション）
       └── Agent          ← エージェント（オプション）
```

## 目次

1. [第1章 CDK の基本概念とライフサイクル](./01_cdk_basics.md) — App / Stack / Construct、synth と deploy、GenU のエントリーポイント
2. [第2章 Construct と AWS リソース定義](./02_constructs_resources.md) — L1/L2/L3、自作 Construct、GenU の `Database` / `Auth` を読む
3. [第3章 Lambda・API Gateway・IAM](./03_lambda_api.md) — NodejsFunction、REST API 定義、grant による権限付与、GenU の `Api` を読む
4. [第4章 マルチスタック構成とパラメータ管理](./04_stacks_parameters.md) — スタック分割の理由、クロスリージョン、cdk.json / parameter.ts / zod
5. [第5章 デプロイ運用と GenU CDK の歩き方](./05_deploy_operations.md) — deploy コマンド、環境切り替え、Web 配信の仕組み、目的別ファイルガイド

- [練習問題 解答例](./answers.md)

## 前提

- TypeScript が読める
- AWS の主要サービス名を聞いたことがある（Lambda、S3、DynamoDB 程度でOK。各章で都度説明します）

## 実際に動かす場合

```bash
# リポジトリルートで
npm ci
npm run cdk:deploy        # 全スタックをデプロイ
npm run cdk:diff          # 差分確認（デプロイ前に必ず）
npm run cdk:destroy       # 全削除
```

初回のみ `npx -w packages/cdk cdk bootstrap` が必要です（第1章参照）。デプロイの詳細は公式ガイド `docs/ja/DEPLOY_ON_AWS.md`、オプション一覧は `docs/ja/DEPLOY_OPTION.md` を参照してください。
