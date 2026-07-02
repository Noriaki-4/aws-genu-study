# 第4章 state とイベント処理

## 4.1 useState — コンポーネントの状態

Props は親から渡される読み取り専用の値でした。コンポーネント自身が持ち、変更できる値が **state** です。

```tsx
import { useState } from 'react';

const Counter = () => {
  const [count, setCount] = useState(0);
  //     ^現在の値  ^更新関数        ^初期値

  return (
    <div>
      <p>{count} 回クリックされました</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
};
```

重要な仕組み:

1. `setCount` を呼ぶと React がコンポーネント関数を**再実行**（再レンダリング）する
2. 再実行時、`useState` は最新の値を返すので画面が更新される
3. `count = count + 1` のような**直接代入では画面は更新されない**（React が変更を検知できない）

### 型の指定

```tsx
const [text, setText] = useState('');              // 初期値から string と推論される
const [user, setUser] = useState<User | null>(null); // 複雑な型は明示する
const [items, setItems] = useState<string[]>([]);
```

GenU では `useState<boolean>(false)` のように明示している箇所が多くあります。

## 4.2 イベント処理

```tsx
// クリック
<button onClick={() => handleClick()}>実行</button>
<button onClick={handleClick}>実行</button>   // 引数がなければこの書き方でも同じ

// テキスト入力
const [text, setText] = useState('');
<input value={text} onChange={(e) => setText(e.target.value)} />
```

### 制御されたコンポーネント（controlled component）

上の input のように「`value` に state を渡し、`onChange` で state を更新する」形を制御されたコンポーネントと呼びます。**入力欄の値の正体は常に state** であり、画面はその写しです。GenU の入力欄（`Textarea.tsx` など）もこの形です。

```tsx
// GenU 風: 親が value と onChange を Props で注入する
type Props = {
  value: string;
  onChange: (value: string) => void;
};

const Textarea = (props: Props) => {
  return (
    <textarea
      value={props.value}
      onChange={(e) => props.onChange(e.target.value)}
    />
  );
};
```

GenU のコンポーネントは `onChange: (value: string) => void` のように「イベントオブジェクトではなく値そのもの」を親に渡すインターフェースで統一されています。使う側がシンプルになるためです。

## 4.3 state 更新の注意点

### (1) 更新は「予約」であり即時ではない

```tsx
const handleClick = () => {
  setCount(count + 1);
  console.log(count);  // まだ古い値が表示される
};
```

`setCount` は「次の再レンダリングで新しい値にする」予約です。連続更新するときは**関数型更新**を使います。

```tsx
setCount((prev) => prev + 1);  // prev は必ず最新の値
setCount((prev) => prev + 1);  // 2回とも正しく反映される
```

### (2) 配列・オブジェクトは新しく作り直す（イミュータブル更新）

React は「参照が変わったか」で変更を検知します。**元の配列やオブジェクトを直接書き換えても再レンダリングされません。**

```tsx
// NG: 元の配列を破壊的に変更 → 画面が更新されない
items.push(newItem);
setItems(items);

// OK: スプレッド構文で新しい配列を作る
setItems([...items, newItem]);

// OK: 削除は filter
setItems(items.filter((item) => item.id !== targetId));

// OK: オブジェクトの一部更新
setUser({ ...user, name: '新しい名前' });
```

GenU のチャットメッセージ追加なども、必ず新しい配列を作る形で実装されています。

## 4.4 state をどこに置くか — リフトアップ

兄弟コンポーネント間で state を共有したいときは、**共通の親に state を持ち上げ（lift up）**、Props で配ります。

```tsx
const Parent = () => {
  const [keyword, setKeyword] = useState('');
  return (
    <>
      <SearchBox value={keyword} onChange={setKeyword} />
      <SearchResult keyword={keyword} />
    </>
  );
};
```

ただし、アプリ全体で共有したい状態（ログインユーザー、チャット履歴など）を毎回リフトアップすると Props のバケツリレーが深くなります。GenU はこの問題を **zustand**（第7章）で解決しています。

---

## 練習問題 4

**問4-1.** 「+1」「-1」「リセット」の3つのボタンを持つカウンターコンポーネントを作ってください。

**問4-2.** テキスト入力欄と「追加」ボタンがあり、追加ボタンを押すと入力内容がリストに追加され、入力欄が空になる TODO リストコンポーネントを作ってください（イミュータブル更新を守ること）。

**問4-3.** 次のコードにはバグがあります。どこが問題で、どう直すべきか説明してください。

```tsx
const [tags, setTags] = useState<string[]>([]);

const addTag = (tag: string) => {
  tags.push(tag);
  setTags(tags);
};
```

**問4-4.** GenU の `packages/web/src/components/Textarea.tsx` を開き、`value` と `onChange` がどう Props 化されているか確認してください。`onChange` の型はイベントオブジェクトを渡す形と値を渡す形のどちらですか。

[解答例](./answers.md#第4章)
