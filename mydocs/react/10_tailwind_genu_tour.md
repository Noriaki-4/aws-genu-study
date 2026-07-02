# 第10章 Tailwind CSS と GenU ソースの歩き方

## 10.1 Tailwind CSS とは

GenU は CSS ファイルをほとんど書かず、**ユーティリティクラスを className に並べる** Tailwind CSS でスタイリングしています。

```tsx
// 従来: CSS ファイルにクラスを定義して参照
<button className="send-button">送信</button>

// Tailwind: 小さなユーティリティクラスを組み合わせる
<button className="flex items-center rounded-lg bg-blue-500 p-1 px-3 text-white hover:brightness-75">
  送信
</button>
```

### 頻出クラス早見表（GenU を読むのに必要な分）

| 分類 | クラス例 | 意味 |
|------|---------|------|
| レイアウト | `flex`, `grid`, `items-center`, `justify-center`, `gap-2` | Flexbox/Grid 配置 |
| 余白 | `p-4`（padding 1rem）, `px-3`（左右）, `m-2`, `mr-2`（右 margin） | 数字 ×0.25rem |
| サイズ | `w-full`, `h-screen`, `max-w-md` | 幅・高さ |
| 文字 | `text-sm`, `text-xl`, `font-bold`, `text-white` | サイズ・太さ・色 |
| 背景・枠 | `bg-white`, `border`, `rounded-lg`, `shadow` | 背景色・境界線・角丸・影 |
| 状態 | `hover:brightness-75`, `disabled:opacity-30` | ホバー時など |
| レスポンシブ | `lg:visible`, `md:w-1/2` | 画面幅による切り替え |
| アニメーション | `animate-spin`, `transition` | 回転（スピナー）など |

### GenU 独自の色

`Button.tsx` に出てきた `bg-aws-smile` や `text-aws-font-color` は Tailwind 標準ではなく、GenU が `packages/web/tailwind.config.js` の `theme.extend.colors` で定義した**カスタムカラー**です。色を変えたいときはこのファイルを見ます。

### 動的なクラスの組み立て

GenU 頻出パターン。テンプレートリテラルと三項演算子でクラスを切り替えます。

```tsx
className={`${props.className ?? ''} flex rounded-lg ${
  props.disabled ? 'opacity-30' : 'hover:brightness-75'
}`}
```

`props.className ?? ''` を先頭に置くのは「親が追加のクラスを注入できるようにする」ためで、`BaseProps` の `className` とセットの定番イディオムです。

## 10.2 GenU フロントエンドのディレクトリ構成

```
packages/web/src/
├── main.tsx          # エントリーポイント・全ルート定義（第9章）
├── App.tsx           # 共通レイアウト（ヘッダー・サイドバー・<Outlet />）
├── pages/            # ページ（ルート1つに対して1ファイル）
│   ├── ChatPage.tsx
│   ├── SummarizePage.tsx
│   └── ...
├── components/       # 再利用可能な UI 部品
│   ├── Button.tsx, Card.tsx, Textarea.tsx ...
│   └── ChatMessage.tsx, ChatList.tsx ...
├── hooks/            # ロジック層（第6〜8章）
│   ├── useChat.ts        # 機能フック（zustand で状態管理）
│   ├── useChatApi.ts     # API フック
│   ├── useHttp.ts        # 通信基盤（axios + SWR）
│   └── ...
├── prompts/          # LLM に渡すプロンプトのテンプレート
├── utils/            # 純粋関数のユーティリティ
├── i18n/             # 多言語対応（react-i18next）の設定
└── @types/           # 共通の型定義（BaseProps など）
```

対応するバックエンドやインフラは:

- `packages/cdk/` — AWS インフラ定義（AWS CDK）。API Gateway、Lambda、Cognito など
- `packages/cdk/lambda/` — バックエンドの Lambda 関数の実装
- `packages/types/` — フロント・バック共通の型定義

## 10.3 1機能を端から端まで追う: チャット送信の旅

「チャット画面でメッセージを送信すると何が起きるか」を追ってみます。GenU 読解の総仕上げです。

```
1. 入力・送信（UI 層）
   pages/ChatPage.tsx
   └── 入力欄 InputChatContent に文字を入力（useState / 制御されたコンポーネント）
   └── 送信ボタン onSend → useChat の postChat(content) を呼ぶ

2. 状態更新（ロジック層）
   hooks/useChat.ts（zustand ストア）
   └── pushMessage でユーザーのメッセージを messages 配列に追加（イミュータブル更新）
       → ChatPage が再レンダリングされ、吹き出しが即座に画面に出る
   └── loading を true に → 送信ボタンがスピナー表示に変わる

3. API 呼び出し（通信層）
   hooks/useChatApi.ts → hooks/useHttp.ts
   └── axios が API Gateway へ POST（interceptor が認証トークンを自動付与）
   └── LLM の応答をストリーミングで受信しながら、届いた分だけ
       アシスタントのメッセージを更新していく → 画面に文字が流れるように表示される

4. 完了
   └── loading を false に戻す
   └── チャット履歴一覧（SWR キャッシュ）を mutate で更新
```

この流れには本教科書の全章が登場しています:

- 制御されたコンポーネント（第4章）
- カスタムフックの層構造（第6章）
- zustand によるグローバル状態とイミュータブル更新（第7章）
- axios / SWR / mutate（第8章）
- ルーティングと URL パラメータ（第9章）

## 10.4 目的別・コードの探し方

| やりたいこと | 見るファイル |
|--------------|-------------|
| 画面の文言・レイアウトを変えたい | `pages/○○Page.tsx` → 使われている `components/` |
| ボタンなど部品の見た目を変えたい | `components/` の該当ファイル + `tailwind.config.js` |
| チャットの挙動を変えたい | `hooks/useChat.ts` |
| LLM への指示（プロンプト）を変えたい | `prompts/claude.ts` |
| API の呼び先を変えたい | `hooks/use○○Api.ts` |
| 新しいページを足したい | `pages/` に作成 → `main.tsx` の routes → `App.tsx` のメニュー |
| 機能の ON/OFF を切り替えたい | `packages/cdk/cdk.json` の context（デプロイ設定） |
| 表示文言の日本語訳を変えたい | `public/locales/` の翻訳 YAML（i18next） |

探すときは「文言で grep」が最速です。ただし画面の文言は i18n 化されているため、日本語文言は `public/locales/` 側でヒットし、そこで得た**翻訳キー**でソースを grep すると該当コンポーネントが見つかります。

---

## 練習問題 10

**問10-1.** 次の Tailwind クラス列が当てられた要素の見た目を説明してください。

```
flex items-center justify-center rounded-lg bg-white p-4 shadow hover:brightness-90
```

**問10-2.** GenU のチャット画面の送信ボタンの色を変えたいとします。調査の手順を、この章の「目的別・コードの探し方」に沿って書いてください（実際に変更しなくてOK。どのファイルに行き着くかまで）。

**問10-3.** GenU の `packages/web/src/App.tsx` を開き、(1) `<Outlet />` の場所、(2) サイドバー（Drawer）に渡しているメニュー項目のデータ、を見つけてください。新しいページをメニューに足すにはどの配列に追加すればよいですか。

**問10-4.**（総合演習）練習用プロジェクトで、この教科書で学んだ要素をすべて使ったミニアプリ「メモ帳」を作ってください。
- ルート: `/`（メモ一覧）と `/memo/:memoId`（詳細）
- メモの追加・削除（zustand でグローバル管理、イミュータブル更新）
- 一覧 → 詳細への遷移は `Link`、追加後の遷移は `useNavigate`
- 見た目は Tailwind（`npm install -D tailwindcss @tailwindcss/vite` などで導入。難しければ素の CSS でも可）

[解答例](./answers.md#第10章)
