# 第9章 react-router-dom によるルーティング

## 9.1 ルーティングとは

GenU は SPA（Single Page Application）です。ブラウザは最初に1つの HTML を読むだけで、以降の画面遷移（`/chat` → `/summarize` など）は **JavaScript が URL に応じてコンポーネントを差し替える**ことで実現します。これを担うのが react-router-dom（v6）です。

## 9.2 基本: ルート定義

```tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const router = createBrowserRouter([
  { path: '/', element: <HomePage /> },
  { path: '/about', element: <AboutPage /> },
  { path: '/users/:userId', element: <UserPage /> },  // :userId は URL パラメータ
  { path: '*', element: <NotFound /> },               // どれにも一致しないとき
]);

// アプリのルートで
<RouterProvider router={router} />
```

## 9.3 画面遷移

```tsx
import { Link, useNavigate } from 'react-router-dom';

// (1) リンク: <a> の代わりに Link（ページ全体を再読み込みしない）
<Link to="/chat">チャットへ</Link>

// (2) プログラムから遷移: ボタンクリック後や処理完了後に
const navigate = useNavigate();
navigate('/chat');
navigate(-1);  // ブラウザバック相当
```

## 9.4 URL パラメータの取得

```tsx
import { useParams } from 'react-router-dom';

// path: '/chat/:chatId' に対して /chat/abc123 でアクセスされたとき
const ChatPage = () => {
  const { chatId } = useParams();  // 'abc123'
  // chatId を使ってデータ取得...
};
```

GenU の `ChatPage` はまさにこの形で、`/chat`（新規チャット）と `/chat/:chatId`（既存チャットの続き）の両方が同じ `ChatPage` コンポーネントに割り当てられています。`chatId` の有無で新規/既存を判別します。

## 9.5 レイアウトルートと Outlet

「全ページ共通のヘッダー・サイドバーの内側に、ページごとの中身を差し込む」には、**親ルート + `children` + `<Outlet />`** を使います。

```tsx
const router = createBrowserRouter([
  {
    path: '/',
    element: <App />,          // 共通レイアウト
    children: [                // 子ルートたち
      { path: '/', element: <LandingPage /> },
      { path: '/chat', element: <ChatPage /> },
    ],
  },
]);

// App.tsx（共通レイアウト側）
import { Outlet } from 'react-router-dom';

const App = () => {
  return (
    <div>
      <Header />
      <Drawer />       {/* サイドバー */}
      <main>
        <Outlet />     {/* ← ここに子ルートの element が描画される */}
      </main>
    </div>
  );
};
```

## 9.6 GenU の実コードを読む: main.tsx のルート定義

GenU の `packages/web/src/main.tsx` は、この章の内容がすべて詰まった実例です（抜粋・簡略化）。

```tsx
// (1) ページごとのルートを配列で定義
const routes: RouteObject[] = [
  { path: '/', element: <LandingPage /> },
  { path: '/setting', element: <Setting /> },
  { path: '/chat', element: <ChatPage /> },
  { path: '/chat/:chatId', element: <ChatPage /> },   // URL パラメータ

  // (2) 環境変数による機能の ON/OFF: 無効な機能はルートごと消す
  enabled('summarize')
    ? { path: '/summarize', element: <SummarizePage /> }
    : null,

  { path: '*', element: <NotFound /> },
].flatMap((r) => (r !== null ? [r] : []));  // null を除去

// (3) 共通レイアウト App の下に children としてぶら下げる
//     さらに認証コンポーネントで包む（ログインしないと中に入れない）
const router = createBrowserRouter([
  {
    path: '/',
    element: samlAuthEnabled ? (
      <AuthWithSAML><App /></AuthWithSAML>
    ) : (
      <AuthWithUserpool><App /></AuthWithUserpool>
    ),
    children: routes,
  },
]);
```

読み取れるポイント:

1. **`App` がレイアウトルート** — `App.tsx` の中に `<Outlet />` があり（375行目付近）、各ページはヘッダー・サイドバー付きの枠の中に描画される
2. **認証でラップ** — `AuthWithUserpool`（Cognito ログイン）が `App` を包んでいるため、未ログインでは全ページにアクセスできない。children を使った「ゲート」パターン
3. **機能フラグでルートを動的に組み立て** — デプロイ設定（`VITE_APP_XXX_ENABLED` 環境変数や `enabled()`）で無効化された機能は、ルート定義自体から除外される。「メニューに出ないだけでなく URL 直打ちでも入れない」ことがこれで保証される

「新しいページを追加したい」ときにやることは: ① `pages/` にコンポーネントを作る → ② `main.tsx` の `routes` に追加 → ③ サイドバーのメニュー（`App.tsx` 内の items）に追加、の3点セットです。

---

## 練習問題 9

**問9-1.** `<a href="/chat">` ではなく `<Link to="/chat">` を使うべき理由を説明してください。

**問9-2.** 練習用プロジェクトに react-router-dom を入れ（`npm install react-router-dom`）、次の構成を作ってください。
- 共通レイアウト（ナビゲーションリンク付き）+ `<Outlet />`
- `/` → Home、`/items/:itemId` → itemId を表示する詳細ページ
- 一致しない URL → 「Not Found」

**問9-3.** GenU で `/summarize` にアクセスしたのに NotFound が表示されるとき、コード上どこを確認すべきですか（この章で読んだ main.tsx の仕組みから答えてください）。

**問9-4.** GenU の `packages/web/src/pages/ChatPage.tsx` で `useParams` がどう使われているか確認し、`chatId` が取れたとき/取れないときで挙動がどう変わるかを大まかに説明してください。

[解答例](./answers.md#第9章)
