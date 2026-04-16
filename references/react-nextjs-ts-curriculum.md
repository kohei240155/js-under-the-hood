# React / Next.js / TypeScript 深層理解 & 面接対策カリキュラム

> 北米テック企業のフロントエンド面接で、React の内部動作・Next.js のアーキテクチャ・TypeScript の型システムを自分の言葉で説明し、ライブコーディングで実装できるようになるためのカリキュラム。全セクション手を動かす演習形式。

---

## 1. React コア概念の深層理解

### 1-1. JSX の正体とレンダリングの仕組み
- JSX を書き、Babel の REPL（https://babeljs.io/repl）でトランスパイル結果を確認し、JSX が `React.createElement()` の呼び出しに変換されることを目で見る
- `React.createElement` を直接使ってコンポーネントを書き、JSX 版と出力が同一であることを確認する
- React 17+ の新しい JSX Transform（`jsx-runtime`）との違いを調べ、`import React from 'react'` が不要になった理由を自分の言葉で説明する文章を書く

### 1-2. state と props の本質
- 親コンポーネントから子に `props` を渡し、子が `props` を直接書き換えようとしたときのエラーを確認する（読み取り専用であることの体感）
- `useState` で管理する値をオブジェクトにし、ミューテーション（直接変更）してもUIが更新されないことを確認する → スプレッド構文で新しいオブジェクトを作ると更新される理由を Object.is による比較から説明する
- 同じ `state` を子コンポーネント 3 つに渡して、「state のリフトアップ」パターンを実装する → props drilling の煩雑さを体感する
- 子 → 親へのデータ伝達をコールバック関数で実装し、単方向データフローの中で逆方向にデータを流すパターンを理解する

### 1-3. 再レンダリングの仕組みと最適化
- 親コンポーネントの state を変更し、`console.log` を仕込んだ子コンポーネントがすべて再レンダリングされることを確認する
- `React.memo` で子をラップし、`props` が変わらなければ再レンダリングがスキップされることを確認する
- `React.memo` を使っていても、親が新しいオブジェクト / 関数を props として渡すとメモ化が壊れることを実験する
- 上記を `useMemo` / `useCallback` で修正し、参照の安定性がメモ化に必要な理由を体感する
- React DevTools の Profiler で「なぜこのコンポーネントが再レンダリングされたか」を確認する方法を実践する

### 1-4. Hooks の深層理解

#### useState
- `useState` の初期値にコストの高い計算を入れ、毎レンダリングで実行されることを確認する → 遅延初期化（関数を渡すパターン）で解消する
- `setState` のバッチ処理を実験する（React 18 以降は `setTimeout` 内でもバッチされることを確認）
- `setState` に関数を渡すパターン（`setCount(prev => prev + 1)`）と値を渡すパターンの違いを、連続呼び出しで実験する

#### useEffect
- `useEffect` の依存配列を空 / 特定の値 / 省略の 3 パターンで書き、実行タイミングの違いを `console.log` で可視化する
- クリーンアップ関数が「次の effect 実行前」と「アンマウント時」に呼ばれることを確認する実験を書く
- `setInterval` を `useEffect` 内で使い、クリーンアップしないとインターバルが重複していくバグを再現 → 修正する
- `useEffect` 内で fetch してデータを取得するパターンを書き、クリーンアップで `AbortController` を使ってメモリリークを防ぐ実装にする

#### useRef
- `useRef` で DOM 要素への参照を取得し、マウント時にフォーカスを当てる実装をする
- `useRef` で「前回のレンダリング時の値」を保持する `usePrevious` カスタムフックを自作する
- `useRef` の値を変更しても再レンダリングが起きないことを確認し、`useState` との使い分けの基準を自分の言葉でまとめる

#### useReducer
- `useState` で管理している複雑なフォーム状態を `useReducer` に書き直し、アクションによる状態遷移の明確さを体感する
- `dispatch` の安定性（再レンダリングしても参照が変わらない）を確認し、`useCallback` が不要になるケースを実験する
- `useReducer` + `useContext` で簡易 Redux を構築する

#### useMemo / useCallback
- 重い計算（例：10000 件のデータのフィルタリング＆ソート）を `useMemo` なしで書き、入力のたびに遅延が発生することを確認する → `useMemo` で計算結果をキャッシュして解消する
- `useCallback` で関数をメモ化し、`React.memo` された子コンポーネントに渡したときに再レンダリングが防げることを確認する
- 不必要な `useMemo` / `useCallback` のオーバーヘッドを計測する実験を書き、「何でもメモ化すればいいわけではない」ことを理解する

### 1-5. カスタムフックの設計と実装
- `useLocalStorage(key, initialValue)` — localStorage と state を同期するフックを自作する
- `useDebounce(value, delay)` — 検索入力に使えるデバウンスフックを自作する
- `useFetch(url)` — data / loading / error を返す汎用データ取得フックを自作する（AbortController によるクリーンアップ込み）
- `useMediaQuery(query)` — レスポンシブ対応のフックを自作する（`window.matchMedia` を内部で使う）
- `useIntersectionObserver(ref, options)` — 要素の可視状態を監視するフックを自作する

### 1-6. Controlled vs Uncontrolled コンポーネント
- テキスト入力を `useState` で制御する Controlled パターンと、`useRef` で値を取得する Uncontrolled パターンの両方で同じフォームを実装する
- ファイルアップロード input が Uncontrolled でしか実装できない理由を実験で確認する
- React Hook Form のような非制御ベースのフォームライブラリの設計思想を理解するため、`register` 関数の簡易版を自作する

### 1-7. コンポーネント設計パターン
- Compound Components パターン：`<Select>` + `<Select.Option>` のように親子が暗黙の状態を共有するコンポーネントを `React.Children` と `cloneElement` で実装する → Context 版でリファクタリングする
- Render Props パターン：マウス座標を追跡する `<MouseTracker>` コンポーネントを render props で実装する → カスタムフックで書き直して比較する
- HOC（Higher-Order Component）パターン：`withAuth(Component)` のような認証ガードを HOC で実装する → カスタムフックで書き直して比較する
- 3 つのパターンのメリット・デメリットを表にまとめ、現代の React では Hooks が主流である理由を自分の言葉で説明する

### 1-8. Context API の深掘り
- `createContext` + `Provider` + `useContext` でテーマ切り替え（ライト / ダーク）を実装する
- Context の値が変わると、`useContext` を使っているすべてのコンポーネントが再レンダリングされることを確認する
- 上記の問題を Context の分割（別々の Context に分ける）で解消するパターンを実装する
- `useMemo` で Provider の value をメモ化して不要な再レンダリングを防ぐパターンを実装する
- Context を多段に使いすぎる「Provider 地獄」を体験し、状態管理ライブラリ（Zustand など）との使い分けの判断基準をまとめる

---

## 2. React の内部アーキテクチャ

### 2-1. Reconciliation（差分検出）アルゴリズム
- 同じ位置に同じ型のコンポーネントを置いた場合と、異なる型に切り替えた場合で、DOM が更新されるか全置換されるかを DevTools で確認する
- リストを `key` なし / `key={index}` / `key={uniqueId}` でレンダリングし、並び替え時に DOM ノードが再利用されるかどうかを確認する
- `key` に `Math.random()` を指定するとどうなるかを実験し、パフォーマンスへの悪影響を計測する
- `key` プロパティを意図的に変えてコンポーネントの状態をリセットするテクニックを実装する（例：ユーザー切り替え時にフォームをリセット）

### 2-2. Fiber の概念理解
- React 15 以前の同期レンダリング（Stack Reconciler）で大量のコンポーネントをレンダリングし、入力がブロックされることを再現する（意図的に重い処理を挟む）
- React 18 の `startTransition` で同じ処理をラップし、入力がブロックされなくなることを確認する
- `useDeferredValue` を使って検索結果の表示を遅延させ、入力のレスポンスを優先するパターンを実装する
- `startTransition` と `useDeferredValue` の使い分けを自分の言葉で説明する文章を書く

### 2-3. React のバッチ更新
- React 17 以前：イベントハンドラ内での `setState` 連続呼び出しがバッチされることを確認する → `setTimeout` 内では個別に反映されることを確認する
- React 18 以降：`setTimeout` / `Promise.then` 内でもバッチされるようになったことを確認する（Automatic Batching）
- `flushSync` でバッチを強制的に解除し、即座に DOM に反映されるケースを実験する

### 2-4. Suspense と並行レンダリング
- `React.lazy` + `Suspense` でコンポーネントの遅延読み込みを実装し、フォールバック UI が表示されることを確認する
- データフェッチを Suspense 対応にするパターン（Promise を throw する仕組み）を簡易実装して、Suspense の内部動作を理解する
- `SuspenseList`（実験的機能）で複数の非同期コンポーネントの表示順序を制御する実験をする

---

## 3. Next.js のアーキテクチャと実践

### 3-1. レンダリング戦略の理解と使い分け
- 同じページを 4 つのレンダリング戦略で実装し、ブラウザの Network タブとソースコードで出力の違いを確認する：
  - **SSG（Static Site Generation）**：`generateStaticParams` でビルド時に静的生成されることを確認する
  - **SSR（Server-Side Rendering）**：リクエストのたびにサーバーで HTML が生成されることを確認する（`cookies()` や `headers()` を使って動的レンダリングを強制）
  - **ISR（Incremental Static Regeneration）**：`revalidate` を設定し、指定秒数後にバックグラウンドで再生成されることを確認する
  - **CSR（Client-Side Rendering）**：`'use client'` + `useEffect` でクライアントのみでデータ取得するパターン
- 実際のユースケース（ブログ / EC / ダッシュボード / SNS）ごとに最適なレンダリング戦略を選ぶ判断フローチャートを自分で作る

### 3-2. Server Components と Client Components
- Server Component 内で `console.log` を書き、ターミナル（サーバー側）にだけ出力されることを確認する → Client Component では ブラウザの DevTools に出力されることを確認する
- Server Component 内で `useState` を使おうとしてエラーになることを確認する → なぜ状態を持てないのかを「サーバーで一度だけ実行される関数」という観点で説明する文章を書く
- Server Component から Client Component に props を渡す際、シリアライズ可能な値しか渡せないことを確認する（関数や Date オブジェクトを渡してエラーを出す）
- 「Server Component の中に Client Component を埋め込む」パターンと「Client Component の中に Server Component を children として渡す」パターンの違いを実装して理解する
- Client Component の境界（`'use client'` の配置場所）の設計判断について、実際のページ構成で練習する（ヘッダー / サイドバー / メインコンテンツ / フォーム）

### 3-3. データフェッチとキャッシュ
- Server Component 内で `fetch` を使い、Next.js がデフォルトでレスポンスをキャッシュすることを確認する（同じ URL への複数回 fetch が 1 回になる）
- `cache: 'no-store'` でキャッシュを無効化し、リクエストのたびにデータが再取得されることを確認する
- `next: { revalidate: 60 }` で時間ベースの再検証を設定し、キャッシュが更新されるタイミングを確認する
- `revalidatePath` / `revalidateTag` によるオンデマンド再検証を Server Action から呼び出す実装をする
- `React.cache` / `unstable_cache` を使ったデータベースクエリ結果のキャッシュを実装する

### 3-4. Server Actions
- フォームの送信処理を Server Action（`'use server'` 関数）で実装し、API Route を使わずにサーバー処理を実行する
- `useFormStatus` で送信中の状態を表示する実装をする
- `useActionState`（旧 `useFormState`）でフォーム送信後のバリデーションエラーを表示する実装をする
- Server Action 内でデータベース操作 → `revalidatePath` でページを再検証する一連のフローを実装する
- Server Action のプログレッシブエンハンスメント（JavaScript が無効でもフォームが動く）を確認する

### 3-5. ミドルウェアとルーティング
- `middleware.ts` でリクエストのパスに応じたリダイレクト処理を実装する
- ミドルウェアで Cookie / JWT を検証し、未認証ユーザーをログインページにリダイレクトする認証ガードを実装する
- ミドルウェアで `NextResponse.rewrite` を使い、A/B テストの振り分けを実装する
- `matcher` 設定でミドルウェアの適用範囲を制限する方法を確認する
- ルートグループ `(auth)` / `(public)` でレイアウトを分ける構成を実装する

### 3-6. 画像・フォント・メタデータ最適化
- `next/image` コンポーネントを使い、自動リサイズ・WebP 変換・lazy loading が適用されることを Network タブで確認する
- 通常の `<img>` タグとの CLS（Cumulative Layout Shift）の違いを Lighthouse で計測する
- `next/font` でカスタムフォント（Google Fonts）をセルフホスティングし、外部リクエストが発生しないことを確認する
- `generateMetadata` 関数で動的な OGP メタデータ（タイトル、説明、OG 画像）を生成する実装をする

### 3-7. エラーハンドリングと特殊ファイル
- `error.tsx` でルートレベルのエラーバウンダリを実装し、子コンポーネントのエラーをキャッチして回復 UI を表示する
- `not-found.tsx` でカスタム 404 ページを実装する
- `loading.tsx` で Suspense ベースのローディング UI を実装する
- `global-error.tsx` でルートレイアウトのエラーを捕捉する実装をする
- `route.ts` で API Route を作り、レスポンスのステータスコードとヘッダーを適切に設定する

---

## 4. TypeScript の型システム深層理解

### 4-1. 基礎型と型推論
- 明示的な型注釈なしでコードを書き、TypeScript の型推論がどこまで正確に型を特定するかを確認する（変数宣言 / 関数の戻り値 / 配列メソッドチェーン）
- `const` と `let` で推論される型の違い（リテラル型 vs ワイド型）を確認する（例：`const x = "hello"` は型が `"hello"`、`let x = "hello"` は型が `string`）
- `as const` アサーションで配列やオブジェクトをリテラル型として固定し、`readonly` なタプルが得られることを確認する

### 4-2. interface vs type の使い分け
- `interface` で宣言した型と `type` で宣言した型を同じ構造で書き、使えるケースと使えないケースを一覧にする
- `interface` の Declaration Merging（同名の interface を複数回宣言すると自動マージされる）を実験する → `type` ではエラーになることを確認する
- `type` でしかできない機能（ユニオン型 / マップ型 / 条件付き型など）を実験する
- チームで統一ルールを決めるための判断基準を自分の言葉でまとめる（「オブジェクトの形状定義は interface、それ以外は type」のような）

### 4-3. ユニオン型と Discriminated Union
- ユニオン型 `string | number` の値に対して `.toUpperCase()` を呼ぼうとしてエラーになることを確認する → 型の絞り込み（`typeof` チェック）で解消する
- Discriminated Union（判別可能なユニオン）で API レスポンスを型安全にモデル化する：
  ```typescript
  type Result =
    | { status: "success"; data: User }
    | { status: "error"; message: string }
    | { status: "loading" }
  ```
- `switch` 文で `status` を分岐し、各ケースで正しい型に絞り込まれることを確認する
- `never` 型を使った網羅性チェック（exhaustive check）を実装し、新しいステータスを追加したときにコンパイルエラーで検出できることを確認する

### 4-4. Generics（ジェネリクス）
- ジェネリック関数 `identity<T>(arg: T): T` を書き、呼び出し時に型が推論されることを確認する
- API レスポンスの汎用型を設計する：
  ```typescript
  type ApiResponse<T> = {
    data: T;
    status: number;
    timestamp: Date;
  }
  ```
- ジェネリック制約（`extends`）を使って「特定のプロパティを持つオブジェクトだけを受け取る関数」を実装する（例：`<T extends { id: string }>`）
- ジェネリクスを使った汎用的なリポジトリパターン（CRUD 操作）のインターフェースを設計する
- 複数の型パラメータ `<K, V>` を使った型安全な `Map` ラッパーを実装する

### 4-5. Utility Types の実践
- `Partial<T>` / `Required<T>` / `Readonly<T>` を実際のユースケースで使い分ける（例：更新 API のリクエストボディは `Partial<User>`）
- `Pick<T, K>` / `Omit<T, K>` でエンティティから必要なフィールドだけを抽出した DTO 型を作る
- `Record<K, V>` で辞書型のデータ構造を型安全に定義する
- `Exclude<T, U>` / `Extract<T, U>` でユニオン型からメンバーを除外 / 抽出する
- `ReturnType<T>` / `Parameters<T>` で既存関数の型情報を取得し、テストヘルパーの型定義に活用する
- `NonNullable<T>` で `null | undefined` を除外するパターンを実装する

### 4-6. 条件付き型（Conditional Types）
- `T extends string ? "text" : "other"` のような条件付き型を書き、型レベルの条件分岐を理解する
- `infer` キーワードで型を抽出するパターンを実装する（例：Promise の中身の型を取り出す `type Unwrap<T> = T extends Promise<infer U> ? U : T`）
- 配列の要素型を取り出す `type ElementOf<T> = T extends (infer E)[] ? E : never` を実装する
- 分配条件付き型（Distributive Conditional Types）の挙動を実験し、ユニオン型に対して条件付き型が各メンバーに分配されることを確認する

### 4-7. Template Literal Types と Mapped Types
- Template Literal Types で CSS のユーティリティ型を作る：`type Margin = \`margin-${"top" | "right" | "bottom" | "left"}\``
- Mapped Types で既存の型のすべてのプロパティをオプショナルにする `MyPartial<T>` を自作する（`Partial<T>` の再現）
- Key Remapping（`as` 句）を使って、プロパティ名を変換する Mapped Type を作る（例：全プロパティに `get` プレフィックスを付けた getter 型）
- Mapped Types + Conditional Types を組み合わせて、オブジェクト型から関数型のプロパティだけを抽出する型を作る

### 4-8. 型ガードとアサーション
- `typeof` / `instanceof` / `in` 演算子による型の絞り込みを実装する
- ユーザー定義型ガード（`function isUser(obj: unknown): obj is User`）を自作し、API レスポンスの型安全な検証に使う
- `asserts` キーワードを使ったアサーション関数（`function assertNonNull<T>(val: T): asserts val is NonNullable<T>`）を実装する
- `unknown` 型の値を安全に特定の型に絞り込む一連のバリデーション関数を書く

---

## 5. React × TypeScript の実践パターン

### 5-1. コンポーネントの型定義
- Props の型を `interface` で定義し、オプショナル props にデフォルト値を設定するパターンを実装する
- `children` の型（`React.ReactNode` vs `React.ReactElement` vs `string`）の違いを実験する
- イベントハンドラの型（`React.ChangeEvent<HTMLInputElement>` / `React.FormEvent<HTMLFormElement>` など）を正確に書く練習をする
- `React.ComponentPropsWithoutRef<"button">` を使って HTML 要素の props を継承するカスタムコンポーネントを作る

### 5-2. ジェネリックコンポーネント
- 汎用的なリストコンポーネント `<List<T> items={items} renderItem={(item) => ...} />` をジェネリクスで型安全に実装する
- 汎用的なテーブルコンポーネント `<Table<T> data={data} columns={columns} />` を実装する（各カラムの accessor が `T` のキーに制約される）
- ジェネリックな `useForm<T>` フックを実装し、フォームフィールド名とバリデーションルールが型レベルで検証されるようにする

### 5-3. 型安全な状態管理
- `useReducer` の action 型を Discriminated Union で定義し、`dispatch` で不正な payload を渡すとコンパイルエラーになることを確認する
- Zustand のストアを TypeScript で型安全に定義し、selector の戻り値型が自動推論されることを確認する
- Context の型定義で `null` チェックを強制するパターン（`createContext<T | null>(null)` + カスタムフック内で `null` チェック）を実装する

---

## 6. テストとアクセシビリティ

### 6-1. React Testing Library による テスト
- `render` / `screen` / `userEvent` を使って、ボタンクリックで状態が変わるコンポーネントのテストを書く
- `getByRole` / `getByLabelText` / `getByText` の使い分けを実践し、アクセシビリティに基づいたクエリの優先順位を理解する
- `waitFor` / `findBy` を使って非同期データ取得コンポーネントのテストを書く（API のモック込み）
- `msw`（Mock Service Worker）を使って API レスポンスをモックし、統合テストを書く
- カスタムフックのテストを `renderHook` で書く

### 6-2. アクセシビリティの実践
- セマンティック HTML（`<nav>` / `<main>` / `<article>` / `<section>` / `<button>` vs `<div>`）の使い分けをスクリーンリーダーで確認する
- ARIA 属性（`aria-label` / `aria-expanded` / `aria-hidden` / `role`）を適切に設定したモーダルダイアログを実装する
- キーボードナビゲーション（Tab / Enter / Escape / Arrow keys）が正しく動くドロップダウンメニューを実装する
- `axe-core` / `eslint-plugin-jsx-a11y` で自動検出できるアクセシビリティ違反を修正する練習をする
- フォーカストラップ（モーダル内にフォーカスを閉じ込める）を自作する

---

## 7. パフォーマンス最適化

### 7-1. バンドルサイズの分析と削減
- `next/bundle-analyzer` でバンドルサイズを可視化し、大きなライブラリを特定する
- 動的インポート（`next/dynamic` / `React.lazy`）でルートごとにコードを分割し、初期読み込みサイズの変化を計測する
- Tree Shaking が効かないケース（名前付きインポート vs デフォルトインポート / CommonJS vs ESM）を実験する
- 巨大なライブラリ（例：`lodash` → `lodash-es` or 個別インポート、`moment` → `date-fns`）の置き換えによるサイズ削減を計測する

### 7-2. ランタイムパフォーマンス
- `React.memo` / `useMemo` / `useCallback` を適切に使ったコンポーネントと使わないコンポーネントで Profiler の計測結果を比較する
- 仮想化（`react-window` / `react-virtuoso`）で 10000 件のリストをレンダリングし、仮想化なしとのスクロールパフォーマンスを比較する
- `useTransition` を使って重い状態更新を低優先度にし、入力のレスポンス性を改善する実装をする
- Web Vitals（LCP / FID / CLS）の計測方法と改善手法を Next.js アプリで実践する

---

## 8. ライブコーディング面接 模擬問題集

### 8-1. よく出るコンポーネント実装問題
- **デバウンス付き検索フォーム**：入力のたびに API を叩かず、300ms 待ってから検索する。ローディング / エラー / 結果の 3 状態を管理する
- **無限スクロール**：`IntersectionObserver` でページ末尾を検知し、次のページをフェッチして既存データに追加する
- **アクセシブルなモーダル**：ESC キーで閉じる / 背景クリックで閉じる / フォーカストラップ / スクリーンリーダー対応の ARIA 属性
- **ドラッグ＆ドロップで並べ替え可能なリスト**：`onDragStart` / `onDragOver` / `onDrop` を使って実装する
- **マルチステップフォーム**：ステップ間のナビゲーション / バリデーション / 最終確認画面 / 送信

### 8-2. 状態管理の設計問題
- **ショッピングカート**：商品追加 / 数量変更 / 削除 / 合計金額計算を `useReducer` で管理する
- **リアルタイムチャット UI**：メッセージ一覧 / 入力 / 送信 / 自動スクロール / タイピングインジケーター
- **通知システム**：トースト通知の表示 / 自動消去 / 手動閉じ / スタック管理

### 8-3. データ取得と表示の問題
- **データテーブル**：ソート（複数カラム） / フィルター / ページネーション / ローディング状態
- **オートコンプリート**：デバウンス / API 呼び出し / キーボードナビゲーション（上下キーで選択、Enter で確定）
- **ダッシュボード**：複数の API からデータを並行取得し、Suspense で段階的に表示する
