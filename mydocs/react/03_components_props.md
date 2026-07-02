# 第3章 コンポーネントと Props

## 3.1 関数コンポーネント

React のコンポーネントは「Props を受け取って JSX を返す関数」です。

```tsx
type Props = {
  name: string;
};

const Greeting = (props: Props) => {
  return <h1>こんにちは、{props.name} さん</h1>;
};

export default Greeting;
```

使う側:

```tsx
<Greeting name="田中" />
```

ルール:

- コンポーネント名は**大文字で始める**（小文字だと HTML タグと解釈される）
- Props は**読み取り専用**。コンポーネント内で書き換えてはいけない（データは常に親から子への一方通行）

## 3.2 Props のいろいろな渡し方

```tsx
<UserCard
  name="田中"                    // 文字列
  age={30}                       // 数値・変数・式は {} で
  isActive={true}
  tags={['admin', 'dev']}
  onClick={() => alert('hi')}    // 関数
/>
```

### children — タグで挟んだ中身

タグの開始と終了で挟んだ内容は `children` という特別な Props として渡ります。

```tsx
type Props = {
  children: React.ReactNode;  // 「JSX として描画できるあらゆるもの」の型
};

const Card = (props: Props) => {
  return <div className="rounded border p-4">{props.children}</div>;
};

// 使う側
<Card>
  <h2>タイトル</h2>
  <p>この中身が children として渡る</p>
</Card>
```

GenU の `Button.tsx` も `children` を受け取り、ボタンのラベルとして表示しています。`<Button onClick={...}>送信</Button>` の「送信」が children です。

## 3.3 GenU の実コードを読む: Button.tsx

`packages/web/src/components/Button.tsx` の全体像です（コメントを追記）。

```tsx
import React from 'react';
import { BaseProps } from '../@types/common';
import { PiSpinnerGap } from 'react-icons/pi';

// BaseProps（className など共通 Props）を & で拡張する GenU の定番パターン
type Props = BaseProps & {
  title?: string;
  'aria-label'?: string;
  disabled?: boolean;
  loading?: boolean;
  outlined?: boolean;
  onClick: () => void;            // クリック時の動作は親が決める
  children: React.ReactNode;      // ボタンのラベル
};

const Button = React.forwardRef<HTMLButtonElement, Props>((props, ref) => {
  return (
    <button
      ref={ref}
      className={`${props.className ?? ''} ${
        props.outlined
          ? 'text-aws-font-color border-aws-font-color/20 border bg-white'
          : 'bg-aws-smile border text-white'
      }
      flex items-center justify-center rounded-lg p-1 px-3 ${
        props.disabled || props.loading ? 'opacity-30' : 'hover:brightness-75'
      }`}
      title={props.title}
      aria-label={props['aria-label']}
      onClick={props.onClick}
      disabled={props.disabled || props.loading}>
      {props.loading && <PiSpinnerGap className="mr-2 animate-spin" />}
      {props.children}
    </button>
  );
});

Button.displayName = 'Button';

export default Button;
```

読み取れる GenU の設計パターン:

1. **`BaseProps & {...}`** — 共通 Props を合成して型定義
2. **見た目のバリエーションは boolean Props で切り替え**（`outlined`, `loading`, `disabled`）
3. **動作（onClick）は親から注入** — Button 自身は「押されたら何をするか」を知らない
4. **`React.forwardRef`** — 親から内部の `<button>` DOM 要素へ参照を渡せるようにする仕組み。初心者のうちは「DOM を直接触りたい特殊ケース用のおまじない」と読み流してOK

## 3.4 コンポーネントの合成

小さい部品を組み合わせて大きい画面を作ります。GenU の `ButtonSend.tsx` は `Button` 系部品の一種として、`ChatPage` の入力欄で使われる…というように、階層的に組み立てられています。

```tsx
// 親: ページコンポーネント
const ChatPage = () => {
  const onSend = () => { /* 送信処理 */ };
  return (
    <div>
      {/* ... メッセージ一覧 ... */}
      <Button onClick={onSend} loading={isSending}>送信</Button>
    </div>
  );
};
```

親が「状態と処理」を持ち、子は「表示と操作の受け付け」に徹する。この役割分担が React の基本形です。

---

## 練習問題 3

**問3-1.** `label`（必須文字列）と `color`（省略可能、`'red' | 'blue'`）を受け取り、`<span>` でラベルを表示する `Badge` コンポーネントを作ってください。`color` 省略時は `'blue'` として扱うこと。

**問3-2.** 問3-1 の `Badge` に `children` も渡せるようにし、`label` の後ろに描画するよう変更してください。

**問3-3.** GenU の `packages/web/src/components/Card.tsx` を開き、(1) Props の型定義、(2) children を使っているか、を確認してください。`Button.tsx` と共通するパターンを2つ挙げてください。

**問3-4.** 「Props は読み取り専用」なのはなぜか、React のデータフローの観点から説明してください。

[解答例](./answers.md#第3章)
