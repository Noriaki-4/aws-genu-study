# 第7章 zustand によるグローバル状態管理

## 7.1 なぜグローバル状態管理が必要か

`useState` の state はそのコンポーネントの中だけのものです。離れたコンポーネント同士で共有するには親へリフトアップ（第4章）しますが、アプリ全体で使う状態を毎回 Props で配るのは大変です（Props のバケツリレー問題）。

GenU では、**どのコンポーネントからでも読み書きできる状態置き場**として **zustand**（ツースタンド）を使っています。

- サイドバー（Drawer）の開閉状態 → `useDrawer.ts`
- チャットのメッセージ・ローディング状態 → `useChat.ts`
- アップロード中のファイル → `useFiles.ts`

## 7.2 基本の使い方

zustand では `create` で「ストア」を作ります。ストアは **状態** と **状態を更新する関数** をまとめたオブジェクトです。

```tsx
import { create } from 'zustand';

const useCounterStore = create<{
  count: number;
  increment: () => void;
}>((set) => ({
  count: 0,
  increment: () => {
    set((state) => ({ count: state.count + 1 }));
  },
}));
```

- `create<型>((set) => ({...}))` — 初期状態と更新関数を定義
- `set` — 状態を更新する関数。**渡したオブジェクトが既存の状態にマージされる**
- `set((state) => ({...}))` — 現在の状態から次の状態を計算する形（useState の関数型更新と同じ発想）

コンポーネントからはフックとして呼ぶだけです。Provider などの準備は不要です。

```tsx
const CounterButton = () => {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);
  return <button onClick={increment}>{count}</button>;
};
```

`useCounterStore((state) => state.count)` の関数は**セレクタ**と呼ばれ、「ストアのどの部分を使うか」を指定します。選んだ部分が変わったときだけ再レンダリングされるので、無駄な描画を防げます。

## 7.3 GenU の実コードを読む: useDrawer.ts

`packages/web/src/hooks/useDrawer.ts` の全文です。GenU の zustand パターンの最小例として最適です。

```tsx
import { create } from 'zustand';

// (1) zustand ストア: サイドバーの開閉状態
const useDrawerState = create<{
  opened: boolean;
  switchOpen: () => void;
}>((set) => {
  return {
    opened: false,
    switchOpen: () => {
      set((state) => ({
        opened: !state.opened,   // イミュータブルに反転
      }));
    },
  };
});

// (2) ストアを直接公開せず、カスタムフックでラップして公開
const useDrawer = () => {
  const [opened, switchOpen] = useDrawerState((state) => [
    state.opened,
    state.switchOpen,
  ]);

  return {
    opened,
    switchOpen,
  };
};

export default useDrawer;
```

GenU の設計パターン:

1. **ストア（`useDrawerState`）は export しない** — 外部にはカスタムフック `useDrawer` だけを公開し、状態の実装方法（zustand か useState か）を隠蔽する
2. **利用側は第6章と同じ「状態 + 操作」インターフェース** — `const { opened, switchOpen } = useDrawer();`

これにより、ヘッダーのハンバーガーボタンとサイドバー本体という離れたコンポーネントが、同じ `opened` を共有できます。

## 7.4 GenU の実コードを読む: useChat.ts の骨格

チャット機能の中核 `packages/web/src/hooks/useChat.ts` も同じ構造です（大幅に簡略化）。

```tsx
const useChatState = create<{
  chats: { [id: string]: ChatMessages };   // 画面ごとのメッセージを ID で管理
  loading: { [id: string]: boolean };
  setLoading: (id: string, newLoading: boolean) => void;
  pushMessage: (id: string, role: string, content: string) => void;
  // ... 多数の状態と操作 ...
}>((set, get) => { ... });

// ページからは useChat('/chat') のように「画面の ID」を渡して使う
const useChat = (id: string) => {
  const { chats, loading, post, ... } = useChatState();
  return {
    loading: loading[id] ?? false,
    messages: chats[id]?.messages ?? [],
    postChat: (content: string) => { ... },
    ...
  };
};
```

ポイント:

- GenU は複数のユースケース画面（チャット、要約、翻訳…）が同じチャット機構を使うため、**`id`（画面のパス）をキーにして状態を辞書で分けて**います
- `create` のコールバックは `(set, get)` の2引数も取れます。`get()` で現在の状態を読めます

`useChat.ts` は900行超の大きなファイルですが、「zustand ストア定義」+「それをラップするカスタムフック」という useDrawer と同じ構造だとわかって読めば怖くありません。

## 7.5 useState と zustand の使い分け

| 状態の性質 | 使うもの |
|-----------|---------|
| 1つのコンポーネント内で完結（入力中の値、モーダル開閉など） | `useState` |
| 複数の離れたコンポーネントで共有 / 画面遷移しても保持したい | zustand |
| サーバーから取得するデータ | SWR（第8章） |

---

## 練習問題 7

**問7-1.** テーマ（`'light' | 'dark'`）を管理する zustand ストアと、それをラップする `useTheme` カスタムフックを作ってください。`theme`（現在値）と `toggleTheme`（切り替え）を返すこと。GenU の useDrawer.ts の構造を真似ること。

**問7-2.** 問7-1 の `useTheme` を使って、(1) ヘッダーの切り替えボタン、(2) 現在のテーマ名を表示するフッター、の2コンポーネントを作り、状態が共有されることを確認してください。

**問7-3.** 「入力フォームの入力中の値」を zustand で管理するのは一般に過剰とされます。なぜか説明してください。

**問7-4.** GenU の `packages/web/src/hooks/useFiles.ts` の先頭部分を読み、zustand ストアがどんな状態を管理しているか、状態のキーを3つ挙げてください。

[解答例](./answers.md#第7章)
