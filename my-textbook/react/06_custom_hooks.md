# 第6章 カスタムフック

## 6.1 カスタムフックとは

`useState` や `useEffect` を組み合わせたロジックを **`use` で始まる関数**に切り出したものがカスタムフックです。GenU の `packages/web/src/hooks/` には約40個のカスタムフックがあり、**GenU のロジックはほぼすべてカスタムフックに書かれています**。ここが読めれば GenU の中身は理解できたも同然です。

```tsx
// カスタムフック: ウィンドウ幅を返す
import { useState, useEffect } from 'react';

const useWindowWidth = () => {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const onResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', onResize);
    return () => window.removeEventListener('resize', onResize);
  }, []);

  return width;
};

export default useWindowWidth;
```

使う側はロジックの中身を知らなくてよくなります。

```tsx
const Header = () => {
  const width = useWindowWidth();
  return <div>{width < 768 ? <MobileMenu /> : <DesktopMenu />}</div>;
};
```

## 6.2 フックのルール

1. **名前は `use` で始める**
2. **コンポーネントまたは他のフックのトップレベルでのみ呼ぶ** — if 文・ループ・通常関数の中では呼ばない（React はフックを「呼ばれた順番」で管理しているため）

```tsx
// NG
if (isLoggedIn) {
  const [name, setName] = useState('');
}

// OK: フックは無条件に呼び、条件は中身/表示で分岐する
const [name, setName] = useState('');
if (!isLoggedIn) return <Login />;
```

## 6.3 GenU のフック設計パターン

GenU の `hooks/` を読むと、役割ごとに命名のパターンがあります。

| パターン | 例 | 役割 |
|----------|-----|------|
| `useXxx` | `useChat`, `useDrawer` | 状態 + 操作をまとめた「機能の窓口」 |
| `useXxxApi` | `useChatApi`, `useImageApi` | API 呼び出しだけを担当（第8章） |

ページコンポーネントは `useXxx` を呼び、`useXxx` が内部で `useXxxApi` を呼ぶ、という層構造です。

```
ChatPage.tsx（表示）
  └── useChat.ts（状態管理 + ロジック）
        └── useChatApi.ts（API 通信）
              └── useHttp.ts（axios + SWR の共通基盤）
```

この分離のおかげで、「画面の見た目を変えたい → pages/」「チャットの挙動を変えたい → hooks/useChat.ts」「API の呼び方を変えたい → hooks/useChatApi.ts」と、目的からファイルを特定できます。

## 6.4 「状態 + 操作」を返すインターフェース

GenU のフックは、オブジェクトで「状態」と「操作関数」をセットで返す形が定番です。

```tsx
// GenU の useDrawer.ts（実物・全文）を単純化した骨格
const useDrawer = () => {
  // ...状態の定義...
  return {
    opened,      // 状態
    switchOpen,  // 操作
  };
};
```

呼び出し側は分割代入で必要なものだけ取り出します。

```tsx
const { opened, switchOpen } = useDrawer();
```

`useChat` のような大きなフックになると、`messages`, `loading`, `postChat`, `clear` など多数の状態・操作を返します。**フックの return 文を見れば、そのフックが提供する機能の一覧がわかる** —— GenU のフックを読むときはまず return を見るのがコツです。

## 6.5 実践: フックへの切り出し判断

次のような場合にカスタムフックへ切り出します。

- 同じロジック（state + effect の組み合わせ）を複数コンポーネントで使う
- コンポーネントが長くなり、表示とロジックを分離したい
- ロジック単体でテストしたい

逆に、単なる計算（state を持たない関数）はフックにせず、`utils/` の通常関数にします。GenU も `packages/web/src/utils/` に日付整形などの純粋関数を置いています。

---

## 練習問題 6

**問6-1.** `useToggle(initial: boolean)` というカスタムフックを作ってください。`[value, toggle]`（現在値と、呼ぶたびに true/false を反転する関数）を返すこと。

**問6-2.** 入力値の変更が止まってから 500ms 後に確定値を返す `useDebounce(value: string, delay: number)` を作ってください（ヒント: `useEffect` + `setTimeout` + クリーンアップ）。

**問6-3.** 次のコードはフックのルールに違反しています。理由を説明し、修正してください。

```tsx
const Page = (props: { isAdmin: boolean }) => {
  if (!props.isAdmin) {
    return <p>権限がありません</p>;
  }
  const [logs, setLogs] = useState<string[]>([]);
  return <LogViewer logs={logs} />;
};
```

**問6-4.** GenU の `packages/web/src/hooks/useDrawer.ts`（30行弱）を読み、(1) このフックが管理している状態、(2) 提供している操作、(3) なぜこれが `useState` ではなく zustand で実装されているのか（推測でOK、答えは第7章）を書いてください。

[解答例](./answers.md#第6章)
