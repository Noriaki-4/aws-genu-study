# 第5章 useEffect と副作用

## 5.1 副作用とは

コンポーネントの本業は「Props と state から JSX を作る」ことです。それ以外の処理 —— API 呼び出し、タイマー設定、ドキュメントタイトル変更、イベントリスナー登録など —— を**副作用（side effect）**と呼び、`useEffect` の中に書きます。

```tsx
import { useEffect, useState } from 'react';

const Clock = () => {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    // 副作用: タイマーの設定
    const timer = setInterval(() => setTime(new Date()), 1000);

    // クリーンアップ: コンポーネントが消えるときに実行される
    return () => clearInterval(timer);
  }, []); // 依存配列

  return <p>{time.toLocaleTimeString()}</p>;
};
```

## 5.2 依存配列 — いつ実行されるか

`useEffect` の第2引数（依存配列）で実行タイミングが決まります。

| 依存配列 | 実行タイミング |
|----------|----------------|
| `[]` | 初回マウント時に1回だけ |
| `[a, b]` | 初回 + `a` か `b` が変わるたび |
| なし（省略） | 毎回の再レンダリング後（ほぼ使わない） |

```tsx
// chatId が変わるたびに、そのチャットのタイトルを設定し直す
useEffect(() => {
  document.title = `Chat: ${chatId}`;
}, [chatId]);
```

**依存配列には「effect 内で使っている Props / state をすべて列挙する」**のが原則です。GenU の ESLint 設定はこれをチェックします（`react-hooks/exhaustive-deps`）。

## 5.3 クリーンアップ

`useEffect` から関数を return すると、「次の effect 実行の前」と「コンポーネントが画面から消える（アンマウント）とき」に呼ばれます。タイマー解除、イベントリスナー解除、購読解除などに使います。

```tsx
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize); // 解除を忘れない
}, []);
```

クリーンアップを忘れるとメモリリークや多重登録の原因になります。

## 5.4 GenU での useEffect の使われどころ

GenU のページコンポーネントでは、次のようなパターンで使われています。

```tsx
// パターン1: URL パラメータが変わったらデータを読み込み直す
// （ChatPage.tsx などの典型形・簡略化）
const { chatId } = useParams();

useEffect(() => {
  if (chatId) {
    loadChat(chatId);
  }
}, [chatId]);

// パターン2: 初回マウント時に初期化処理
useEffect(() => {
  clear();          // 画面を初期状態に
}, []);

// パターン3: メッセージが増えたら一番下までスクロール
useEffect(() => {
  scrollToBottom();
}, [messages]);
```

なお、**「API からデータを取得して表示する」だけなら GenU では useEffect ではなく SWR（第8章）を使う**のが基本です。useEffect での手動 fetch はローディング・エラー・キャッシュの管理を全部自分で書くことになるためです。この役割分担（取得は SWR、それ以外の副作用は useEffect）を覚えておくと GenU のコードが読みやすくなります。

## 5.5 よくある間違い

### (1) 依存配列に入れ忘れて古い値を掴む

```tsx
// NG: count を使っているのに依存配列に入れていない
useEffect(() => {
  const timer = setInterval(() => {
    console.log(count);  // ずっと初期値の 0 が表示される
  }, 1000);
  return () => clearInterval(timer);
}, []);  // ← [count] にするか、関数型更新を使う
```

### (2) effect 内で state を更新して無限ループ

```tsx
// NG: items を更新すると再レンダリング → effect 再実行 → また更新…
useEffect(() => {
  setItems([...items, 'new']);
}, [items]);
```

依存に入れた値を effect 内で更新する構造になっていないか確認しましょう。

---

## 練習問題 5

**問5-1.** マウントから経過した秒数を表示するコンポーネント `ElapsedTimer` を作ってください。クリーンアップも正しく実装すること。

**問5-2.** Props で `title: string` を受け取り、その値をブラウザのタブタイトル（`document.title`）に反映するコンポーネントを作ってください。`title` が変わったら追従すること。

**問5-3.** 次のコードの問題点を2つ指摘してください。

```tsx
const SearchResult = (props: { keyword: string }) => {
  const [results, setResults] = useState<string[]>([]);

  useEffect(() => {
    window.addEventListener('scroll', handleScroll);
    fetch(`/api/search?q=${props.keyword}`)
      .then((res) => res.json())
      .then((data) => setResults(data));
  }, []);
  // ...
};
```

**問5-4.** GenU の `packages/web/src/pages/ChatPage.tsx` を開き、`useEffect` が何箇所で使われているか数え、そのうち1つを選んで「何をトリガーに」「何をしているか」を説明してください。

[解答例](./answers.md#第5章)
