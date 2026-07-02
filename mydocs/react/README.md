# React 教科書 — GenU のソースを読むための最短コース

React 初心者が **GenU (generative-ai-use-cases) のフロントエンドソースを読める・軽微な改修ができる** ようになることをゴールにした教科書です。

## GenU フロントエンドの技術スタック

この教科書は、GenU (`packages/web/`) で実際に使われている以下の技術に絞って解説します。

| 技術 | 役割 | 対応する章 |
|------|------|-----------|
| React 18 + TypeScript | UI ライブラリ | 第1〜6章 |
| Vite | ビルドツール・開発サーバー | 第1章 |
| zustand | グローバル状態管理 | 第7章 |
| SWR + axios | API 通信・データ取得 | 第8章 |
| react-router-dom v6 | 画面遷移（ルーティング） | 第9章 |
| Tailwind CSS | スタイリング | 第10章 |

## 目次

1. [第1章 React とは・開発環境](./01_react_basics.md) — React の考え方、Vite、プロジェクトの起動
2. [第2章 JSX と TypeScript の基礎](./02_jsx_typescript.md) — JSX 記法、型注釈、GenU コードを読むための TS 最低限
3. [第3章 コンポーネントと Props](./03_components_props.md) — 関数コンポーネント、Props、GenU の `Button.tsx` を読む
4. [第4章 state とイベント処理](./04_state_events.md) — useState、フォーム入力、条件付きレンダリング
5. [第5章 useEffect と副作用](./05_useeffect.md) — 副作用、依存配列、クリーンアップ
6. [第6章 カスタムフック](./06_custom_hooks.md) — ロジックの再利用、GenU のフック設計パターン
7. [第7章 zustand によるグローバル状態管理](./07_zustand.md) — GenU の `useDrawer` / `useChat` の仕組み
8. [第8章 SWR + axios によるデータ取得](./08_swr_axios.md) — GenU の `useHttp` を読み解く
9. [第9章 react-router-dom によるルーティング](./09_routing.md) — GenU の `main.tsx` のルート定義を読む
10. [第10章 Tailwind CSS と GenU ソースの歩き方](./10_tailwind_genu_tour.md) — スタイリング、ディレクトリ構成、1機能を端から端まで追う

- [練習問題 解答例](./answers.md)

## 学習の進め方

- 各章は「解説 → GenU の実コード → 練習問題」の構成です。順番に読んでください。
- 練習問題は必ず手を動かしてから [解答例](./answers.md) を見てください。
- GenU の実コードへの参照は `packages/web/src/...` のパスで示します。エディタで開きながら読むと効果的です。

## 前提知識

- HTML / CSS / JavaScript の基本文法（変数、関数、配列、オブジェクト）
- ターミナルの基本操作（`cd`, `npm` コマンド）

## 手を動かす環境の作り方

練習問題用に、ローカルに小さな React プロジェクトを作っておくと便利です。

```bash
# Vite で React + TypeScript プロジェクトを作成（GenU と同じ構成）
npm create vite@latest react-practice -- --template react-ts
cd react-practice
npm install
npm run dev
# → http://localhost:5173 で起動
```

GenU 本体をローカルで動かす手順は `docs/ja/DEVELOPMENT.md`（このリポジトリでは `dev-guide/ja/DEVELOPMENT.md` にもコピーあり）を参照してください。
