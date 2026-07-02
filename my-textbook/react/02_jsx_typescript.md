# 第2章 JSX と TypeScript の基礎

GenU のソースは全て TypeScript + JSX（= `.tsx`）で書かれています。この章では、GenU のコードを読むのに必要な最低限の文法を押さえます。

## 2.1 JSX とは

JSX は「JavaScript の中に HTML のようなタグを書ける」構文です。

```tsx
const element = <h1>こんにちは</h1>;
```

見た目は HTML ですが、実体は JavaScript の式です。そのため変数に代入したり、関数から return したりできます。

### HTML との違い（重要）

| HTML | JSX | 理由 |
|------|-----|------|
| `class="btn"` | `className="btn"` | `class` は JS の予約語 |
| `for="name"` | `htmlFor="name"` | `for` は JS の予約語 |
| `onclick="..."` | `onClick={...}` | キャメルケース + 関数を渡す |
| `<br>` | `<br />` | タグは必ず閉じる |

### `{}` で JavaScript を埋め込む

JSX 内の `{}` の中には任意の JavaScript 式が書けます。

```tsx
const name = 'GenU';
const element = <h1>ようこそ {name} へ。今日は {new Date().getMonth() + 1} 月です</h1>;
```

### 複数要素は1つの親で包む

コンポーネントが返す JSX は1つのルート要素でなければなりません。余計な `div` を増やしたくないときは **フラグメント** `<>...</>` を使います。

```tsx
return (
  <>
    <h1>タイトル</h1>
    <p>本文</p>
  </>
);
```

## 2.2 条件付きレンダリング

GenU のソースで頻出のパターンです。

```tsx
// パターン1: && （条件が true のときだけ表示）
{props.loading && <PiSpinnerGap className="mr-2 animate-spin" />}

// パターン2: 三項演算子（どちらかを表示）
{isOpen ? <Drawer /> : null}

// パターン3: クラス名の切り替え（GenU の Button.tsx より）
className={props.outlined ? 'border bg-white' : 'bg-aws-smile text-white'}
```

パターン1は GenU の `Button.tsx` に実際にあるコードで、「loading が true のときだけスピナーアイコンを表示する」という意味です。

## 2.3 リストのレンダリング

配列を画面に並べるには `map` を使います。GenU のチャット履歴・メッセージ一覧はすべてこのパターンです。

```tsx
const messages = ['こんにちは', '調子はどう？', '元気です'];

return (
  <ul>
    {messages.map((msg, idx) => (
      <li key={idx}>{msg}</li>
    ))}
  </ul>
);
```

- `key` は React が「どの要素が追加・削除されたか」を判別するための必須属性
- 配列の中身に一意な ID があるなら index より ID を使うのが望ましい（GenU では `key={chat.chatId}` のように ID を使っています）

## 2.4 GenU を読むための TypeScript 最低限

### 型注釈

変数や引数に「この値はこの型」と注釈をつけます。

```ts
const title: string = 'GenU';
const count: number = 0;
const visible: boolean = true;
const tags: string[] = ['ai', 'aws'];       // 文字列の配列

function greet(name: string): string {      // 引数と戻り値の型
  return `Hello, ${name}`;
}
```

### type でオブジェクトの形を定義

GenU で最も頻出なのがこれです。コンポーネントの Props（第3章）は必ず `type` で定義されています。

```ts
type Props = {
  title: string;        // 必須
  disabled?: boolean;   // ? は「省略可能」
  onClick: () => void;  // 「引数なし・戻り値なしの関数」型
};
```

`() => void` は関数の型です。`(value: string) => void` なら「string を1つ受け取る関数」です。

### ユニオン型と型の合成

```ts
// ユニオン型: どちらかの値を取る
type Status = 'loading' | 'success' | 'error';

// & による合成: GenU の Button.tsx はこのパターン
type Props = BaseProps & {
  title?: string;
  onClick: () => void;
};
```

GenU の `BaseProps`（`packages/web/src/@types/common.d.ts`）は `className` などの全コンポーネント共通の Props を定義していて、各コンポーネントは `BaseProps & {...}` で拡張します。

### ジェネリクス（読めればOK）

`<T>` は「型を後から差し込める」仕組みです。GenU では `useState<string>('')` や `useSWR<Data, Error>(...)` の形で頻出します。「`<>` の中は扱うデータの型を指定している」と読めれば十分です。

### よく見る記号

```ts
const token = session.tokens?.idToken;  // ?. : null/undefined なら止まる（オプショナルチェーン）
const cls = props.className ?? '';      // ?? : 左が null/undefined なら右を使う
document.getElementById('root')!        // !  : 「null ではない」と断言（多用は避ける）
```

## 2.5 import / export

```ts
// default export（1ファイル1つ）: GenU のコンポーネントはほぼこれ
export default Button;
import Button from '../components/Button';   // 名前は自由に付けられる

// named export（複数可）: 定数やユーティリティに使われる
export const MODELS = {...};
import { MODELS } from './hooks/useModel';   // 名前は一致させる必要がある
```

---

## 練習問題 2

**問2-1.** 次の HTML を JSX に書き直してください。

```html
<div class="card">
  <label for="username">名前</label>
  <input id="username">
  <br>
</div>
```

**問2-2.** `items: { id: string; label: string }[]` という配列を `<ul>` のリストとして表示する JSX を書いてください。`key` を適切に設定すること。

**問2-3.** 「`text` は必須の文字列、`count` は省略可能な数値、`onSelect` は文字列を1つ受け取る関数」という Props の `type Props` を定義してください。

**問2-4.** GenU の `packages/web/src/components/Button.tsx` を開き、(1) 条件付きレンダリング（`&&`）と (2) クラス名の三項演算子切り替え が使われている行をそれぞれ見つけてください。

[解答例](./answers.md#第2章)
