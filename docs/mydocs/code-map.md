# GenU ソースコード・コードマップ

このリポジトリ（Generative AI Use Cases / GenU）のソースコード構成を、**Mermaid図**で視覚的にまとめたものです。
インフラ構成図（[arch.drawio.png](../assets/images/arch.drawio.png)）が「AWSリソースの繋がり」を表すのに対し、こちらは **「コード（パッケージ・モジュール）の繋がり」** を表します。

> Mermaid図はGitHub・VSCode（Markdown Preview Mermaid Support拡張）・mkdocs(mermaidプラグイン)でそのまま描画されます。

---

## 1. モノレポ全体構成（npm workspaces）

ルートの `package.json` は `packages/*` を workspaces として束ねています。
`common` と `types` が共有基盤で、`web`（フロント）と `cdk`（インフラ）の両方から参照されます。

```mermaid
graph TD
    root["📦 root<br/>(npm workspaces)"]

    subgraph packages["packages/"]
        web["🖥️ web<br/>React + Vite フロントエンド"]
        cdk["☁️ cdk<br/>AWS CDK インフラ + Lambda"]
        common["🔧 common<br/>フロント/バック共通ロジック"]
        types["📐 types<br/>共有 TypeScript 型定義 (.d.ts)"]
        eslint["🧹 eslint-plugin-i18nhelper<br/>i18n用 独自Lintルール"]
    end

    root --> web
    root --> cdk
    root --> common
    root --> types
    root --> eslint

    web -->|import 型/共通処理| common
    web -->|import 型| types
    cdk -->|import 型/共通処理| common
    cdk -->|import 型| types
    common -->|参照| types

    classDef shared fill:#fef3c7,stroke:#d97706,stroke-width:2px;
    classDef app fill:#dbeafe,stroke:#2563eb,stroke-width:2px;
    class common,types shared;
    class web,cdk app;
```

| パッケージ | 役割 | 主な技術 |
|-----------|------|---------|
| `packages/web` | ブラウザで動くUI。各ユースケースの画面 | React 18 / Vite / TailwindCSS / Zustand / SWR / react-router |
| `packages/cdk` | AWSインフラ定義とバックエンドLambda | AWS CDK / TypeScript / Lambda |
| `packages/common` | フロント・バックで共有するアプリロジック | TypeScript |
| `packages/types` | 共有する型定義のみ | TypeScript (`.d.ts`) |
| `packages/eslint-plugin-i18nhelper` | 多言語対応を強制する独自ESLintルール | ESLint Plugin |

---

## 2. リクエストの流れ（フロント → インフラ → LLM）

ユーザー操作が、どのコードを通ってAWS Bedrockまで届くかの流れです。
`web`側の **hooks** がAPIを呼び、`cdk`側の **Lambda** が実処理を担います。

```mermaid
graph LR
    subgraph browser["ブラウザ (packages/web)"]
        page["pages/*.tsx<br/>画面"]
        hookUI["hooks/useChat.ts など<br/>(状態管理 Zustand)"]
        hookApi["hooks/*Api.ts<br/>(useChatApi 等)"]
        http["hooks/useHttp.ts<br/>(axios + 認証トークン)"]
    end

    subgraph aws["AWS (packages/cdk で定義)"]
        apigw["API Gateway"]
        lambda["lambda/*.ts<br/>(predictStream 等)"]
        bedrock["Amazon Bedrock<br/>(LLM)"]
    end

    page --> hookUI --> hookApi --> http
    http -->|HTTPS + Cognito JWT| apigw
    apigw --> lambda --> bedrock
    bedrock -.->|ストリーミング応答| lambda -.-> http

    classDef web fill:#dbeafe,stroke:#2563eb;
    classDef infra fill:#dcfce7,stroke:#16a34a;
    class page,hookUI,hookApi,http web;
    class apigw,lambda,bedrock infra;
```

**ポイント**
- 画面(`pages`)は状態管理hook(`useChat`など)を呼ぶだけで、通信の詳細は知らない
- `*Api.ts` hook がエンドポイントを、`useHttp.ts` が認証ヘッダ付与を担当（責務分離）
- リクエスト/レスポンスの型は `packages/types` の `.d.ts` をフロント・バック双方が共有 → **型の齟齬が起きない**

---

## 3. フロントエンド内部構造（packages/web/src）

```mermaid
graph TD
    main["main.tsx<br/>エントリポイント"]
    app["App.tsx<br/>ルーティング定義"]

    main --> app

    app --> pages
    subgraph pages["pages/ (26画面)"]
        chat["ChatPage"]
        rag["RagPage / RagKnowledgeBasePage"]
        img["GenerateImagePage"]
        agent["AgentChatPage / ResearchAgentPage"]
        etc["… Summarize / Translate / Transcribe 他"]
    end

    pages --> components
    pages --> hooks

    subgraph components["components/ (67+)"]
        ui["Button/Card/... 汎用UI"]
        feature["Mermaid / ECharts / Writer<br/>機能別UI"]
    end

    subgraph hooks["hooks/ (63)"]
        state["状態管理系<br/>useChat / useDrawer (Zustand)"]
        api["API系<br/>useChatApi / useFileApi …"]
        util["ユーティリティ系<br/>useHttp / useLocalStorage …"]
    end

    hooks --> shared
    subgraph shared["共有"]
        types2["@types/generative-ai-use-cases"]
        common2["@generative-ai-use-cases/common"]
    end

    subgraph others["その他"]
        i18n["i18n/ 多言語辞書"]
        prompts["prompts/ プロンプト定義"]
        utils["utils/ 汎用関数"]
    end
    pages -.-> i18n
    hooks -.-> prompts

    classDef entry fill:#fce7f3,stroke:#db2777;
    class main,app entry;
```

**設計の勘所**
- `pages` = ユースケース1つ = 1ファイル（画面追加は基本ここ＋ルート追加）
- `hooks` は **状態管理系 / API系 / ユーティリティ系** の3系統に整理されている
- ロジックはhooksに寄せ、`components`は表示に専念（テスト・再利用しやすい）

---

## 4. CDK（インフラ）内部構造（packages/cdk）

```mermaid
graph TD
    bin["bin/generative-ai-use-cases.ts<br/>CDK アプリ起点"]
    createStacks["lib/create-stacks.ts<br/>スタック生成の司令塔"]
    stackInput["lib/stack-input.ts<br/>パラメータ検証(zod)"]

    bin --> createStacks
    createStacks --> stackInput

    createStacks --> stacks
    subgraph stacks["lib/*-stack.ts (スタック)"]
        main["generative-ai-use-cases-stack<br/>(メイン)"]
        rag["rag-knowledge-base-stack"]
        agent["agent-stack / agent-core-stack"]
        dash["dashboard-stack"]
        waf["cloud-front-waf-stack"]
        closed["closed-network-stack"]
    end

    main --> constructs
    subgraph constructs["lib/construct/ (部品)"]
        apiC["api.ts"]
        authC["auth.ts (Cognito)"]
        webC["web.ts (CloudFront+S3)"]
        dbC["database.ts (DynamoDB)"]
        ragC["rag.ts (Kendra)"]
        agentC["agent.ts"]
    end

    apiC --> lambdas
    subgraph lambdas["lambda/ (48関数)"]
        predict["predictStream.ts<br/>LLM呼び出し(ストリーム)"]
        crud["createChat / listChats …<br/>チャット履歴CRUD"]
        file["getFileUploadSignedUrl …<br/>ファイル操作"]
        ragL["retrieveKendra / retrieveKnowledgeBase"]
        repo["repository.ts<br/>DynamoDBアクセス層"]
    end
    crud --> repo

    classDef entry fill:#fce7f3,stroke:#db2777;
    classDef construct fill:#e0e7ff,stroke:#4f46e5;
    class bin,createStacks entry;
    class apiC,authC,webC,dbC,ragC,agentC construct;
```

**階層の考え方（上ほど抽象）**
1. `bin/` … CDKアプリの起点
2. `lib/*-stack.ts` … デプロイ単位（**Stack**）。機能ごとに分割
3. `lib/construct/` … 再利用可能な部品（**Construct**）。API・認証・DBなど
4. `lambda/` … 実際のバックエンド処理。`repository.ts` がデータアクセスを集約

---

## 5. ディレクトリ早見表

```text
aws-genu-study/
├── packages/
│   ├── web/                    # フロントエンド (React)
│   │   └── src/
│   │       ├── main.tsx        #   エントリポイント
│   │       ├── App.tsx         #   ルーティング
│   │       ├── pages/          #   画面 (ユースケース単位, 26枚)
│   │       ├── components/     #   UI部品 (67+)
│   │       ├── hooks/          #   ロジック (状態/API/ユーティリティ, 63)
│   │       ├── prompts/        #   プロンプト定義
│   │       ├── i18n/           #   多言語辞書
│   │       └── utils/          #   汎用関数
│   │
│   ├── cdk/                    # インフラ + バックエンド (AWS CDK)
│   │   ├── bin/                #   CDKアプリ起点
│   │   ├── lib/
│   │   │   ├── *-stack.ts      #   スタック (デプロイ単位)
│   │   │   ├── construct/      #   構成部品 (API/認証/DB…)
│   │   │   └── stack-input.ts  #   パラメータ検証
│   │   └── lambda/             #   Lambda関数 (48)
│   │
│   ├── common/                 # フロント/バック共通ロジック
│   ├── types/                  # 共有型定義 (.d.ts)
│   └── eslint-plugin-i18nhelper/  # 独自Lintルール
│
├── docs/                       # ドキュメント (mkdocs)
│   └── mydocs/                 #   ← このファイルの場所
├── my-textbook/                # 学習用教科書 (react / cdk)
└── browser-extension/          # ブラウザ拡張
```

---

## 関連資料
- インフラ構成図: [../assets/images/arch.drawio.png](../assets/images/arch.drawio.png)
- 閉域網構成図: [../assets/images/arch-closed-network.drawio.png](../assets/images/arch-closed-network.drawio.png)
- CDK学習教科書: [../../my-textbook/cdk/README.md](../../my-textbook/cdk/README.md)
- React学習教科書: [../../my-textbook/react/README.md](../../my-textbook/react/README.md)
