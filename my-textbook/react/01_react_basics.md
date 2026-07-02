# 第1章 React とは・開発環境

## 1.1 React とは

React は「UI をコンポーネント（部品）の組み合わせで作る」ための JavaScript ライブラリです。

従来の JavaScript（jQuery など）では「DOM を直接操作して画面を書き換える」スタイルでした。

```js
// 従来: DOM を命令的に操作する
document.getElementById('count').textContent = count + 1;
```

React では「**状態（データ）を宣言し、状態が変わったら React が自動で画面を更新する**」スタイルになります。

```jsx
// React: 状態を宣言する。画面更新は React に任せる
const [count, setCount] = useState(0);
return <div>{count}</div>; // count が変わると自動で再描画される
```

この「状態 → 画面」の一方向の流れが React の最重要コンセプトです。GenU のチャット画面も、「メッセージの配列」という状態が更新されると、チャット欄が自動で再描画される仕組みで動いています。

## 1.2 コンポーネントという考え方

React では画面を「コンポーネント」という部品に分割します。GenU のチャット画面なら:

```
ChatPage（ページ全体）
├── ChatList（会話履歴リスト）
│   └── ChatListItem（履歴1件）
├── ChatMessage（メッセージ1件の吹き出し）
└── InputChatContent（入力欄）
    └── ButtonSend（送信ボタン）
```

実際に `packages/web/src/components/` を見ると、`Button.tsx`、`Card.tsx`、`ChatMessage.tsx` など、部品ごとにファイルが分かれています。**1ファイル = 1コンポーネント** が基本です。

## 1.3 Vite と開発サーバー

GenU は **Vite**（ヴィート）というビルドツールを使っています。役割は2つです。

- **開発サーバー**: `npm run dev` で起動。コードを保存すると即座にブラウザへ反映（ホットリロード）
- **ビルド**: `npm run build` で本番用に TypeScript をコンパイルし、ファイルをまとめて圧縮

GenU の `packages/web/package.json` の scripts を見てみましょう。

```json
"scripts": {
  "dev": "vite",
  "build": "tsc && vite build",
  "lint": "eslint . --ext ts,tsx,yaml,yml ...",
  "preview": "vite preview",
  "test": "vitest run"
}
```

GenU をローカル起動する場合は、リポジトリルートで `npm run web:devw` を実行します（AWS 環境の情報を自動取得してから Vite を起動するラッパーです。詳細は `docs/ja/DEVELOPMENT.md`）。

## 1.4 エントリーポイント: index.html → main.tsx

React アプリの起動の流れは次のとおりです。

```
index.html            ← ブラウザが最初に読む。<div id="root"> がある
  └── main.tsx        ← React をここで起動。<div id="root"> に描画する
        └── App.tsx / 各ページ ← 実際の画面
```

GenU の `packages/web/src/main.tsx` の末尾（抜粋・簡略化）:

```tsx
import ReactDOM from 'react-dom/client';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>
);
```

- `createRoot(...).render(...)` が「この HTML 要素の中に React アプリを描画せよ」という命令
- `RouterProvider` はルーティング（第9章）の起点
- `React.StrictMode` は開発時に問題を検出しやすくするためのラッパー

## 1.5 ファイル拡張子の意味

| 拡張子 | 意味 |
|--------|------|
| `.tsx` | TypeScript + JSX（コンポーネントを書くファイル） |
| `.ts` | TypeScript のみ（ロジック・型定義・フックなど） |

GenU では、コンポーネントは `.tsx`、フック（`hooks/`）やユーティリティ（`utils/`）は `.ts` が多いですが、JSX を返すかどうかで決まります。

---

## 練習問題 1

**問1-1.** 従来の DOM 操作と React の最大の違いを一言で説明してください。

**問1-2.** 自分のPCに練習用プロジェクトを作って起動してください。

```bash
npm create vite@latest react-practice -- --template react-ts
cd react-practice
npm install
npm run dev
```

ブラウザで http://localhost:5173 を開き、`src/App.tsx` の中の文字列を書き換えて保存し、画面が即座に変わることを確認してください。

**問1-3.** GenU の `packages/web/src/main.tsx` を開き、`createRoot` を呼んでいる行を見つけてください。`document.getElementById('root')` の `'root'` はどのファイルで定義されているか探してください（ヒント: `packages/web/` 直下）。

[解答例](./answers.md#第1章)
