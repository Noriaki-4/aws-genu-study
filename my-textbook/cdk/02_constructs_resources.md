# 第2章 Construct と AWS リソース定義

## 2.1 リソース定義の基本形

CDK ではリソースを「クラスのインスタンス化」で定義します。**すべてのリソースは同じ3引数パターン**です。

```ts
new リソースクラス(scope, id, props);
//               ^親      ^識別子  ^設定
```

```ts
import { Bucket } from 'aws-cdk-lib/aws-s3';

const bucket = new Bucket(this, 'FileBucket', {
  encryption: BucketEncryption.S3_MANAGED,
  versioned: true,
});
```

- **scope（第1引数）**: 親 Construct。ほぼ常に `this`（今いる Stack や Construct の中に作る、という意味）
- **id（第2引数）**: 親の中で一意な識別子。**CloudFormation 上のリソース名（論理ID）の元になる**
- **props（第3引数）**: リソースの設定。省略可能なものも多い

### id を変えてはいけない

CloudFormation は論理IDでリソースを同一視します。**既にデプロイ済みのリソースの id を変更すると「旧リソースを削除して新リソースを作成」**と解釈されます。DynamoDB テーブルなら**データが消えます**。GenU のコードで既存の id を安易にリネームしてはいけない理由がこれです。

## 2.2 L1 / L2 / L3 Construct

CDK のリソースクラスには抽象度のレベルがあります。

| レベル | 命名 | 特徴 | GenU での例 |
|--------|------|------|------------|
| L1 | `Cfn` プレフィックス | CloudFormation と1対1。全項目を手動指定 | `CfnDistribution`, `CfnWebACLAssociation` |
| L2 | 通常名 | 適切なデフォルト値・便利メソッド付き。**基本はこれを使う** | `Bucket`, `Table`, `UserPool`, `RestApi` |
| L3 (パターン) | ライブラリによる | 複数リソースの定番構成をまとめたもの | `CloudFrontToS3`（AWS Solutions Constructs）, `NodejsBuild` |

L2 に無い細かい設定をしたいときだけ L1 に降りる、というのが実践パターンです。GenU の `web.ts` では L3 の `CloudFrontToS3` を使いつつ、L1 の `CfnDistribution` を触って細部を調整しています。

## 2.3 GenU の実コードを読む (1): Database

`packages/cdk/lib/construct/database.ts` は最小の自作 Construct の好例です（実物・コメント追記）。

```ts
import { Construct } from 'constructs';
import * as ddb from 'aws-cdk-lib/aws-dynamodb';

// Construct を継承した「自作の部品」
export class Database extends Construct {
  // 外部（他の Construct）に公開するリソース
  public readonly table: ddb.Table;
  public readonly statsTable: ddb.Table;
  public readonly feedbackIndexName: string;

  constructor(scope: Construct, id: string) {
    super(scope, id);   // 親クラスの初期化（お決まり）

    // チャット履歴テーブル
    const feedbackIndexName = 'FeedbackIndex';
    const table = new ddb.Table(this, 'Table', {
      partitionKey: { name: 'id', type: ddb.AttributeType.STRING },
      sortKey: { name: 'createdDate', type: ddb.AttributeType.STRING },
      billingMode: ddb.BillingMode.PAY_PER_REQUEST,  // 従量課金
    });

    // フィードバック検索用のインデックス
    table.addGlobalSecondaryIndex({
      indexName: feedbackIndexName,
      partitionKey: { name: 'feedback', type: ddb.AttributeType.STRING },
    });

    // トークン使用量の統計テーブル
    const statsTable = new ddb.Table(this, 'StatsTable', {
      partitionKey: { name: 'id', type: ddb.AttributeType.STRING },
      sortKey: { name: 'userId', type: ddb.AttributeType.STRING },
      billingMode: ddb.BillingMode.PAY_PER_REQUEST,
    });

    // 公開プロパティに代入
    this.table = table;
    this.statsTable = statsTable;
    this.feedbackIndexName = feedbackIndexName;
  }
}
```

自作 Construct のパターン:

1. `Construct` を継承し、コンストラクタは `(scope, id, props?)` を受けて `super(scope, id)` を呼ぶ
2. 中で `new リソース(this, ...)` — `this` を親にすることでこの Construct の配下になる
3. **他の部品から参照させたいリソースは `public readonly` で公開**する

使う側（`generative-ai-use-cases-stack.ts`）:

```ts
const database = new Database(this, 'Database');
// ...
const api = new Api(this, 'API', {
  table: database.table,          // 公開プロパティを他の Construct に渡す
  statsTable: database.statsTable,
  ...
});
```

この「**Construct A が公開したリソースを、Stack が Construct B の props に配線する**」のが GenU 全体の基本構造です。Stack 本体（`generative-ai-use-cases-stack.ts`）は配線係で、実装は `lib/construct/` の各部品にあります。

## 2.4 GenU の実コードを読む (2): Auth

`packages/cdk/lib/construct/auth.ts` は「props で挙動を切り替える Construct」の好例です（抜粋）。

```ts
export interface AuthProps {
  readonly selfSignUpEnabled: boolean;
  readonly allowedIpV4AddressRanges?: string[] | null;
  readonly allowedSignUpEmailDomains?: string[] | null;
  readonly samlAuthEnabled: boolean;
}

export class Auth extends Construct {
  readonly userPool: UserPool;
  readonly client: UserPoolClient;
  readonly idPool: IdentityPool;

  constructor(scope: Construct, id: string, props: AuthProps) {
    super(scope, id);

    const userPool = new UserPool(this, 'UserPool', {
      // SAML 認証時はセルフサインアップを強制無効化（設定値の組み合わせをコードで担保）
      selfSignUpEnabled: props.samlAuthEnabled ? false : props.selfSignUpEnabled,
      signInAliases: { username: false, email: true },
      passwordPolicy: {
        requireUppercase: true,
        requireSymbols: true,
        requireDigits: true,
        minLength: 8,
      },
    });

    const client = userPool.addClient('client', {
      idTokenValidity: Duration.days(1),
    });
    // ...
    // IP 制限が設定されているときだけポリシーを追加
    if (props.allowedIpV4AddressRanges || props.allowedIpV6AddressRanges) {
      idPool.authenticatedRole.attachInlinePolicy(...);
    }
  }
}
```

読み取れるパターン:

- **props インターフェースを定義**し、デプロイ設定（cdk.json 由来）を受け取る
- **`if` や三項演算子で「条件付きリソース作成」**。CloudFormation の YAML では書きづらい部分が、CDK では普通のコードで書ける
- `userPool.addClient(...)` のような **add 系メソッド**: L2 Construct はリソース同士の典型的な関連付けをメソッドとして提供している
- `Duration.days(1)` — CDK は「1日 = 86400」のような生の数値の代わりに型付きの `Duration` / `Size` クラスを使う

## 2.5 Construct ツリーと論理ID

リソースの CloudFormation 論理IDは、**ツリー上のパス（親の id + 自分の id + ハッシュ）**から生成されます。

```
GenerativeAiUseCasesStack
└── Database (id: 'Database')
    └── Table (id: 'Table')       → 論理ID: DatabaseTableXXXXXXXX
```

だから同じ `Table` という id でも `Database` の下と `UseCaseBuilder` の下では衝突しません。「Construct = 名前空間」でもあるわけです。

---

## 練習問題 2

**問2-1.** `new Bucket(this, 'MyBucket', {...})` の3つの引数それぞれの役割を説明してください。

**問2-2.** デプロイ済みの CDK アプリで、DynamoDB テーブルの id (`'Table'`) を `'MainTable'` にリネームして deploy すると何が起きますか。なぜ危険ですか。

**問2-3.** 「S3 バケット + その中身を空にしてから削除する設定」のような複数リソースの定番構成を再利用したい場合、L1 / L2 / 自作 Construct のどれで実現するのが適切ですか。自作 Construct の骨格コード（constructor と public readonly まで）を書いてください。

**問2-4.** GenU の `packages/cdk/lib/construct/transcribe.ts` を開き、(1) どんな AWS リソースを作っているか、(2) `public readonly` で何を公開しているか、(3) それは Stack のどこで誰に配線されているか（`generative-ai-use-cases-stack.ts` を grep）を調べてください。

**問2-5.** `auth.ts` で SAML 認証有効時にセルフサインアップを強制的に false にしているのはなぜだと思いますか。設定の組み合わせをコードで担保するメリットを説明してください。

[解答例](./answers.md#第2章)
