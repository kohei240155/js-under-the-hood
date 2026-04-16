## 1-4. コールスタックと実行コンテキスト — ハンズオン学習ガイド

> 関数を呼ぶと JavaScript エンジンの中で何が積まれ、何が作られ、いつ消えるのか。スタックトレースを読めるようになり、`this` やスコープの正体を「実行コンテキスト」という箱で説明できるようになるための教材。

---

## 0. この教材のねらい

**想定所要時間: 約 60 分**

完成時に以下ができるようになる：

- 関数呼び出しのたびにコールスタックへ「実行コンテキスト」が積まれるイメージを図で説明できる
- スタックトレース（エラーログ）を読んで、どの関数がどの順で呼ばれたか追える
- 実行コンテキストが持つ 3 つの要素（変数環境 / レキシカル環境 / `this` バインディング）を列挙できる
- なぜ非同期処理（`setTimeout` など）はコールスタックに積まれたままにならないのかを言葉で説明できる

---

## 1. コールスタックを可視化する

### Step 1: 関数の呼び出し順をスタックで追う

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// call-stack-basic.js
function third() {
  console.log('step: third() 実行中');
  console.trace('現在のコールスタック'); // スタックの中身を出力
}

function second() {
  console.log('step: second() 実行中');
  third();
  console.log('step: second() 再開(third から戻った)');
}

function first() {
  console.log('step: first() 実行中');
  second();
  console.log('step: first() 再開(second から戻った)');
}

console.log('step: main 開始');
first();
console.log('step: main 終了');
```

`node call-stack-basic.js` で実行してください。

**実行結果（抜粋）:**

```
step: main 開始
step: first() 実行中
step: second() 実行中
step: third() 実行中
Trace: 現在のコールスタック
    at third (call-stack-basic.js:4:11)
    at second (call-stack-basic.js:9:3)
    at first (call-stack-basic.js:15:3)
    at Object.<anonymous> (call-stack-basic.js:20:1)
step: second() 再開(third から戻った)
step: first() 再開(second から戻った)
step: main 終了
```

**なぜこうなるか:**

> 関数を呼ぶたびに、JavaScript エンジンは「実行コンテキスト」という箱を作り、**コールスタック**という LIFO(後入れ先出し) のスタックに積む。関数が `return` するとその箱はポップされて消える。
>
> ```
> 時間 →
>                        [third]
>              [second]  [second]  [second]
>    [first]   [first]   [first]   [first]   [first]
>   [<main>]  [<main>]  [<main>]  [<main>]  [<main>]  [<main>]
>    ↑ first   ↑ second  ↑ third  ↑ pop     ↑ pop     ↑ pop
>      push     push      push     third     second    first
> ```
>
> `console.trace` が出すスタックトレースは、**その瞬間に積まれている箱を上から順に並べたもの**。一番上が今実行中の関数、下に行くほど呼び出し元。

**React との比較:**

> React のコンポーネントツリーが「親 → 子 → 孫」と描画されるのに似ているが、コールスタックは **実行時の時間軸** のツリー。React DevTools のコンポーネントツリーが静的な親子関係なのに対し、コールスタックは「今この瞬間、誰が誰を呼んでいるか」の動的なスナップショット。エラー画面で見る `at Component` のスタックトレースはまさにこれ。

---

### Step 2: スタックトレースを読んでデバッグする

```js
// stack-trace-error.js
function parseUser(raw) {
  // わざとエラーを起こす: raw が null のとき .name にアクセスできない
  return { name: raw.name.toUpperCase() };
}

function loadUser(id) {
  const raw = id === 1 ? { name: 'alice' } : null;
  return parseUser(raw);
}

function renderProfile(id) {
  const user = loadUser(id);
  console.log('profile:', user);
}

renderProfile(1); // 成功
renderProfile(2); // 失敗
```

**実行結果:**

```
profile: { name: 'ALICE' }

TypeError: Cannot read properties of null (reading 'name')
    at parseUser (stack-trace-error.js:4:27)
    at loadUser (stack-trace-error.js:9:10)
    at renderProfile (stack-trace-error.js:13:16)
    at Object.<anonymous> (stack-trace-error.js:18:1)
```

**なぜこうなるか:**

> エラーが投げられた瞬間のコールスタックがそのまま `Error.stack` に記録される。**一番上の `at parseUser` が事故現場、下に行くほど「誰がその関数を呼んだか」の履歴**。デバッグの基本は「上から順に読んで、自分のコードが最初に出てきた行」を探すこと。

---

## 2. 実行コンテキストの中身

### Step 3: 実行コンテキストとレキシカルスコープを理解する

実行コンテキストは、関数が動くために必要な「環境」をひとまとめにした箱。中には以下の 3 つが入っている：

1. **Variable Environment（変数環境）** — `var` で宣言された変数、関数宣言
2. **Lexical Environment（レキシカル環境）** — `let` / `const`、スコープチェーン（外側の環境への参照）
3. **ThisBinding** — その実行コンテキストでの `this` の値

実際に観察する：

```js
// execution-context.js
const globalName = 'GLOBAL';

function outer() {
  const outerName = 'OUTER';

  function inner() {
    const innerName = 'INNER';
    // この瞬間のスコープチェーン: inner → outer → global
    console.log('innerName:', innerName);
    console.log('outerName:', outerName); // 外側の環境を参照
    console.log('globalName:', globalName); // さらに外側を参照
  }

  inner();
}

outer();

// レキシカルスコープの確認: "定義された場所" の name を見る
const name = 'GLOBAL';

function sayName() {
  console.log('name:', name);
}

function callerWithLocalName() {
  const name = 'LOCAL IN CALLER';
  sayName(); // sayName は呼び出し元の name ではなく、定義時の GLOBAL を見る
}

callerWithLocalName();
```

**実行結果:**

```
innerName: INNER
outerName: OUTER
globalName: GLOBAL
name: GLOBAL
```

**なぜこうなるか:**

> `inner` の実行コンテキストが作られるとき、エンジンは「自分の変数環境」+「外側（`outer`）のレキシカル環境への参照」を記録する。これが **スコープチェーン** の正体。
>
> ```
> [inner EC]
>   variables: { innerName: 'INNER' }
>   outer ─────┐
>              ↓
>          [outer EC]
>            variables: { outerName: 'OUTER' }
>            outer ─────┐
>                       ↓
>                   [global EC]
>                     variables: { globalName: 'GLOBAL' }
>                     outer: null
> ```
>
> 関数の `[[Environment]]` 内部スロットは **定義されたとき** に決まる。誰が呼んだかは関係ない。これを **レキシカルスコープ（静的スコープ）** と呼ぶ。`sayName` の中の `name` が `callerWithLocalName` のローカルではなくグローバルを見るのはこのため。この性質が後で学ぶ **クロージャ** の土台になる。

**React との比較:**

> React の Context API の `useContext` に似ている。子コンポーネントは自分に一番近い `Provider` の値を取る。JS スコープチェーンも「一番内側で最初に見つかった変数」を取る。違いは、React Context は **コンポーネントツリー（親子関係）** を辿るのに対し、JS スコープチェーンは **ソースコード上の入れ子（レキシカル構造）** を辿ること。

---

## 3. 非同期とコールスタックの関係

### Step 4: setTimeout はスタックに積まれない

```js
// async-and-stack.js
function sync() {
  console.log('[sync] start');
  console.log('[sync] end');
}

function async() {
  console.log('[async] start');
  setTimeout(() => {
    console.log('[async] timer callback');
    console.trace('timer callback stack');
  }, 0);
  console.log('[async] end (timer はまだ実行されていない)');
}

console.log('--- main start ---');
sync();
async();
console.log('--- main end ---');
```

**実行結果:**

```
--- main start ---
[sync] start
[sync] end
[async] start
[async] end (timer はまだ実行されていない)
--- main end ---
[async] timer callback
Trace: timer callback stack
    at Timeout._onTimeout (async-and-stack.js:9:13)
    at listOnTimeout (node:internal/timers:...)
    ...
```

**なぜこうなるか:**

> `setTimeout` のコールバックは **コールスタックに直接積まれない**。ブラウザ/Node の Web API（タイマー機構）に渡され、時間が経つとタスクキューに入り、**コールスタックが空になったあと** にイベントループがキューから取り出して積み直す。
>
> 注目ポイント：コールバック実行時のスタックトレースに `async` 関数が **いない**。つまり呼び出し元のコンテキストは既に消えている。ここで保持されている変数は「クロージャ」経由のものだけ。これがイベントループ編（次の章）への橋渡し。

> **コラム: スタックオーバーフロー**
> コールスタックには有限のメモリしか割り当てられていない。`function recurse() { recurse(); } recurse();` のように `return` しないまま呼び出され続けると、`RangeError: Maximum call stack size exceeded` になる（数千〜数万段で限界）。これが「再帰で書くとクラッシュする」現象の正体。回避策は、再帰をループに書き換える、または `setTimeout` でスタックを一度リセットすること。詳しくは発展課題で。

---

## 4. 発展課題（任意）

時間に余裕があれば挑戦してください。

### 課題 1: 自分でエラーを投げたときのスタックトレース
> Step 2 の `parseUser` の中で `if (!raw) throw new Error('raw is required')` と自分で投げた場合、スタックトレースの一番上は何になるでしょうか？予測してから試してみてください。

### 課題 2: スタックオーバーフローを起こしてみる
> 以下を `node` で実行し、到達できる `depth` を確認してください。さらに `recurse` の中で `const big = new Array(10000).fill(0)` を宣言してから再帰したら、`depth` は増える？減る？
>
> ```js
> let depth = 0;
> function recurse() { depth++; recurse(); }
> try { recurse(); } catch (e) { console.log('depth:', depth); }
> ```

### 課題 3: setTimeout からエラーを投げる
> Step 4 の `setTimeout` の中で `throw new Error('boom')` を実行したら、スタックトレースに `async` 関数は出るでしょうか？予測してから試してください。

---

## 5. 理解チェック — 面接で聞かれたら

### Q1: コールスタックとは何か説明してください。

**模範回答（30 秒版）:**

> コールスタックは、実行中の関数呼び出しを記録する LIFO(後入れ先出し) のスタック構造です。関数を呼ぶたびに「実行コンテキスト」という箱が積まれ、`return` するとポップされます。JavaScript はシングルスレッドなので、このスタックは 1 本しかありません。スタックトレースはこのスタックの瞬間のスナップショットで、一番上が現在実行中の関数、下が呼び出し元の履歴です。

---

### Q2: 実行コンテキストには何が含まれますか？

**模範回答（30 秒版）:**

> 変数環境（`var` と関数宣言）、レキシカル環境（`let`/`const` と外側環境への参照 = スコープチェーン）、そして `this` バインディングの 3 つです。レキシカル環境の「外側への参照」は **関数が定義された場所** で決まり、呼び出し元ではありません。これがレキシカルスコープであり、クロージャの基礎になります。
