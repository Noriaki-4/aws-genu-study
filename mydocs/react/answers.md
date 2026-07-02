# 練習問題 解答例

必ず自分で解いてから読んでください。コード解答は一例であり、動けば書き方が違っても構いません。

## 第1章

**問1-1.** 従来は「DOM をどう書き換えるか（手順）」を命令的に書くのに対し、React は「状態がこうならば画面はこう」という対応（宣言）だけを書き、DOM の書き換えは React に任せる。

**問1-2.**（実施確認のみ）`src/App.tsx` 内の文字列を変更して保存 → ブラウザが自動更新されれば OK。これが Vite のホットリロード。

**問1-3.** `packages/web/index.html` の 19 行目に `<div id="root"></div>` がある。`main.tsx` はこの div を掴んで React アプリを描画している。

## 第2章

**問2-1.**

```tsx
<div className="card">
  <label htmlFor="username">名前</label>
  <input id="username" />
  <br />
</div>
```

変更点: `class`→`className`、`for`→`htmlFor`、`<input>` と `<br>` を自己終了タグに。

**問2-2.**

```tsx
<ul>
  {items.map((item) => (
    <li key={item.id}>{item.label}</li>
  ))}
</ul>
```

一意な `id` があるので index ではなく `item.id` を key にする。

**問2-3.**

```ts
type Props = {
  text: string;
  count?: number;
  onSelect: (value: string) => void;
};
```

**問2-4.** `Button.tsx` 内:
- `&&`: `{props.loading && <PiSpinnerGap className="mr-2 animate-spin" />}`（ローディング中のみスピナー表示）
- 三項演算子: `props.outlined ? 'text-aws-font-color ... bg-white' : 'bg-aws-smile border text-white'`（枠線スタイル/塗りつぶしスタイルの切り替え）。`props.disabled || props.loading ? 'opacity-30' : 'hover:brightness-75'` も該当。

## 第3章

**問3-1.**

```tsx
type Props = {
  label: string;
  color?: 'red' | 'blue';
};

const Badge = (props: Props) => {
  const color = props.color ?? 'blue';
  return (
    <span style={{ backgroundColor: color, color: 'white', padding: '2px 8px', borderRadius: 8 }}>
      {props.label}
    </span>
  );
};

export default Badge;
```

**問3-2.** Props に `children?: React.ReactNode;` を追加し、`{props.label}` の後ろに `{props.children}` を描画する。

**問3-3.** `Card.tsx` は `label` / `help` / `sub`（いずれも省略可）と `children` を受け取る。共通パターン:
1. `type Props = BaseProps & {...}` で共通 Props を合成している
2. `children: React.ReactNode` で中身を受け取る
3. `props.className ?? ''` を先頭に連結し、親からクラスを注入できるようにしている
（このうち2つ挙げられればOK）

**問3-4.** React のデータは親から子への一方通行（単方向データフロー）。子が Props を書き換えられると「どこで値が変わったか」を追えなくなり、状態変化 → 再レンダリングという React の仕組みも成立しなくなる。値を変えたいときは、親が state を変更して新しい Props を渡し直す。

## 第4章

**問4-1.**

```tsx
const Counter = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount((prev) => prev + 1)}>+1</button>
      <button onClick={() => setCount((prev) => prev - 1)}>-1</button>
      <button onClick={() => setCount(0)}>リセット</button>
    </div>
  );
};
```

**問4-2.**

```tsx
const TodoList = () => {
  const [text, setText] = useState('');
  const [todos, setTodos] = useState<string[]>([]);

  const addTodo = () => {
    if (text.trim() === '') return;      // 入力バリデーション
    setTodos([...todos, text]);          // イミュータブル更新
    setText('');                         // 入力欄をクリア
  };

  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={addTodo}>追加</button>
      <ul>
        {todos.map((todo, idx) => (
          <li key={idx}>{todo}</li>
        ))}
      </ul>
    </div>
  );
};
```

**問4-3.** `tags.push(tag)` が元の配列を破壊的に変更している。`setTags(tags)` に渡る配列は同じ参照のままなので、React は「変化なし」と判断し再レンダリングしない。修正: `setTags([...tags, tag]);` または `setTags((prev) => [...prev, tag]);`

**問4-4.** `packages/web/src/components/Textarea.tsx` の Props は `onChange: (value: string) => void`（21行目）。内部で `onChange={(e) => { props.onChange(e.target.value); ... }}` とイベントから値を取り出してから親に渡している。つまり**値を渡す形**。

## 第5章

**問5-1.**

```tsx
const ElapsedTimer = () => {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setSeconds((prev) => prev + 1);   // 関数型更新なので依存配列は [] でよい
    }, 1000);
    return () => clearInterval(timer);  // クリーンアップ
  }, []);

  return <p>経過: {seconds} 秒</p>;
};
```

**問5-2.**

```tsx
const TitleSetter = (props: { title: string }) => {
  useEffect(() => {
    document.title = props.title;
  }, [props.title]);   // title が変わるたび実行
  return null;         // 画面には何も描画しない
};
```

**問5-3.**
1. **クリーンアップがない**: `addEventListener` した scroll リスナーを解除していないため、アンマウント後もリスナーが残る（メモリリーク・多重登録）
2. **依存配列に `props.keyword` が入っていない**: keyword が変わっても再検索されず、初回の検索結果が表示され続ける

（補足: 検索とスクロール監視は関心が別なので、useEffect を2つに分けるのが望ましい）

**問5-4.** `ChatPage.tsx` には useEffect が7箇所ある（2026年7月時点の main）。例: 「`chatId`（URL パラメータ）が変わったとき、そのチャットの内容を読み込む/新規チャットとして初期化する」など。「トリガー = 依存配列」「やること = コールバックの中身」を対応づけて説明できていればOK。

## 第6章

**問6-1.**

```tsx
import { useState, useCallback } from 'react';

const useToggle = (initial: boolean): [boolean, () => void] => {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((prev) => !prev), []);
  return [value, toggle];
};

export default useToggle;
```

（`useCallback` は関数の再生成を防ぐ最適化。なくても動作は正しい）

**問6-2.**

```tsx
import { useState, useEffect } from 'react';

const useDebounce = (value: string, delay: number) => {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);  // value が変わったら前のタイマーを破棄
  }, [value, delay]);

  return debounced;
};

export default useDebounce;
```

クリーンアップで前回のタイマーをキャンセルするのが肝。入力が続く限り確定せず、止まって delay 経過後に確定する。

**問6-3.** `return` の後に `useState` があるため、`isAdmin` が false のときはフックが呼ばれず、true のときは呼ばれる。「フックは毎回同じ順番・同じ回数呼ばれる」というルールに違反。修正はフックを先頭に移動する:

```tsx
const Page = (props: { isAdmin: boolean }) => {
  const [logs, setLogs] = useState<string[]>([]);
  if (!props.isAdmin) {
    return <p>権限がありません</p>;
  }
  return <LogViewer logs={logs} />;
};
```

**問6-4.**
1. 状態: `opened`（サイドバーが開いているか）
2. 操作: `switchOpen`（開閉を反転）
3. 理由: 開閉ボタン（ヘッダー側）とサイドバー本体という**別々のコンポーネントが同じ状態を共有する**必要があるため。`useState` だと共通の親までリフトアップして Props で配る必要があるが、zustand ならどこからでも `useDrawer()` を呼ぶだけでよい。

## 第7章

**問7-1.**

```tsx
import { create } from 'zustand';

const useThemeState = create<{
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}>((set) => ({
  theme: 'light',
  toggleTheme: () => {
    set((state) => ({
      theme: state.theme === 'light' ? 'dark' : 'light',
    }));
  },
}));

const useTheme = () => {
  const theme = useThemeState((state) => state.theme);
  const toggleTheme = useThemeState((state) => state.toggleTheme);
  return { theme, toggleTheme };
};

export default useTheme;
```

**問7-2.**

```tsx
const Header = () => {
  const { toggleTheme } = useTheme();
  return <button onClick={toggleTheme}>テーマ切り替え</button>;
};

const Footer = () => {
  const { theme } = useTheme();
  return <footer>現在のテーマ: {theme}</footer>;
};
```

Header のボタンを押すと Footer の表示が変わる。2つのコンポーネントに親子関係がなくても状態が共有される点が確認ポイント。

**問7-3.** 入力中の値はその入力欄（とせいぜい親フォーム）しか使わないローカルな状態であり、グローバルに置くと (1) 関係ないコンポーネントから書き換え可能になり追跡が難しくなる、(2) キー入力のたびにストア購読側の再レンダリング範囲の管理が必要になる、(3) コンポーネントを削除してもストアに状態が残り続ける。ローカルで完結する状態は `useState` が適切（第7.5節の使い分け表を参照）。

**問7-4.** `useFiles.ts` のストアが管理する状態の例: `uploadedFilesDict`（画面IDごとのアップロード済みファイル）、`errorMessagesDict`（画面IDごとのエラーメッセージ）、`base64Cache`（ダウンロードした画像の base64 キャッシュ）。`useChat` と同様、`Record<string, ...>` で画面 ID ごとに状態を分けている。

## 第8章

**問8-1.** ①ローディング状態（isLoading フラグ）②エラー状態とハンドリング ③キャッシュ（同じデータを何度も取得しない仕組み）。ほかに、再取得（再検証）のタイミング制御、コンポーネントを跨いだデータ共有、アンマウント後に setState しないための後始末、など。

**問8-2.**

```tsx
import useSWR from 'swr';
import axios from 'axios';

type User = { id: number; name: string };

const fetcher = (url: string) => axios.get(url).then((res) => res.data);

const UserList = () => {
  const { data, isLoading, error } = useSWR<User[]>(
    'https://jsonplaceholder.typicode.com/users',
    fetcher
  );

  if (isLoading) return <p>読み込み中...</p>;
  if (error) return <p>エラーが発生しました</p>;
  return (
    <ul>
      {data?.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
};
```

**問8-3.** 原因: DELETE リクエストは成功しているが、一覧データは SWR のキャッシュから表示されているため、削除後に再検証（mutate）していないと古いキャッシュが表示され続ける。修正: 削除 API の成功後に一覧の `mutate()` を呼び、キャッシュを再取得・更新する。

**問8-4.**（2026年7月時点の main の例）
- `get`: `findChatById` が `http.get(chatId ? `chats/${chatId}` : null)` — チャット1件の取得。chatId が無いときは `null` を渡して通信を止める条件付き取得になっている点にも注目
- `post`: `createChat` が `http.post('chats', {})` — 新規チャットの作成。ほかに `predict`（LLM への推論リクエスト）なども post

## 第9章

**問9-1.** `<a href>` はブラウザがページ全体を再読み込みするため、SPA の状態（zustand のストアや SWR のキャッシュ）がすべて破棄され、JS・CSS も読み込み直しになる。`<Link>` は履歴だけを書き換えて React Router がコンポーネントを差し替えるので、高速で状態も保たれる。

**問9-2.**

```tsx
// main.tsx
import { createBrowserRouter, RouterProvider, Link, Outlet, useParams } from 'react-router-dom';

const Layout = () => (
  <div>
    <nav>
      <Link to="/">Home</Link> | <Link to="/items/1">Item 1</Link>
    </nav>
    <Outlet />
  </div>
);

const Home = () => <h1>Home</h1>;

const ItemDetail = () => {
  const { itemId } = useParams();
  return <h1>Item: {itemId}</h1>;
};

const NotFound = () => <h1>Not Found</h1>;

const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    children: [
      { path: '/', element: <Home /> },
      { path: '/items/:itemId', element: <ItemDetail /> },
      { path: '*', element: <NotFound /> },
    ],
  },
]);

// <RouterProvider router={router} /> を render する
```

**問9-3.** `main.tsx` の routes で `/summarize` は `enabled('summarize') ? {...} : null` と定義されており、機能が無効だとルート自体が除外されて `*`（NotFound）にマッチする。確認箇所: ①`main.tsx` の該当ルートの条件 ②その条件の元になるデプロイ設定（`packages/cdk/cdk.json` の context、および環境変数 `VITE_APP_XXX`）。

**問9-4.** `ChatPage` は `/chat` と `/chat/:chatId` の両方に割り当てられている。`useParams` で `chatId` を取得し、**ある場合**は既存チャットとしてそのメッセージ履歴を読み込んで続きから会話でき、**ない場合**は新規チャットとして空の状態から開始する（送信時に新しいチャットが作成される）。

## 第10章

**問10-1.** Flexbox で子要素を縦横中央に配置し、角丸（大）、白背景、内側余白 1rem、影付き。マウスを乗せると全体がやや暗くなる（brightness 90%）。カード状のクリック可能な要素の典型。

**問10-2.**
1. 送信ボタンの部品を特定する: `components/` で `ButtonSend.tsx` を見つける（またはチャット入力欄 `InputChatContent.tsx` から辿る）
2. その className を確認 → `bg-aws-smile` のようなカスタムカラーが使われている
3. カスタムカラーの定義場所 `packages/web/tailwind.config.js` の `theme.extend.colors` に行き着く
4. 変更方法は2択: 全体の色を変えるなら tailwind.config.js のカラー定義、ボタンだけ変えるなら該当コンポーネントの className

**問10-3.** (1) `<Outlet />` は `App.tsx` の 375 行目付近（`<main>` 相当の領域内）。(2) メニュー項目は `App.tsx` 内の `const items: ItemProps[] = [...]`（83行目付近）で定義され、`<Drawer items={items} />` でサイドバーに渡されている。新しいページをメニューに足すにはこの `items` 配列に追加する（＋ `main.tsx` の routes にもルートを追加）。

**問10-4.**（構成例。完全なコードは長いため骨格のみ）

```
src/
├── main.tsx           # ルート定義: Layout の children に / と /memo/:memoId
├── components/
│   └── Layout.tsx     # ナビ + <Outlet />
├── hooks/
│   └── useMemos.ts    # zustand ストア + ラップフック（GenU パターン）
└── pages/
    ├── MemoListPage.tsx    # 一覧・追加フォーム・削除ボタン
    └── MemoDetailPage.tsx  # useParams で memoId を取得して表示
```

```tsx
// hooks/useMemos.ts の骨格
const useMemoState = create<{
  memos: { id: string; text: string }[];
  addMemo: (text: string) => string;      // 追加して新しい id を返す
  deleteMemo: (id: string) => void;
}>((set) => ({
  memos: [],
  addMemo: (text) => {
    const id = crypto.randomUUID();
    set((state) => ({ memos: [...state.memos, { id, text }] }));  // イミュータブル
    return id;
  },
  deleteMemo: (id) => {
    set((state) => ({ memos: state.memos.filter((m) => m.id !== id) }));
  },
}));
```

チェックポイント:
- 追加・削除がイミュータブル更新になっているか
- 一覧 → 詳細が `<Link to={`/memo/${memo.id}`}>`、追加後が `navigate(`/memo/${id}`)` になっているか
- 詳細ページで `useParams` から id を取り、ストアから該当メモを引けているか
- 別ページに遷移してもメモが消えない（グローバル状態）ことを確認できたか
