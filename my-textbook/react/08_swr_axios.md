# 第8章 SWR + axios によるデータ取得

## 8.1 GenU の API 通信の全体像

GenU のフロントエンドは、API Gateway 経由でバックエンド（Lambda）と通信します。その基盤が3層になっています。

```
useChatApi.ts など（機能別 API フック）
  └── useHttp.ts（共通基盤: axios + SWR + 認証ヘッダー）
        ├── axios  … HTTP リクエストを送るライブラリ
        └── SWR    … GET したデータのキャッシュ・再取得を管理するライブラリ
```

## 8.2 axios — HTTP クライアント

axios は `fetch` の高機能版です。GenU の `packages/web/src/hooks/useHttp.ts` 冒頭で共通インスタンスを作っています（実物・コメント追記）。

```ts
import axios from 'axios';

// ベース URL を設定した axios インスタンス
const api = axios.create({
  baseURL: import.meta.env.VITE_APP_API_ENDPOINT,  // 環境変数から API の URL を取得
});

// リクエスト前処理: 全リクエストに Cognito の認証トークンを自動付与
api.interceptors.request.use(async (config) => {
  const token = (await fetchAuthSession()).tokens?.idToken?.toString();
  if (token) {
    config.headers['Authorization'] = token;
  }
  config.headers['Content-Type'] = 'application/json';
  return config;
});
```

- `import.meta.env.VITE_APP_XXX` — Vite の環境変数の読み方
- `interceptors` — 全リクエスト/レスポンスに共通処理を差し込む仕組み。**GenU では認証トークンの付与をここで一元化**しているため、各 API 呼び出しコードに認証処理が出てきません

基本的な使い方:

```ts
const res = await api.get('/chats');          // GET
const res = await api.post('/chats', body);   // POST（第2引数がリクエストボディ）
await api.delete(`/chats/${chatId}`);         // DELETE
// res.data にレスポンスボディが入る
```

## 8.3 SWR — GET データの管理

「画面表示用のデータ取得」を useEffect + axios で手書きすると、ローディング状態・エラー状態・キャッシュ・再取得をすべて自前管理することになります。SWR はこれを1行にします。

```tsx
import useSWR from 'swr';

const fetcher = (url: string) => api.get(url).then((res) => res.data);

const ChatList = () => {
  const { data, isLoading, error, mutate } = useSWR('/chats', fetcher);

  if (isLoading) return <p>読み込み中...</p>;
  if (error) return <p>エラーが発生しました</p>;
  return <ul>{data.chats.map((c) => <li key={c.chatId}>{c.title}</li>)}</ul>;
};
```

SWR の重要な性質:

- **キャッシュ**: 同じ URL への useSWR は1回しか通信せず、結果を共有する
- **自動再検証**: タブを再フォーカスしたときなどに裏で再取得して最新化する（Stale-While-Revalidate = 古いデータを見せつつ裏で更新、が名前の由来）
- **`mutate`**: 「データが変わったから取り直して」と明示的に伝える関数。**POST や DELETE でデータを変更した後に呼ぶ**のが定番パターン

## 8.4 GenU の実コードを読む: useHttp.ts

GenU は useSWR / axios を直接使わず、`useHttp` というフックにまとめています（抜粋・簡略化）。

```ts
const useHttp = () => {
  return {
    api,

    // GET: SWR でキャッシュ管理付き取得
    get: <Data = any, Error = any>(url: string | null, config?: SWRConfiguration) => {
      return useSWR<Data, Error>(url, fetcher, config);
    },

    // POST/PUT/DELETE: axios を直接使う（データ変更系はキャッシュ不要）
    post: <RES = any, DATA = any>(url: string, data: DATA) => {
      return api.post<RES>(url, data);
    },
    put: ...,
    delete: ...,
  };
};
```

覚えるべき使い分け:

| メソッド | 実装 | 用途 |
|---------|------|------|
| `get` | SWR | 画面に表示するデータの取得（キャッシュ・自動再取得付き） |
| `post` / `put` / `delete` | axios 直 | データの作成・更新・削除 |

`get(url)` の `url` に **`null` を渡すと通信しない**のも SWR の重要な機能です。「chatId が確定するまで取得しない」のような条件付き取得に使います。

```ts
// chatId が undefined の間はリクエストが飛ばない
const { data } = http.get(chatId ? `/chats/${chatId}` : null);
```

## 8.5 GenU の実コードを読む: useChatApi.ts の層

機能別 API フックは `useHttp` を使って各エンドポイントを関数化したものです（概念形）。

```ts
const useChatApi = () => {
  const http = useHttp();
  return {
    listChats: () => http.get('/chats'),
    createChat: () => http.post('/chats', {}),
    deleteChat: (chatId: string) => http.delete(`/chats/${chatId}`),
    ...
  };
};
```

ページ → `useChat`（状態管理）→ `useChatApi`（エンドポイント定義）→ `useHttp`（通信基盤）という第6章で見た層構造がここで完成します。

---

## 練習問題 8

**問8-1.** SWR を使わず useEffect + fetch でデータ取得を実装した場合、自分で管理しなければならないものを3つ挙げてください。

**問8-2.** 無料 API の JSONPlaceholder（`https://jsonplaceholder.typicode.com/users`）からユーザー一覧を取得して名前を表示するコンポーネントを、useSWR を使って作ってください（練習用プロジェクトに `npm install swr axios` してから）。ローディングとエラーの表示も実装すること。

**問8-3.** 「チャットを削除した直後、一覧画面に削除済みのチャットがまだ表示されている」というバグが起きたとします。SWR の観点から、原因として何が考えられ、どう直しますか。

**問8-4.** GenU の `packages/web/src/hooks/useChatApi.ts` を開き、(1) `useHttp` の `get` を使っている関数、(2) `post` を使っている関数、をそれぞれ1つずつ見つけて、何の API か説明してください。

[解答例](./answers.md#第8章)
