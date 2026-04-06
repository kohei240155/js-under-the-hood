## 1-4. コールスタックと実行コンテキスト — ハンズオン学習ガイド

> 関数を呼ぶと JavaScript エンジンの中で何が積まれ、何が作られ、いつ消えるのか。スタックトレースを読めるようになり、`this` やスコープの正体を「実行コンテキスト」という箱で説明できるようになるための教材。

---

## 0. この教材のねらい

完成時に以下ができるようになる：

- 関数呼び出しのたびにコールスタックへ「実行コンテキスト」が積まれるイメージを図で説明できる
- スタックトレース（エラーログ）を読んで、どの関数がどの順で呼ばれたか追える
- 実行コンテキストが持つ 3 つの要素（変数環境 / レキシカル環境 / `this` バインディング）を列挙できる
- 「スタックオーバーフロー」がなぜ起きるか、実際に起こしながら説明できる
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

**やってみよう（予測 → 検証）:**

> `second` の中で `third()` を呼ぶ前に `return` したら、スタックトレースはどう変わるでしょうか？予測してから試してください。

---

### Step 2: エラーでスタックトレースを読む

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
/.../stack-trace-error.js:4:27
    return { name: raw.name.toUpperCase() };
                          ^

TypeError: Cannot read properties of null (reading 'name')
    at parseUser (stack-trace-error.js:4:27)
    at loadUser (stack-trace-error.js:9:10)
    at renderProfile (stack-trace-error.js:13:16)
    at Object.<anonymous> (stack-trace-error.js:18:1)
```

**なぜこうなるか:**

> エラーが投げられた瞬間のコールスタックがそのまま `Error.stack` に記録される。**一番上の `at parseUser` が事故現場、下に行くほど「誰がその関数を呼んだか」の履歴**。デバッグの基本は「上から順に読んで、自分のコードが最初に出てきた行」を探すこと。

**やってみよう（予測 → 検証）:**

> `parseUser` の中で `if (!raw) throw new Error('raw is required')` と自分で投げた場合、スタックトレースの一番上は何になるでしょうか？

---

## 2. 実行コンテキストの中身をのぞく

### Step 3: 実行コンテキストが持つ 3 つのもの

実行コンテキストは、関数が動くために必要な「環境」をひとまとめにした箱。中には以下の 3 つが入っている：

1. **Variable Environment（変数環境）** — `var` で宣言された変数、関数宣言
2. **Lexical Environment（レキシカル環境）** — `let` / `const`、スコープチェーン（外側の環境への参照）
3. **ThisBinding** — その実行コンテキストでの `this` の値

実際にこれらが切り替わる様子を観察する：

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
    console.log('this:', this); // non-strict では globalThis、strict では undefined
  }

  inner();
}

outer();
```

**実行結果:**

```
innerName: INNER
outerName: OUTER
globalName: GLOBAL
this: <ref *1> Object [global] { ... }
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
> 変数を参照するとき、エンジンはこのチェーンを上から下へ辿る。見つかったらそこで止める（= シャドーイング）。

**React との比較:**

> React の Context API の `useContext` に似ている。子コンポーネントは自分に一番近い `Provider` の値を取る。JavaScript のスコープチェーンも「一番内側で最初に見つかった変数」を取る。違いは、React Context は **コンポーネントツリー（親子関係）** を辿るのに対し、JS スコープチェーンは **ソースコード上の入れ子（レキシカル構造）** を辿ること。実行時にどこから呼ばれたかは関係ない。

---

### Step 4: レキシカルスコープ vs 呼び出し元

この違いを体感するために、わざと「呼び出し元のスコープ」を期待させて裏切るコードを書く：

```js
// lexical-vs-dynamic.js
const name = 'GLOBAL';

function sayName() {
  // この関数は "定義された場所" の name を見る（レキシカル）
  console.log('name:', name);
}

function callerWithLocalName() {
  const name = 'LOCAL IN CALLER';
  sayName(); // ここで呼ばれても、sayName の中の name は GLOBAL
}

callerWithLocalName();
```

**実行結果:**

```
name: GLOBAL
```

**なぜこうなるか:**

> `sayName` のレキシカル環境は **定義されたとき** に決まる（関数の `[[Environment]]` 内部スロットに保存される）。誰が呼んだかは関係ない。これを **レキシカルスコープ（静的スコープ）** と呼ぶ。もし「呼び出し元の環境」を見に行くなら **動的スコープ**（Bash やレガシー言語の一部）だが、JavaScript はレキシカルスコープ。
>
> この性質が後で学ぶ **クロージャ** の土台になる。

**やってみよう（予測 → 検証）:**

> `sayName` を `callerWithLocalName` の中で **定義** したら結果はどうなるでしょうか？

---

## 3. スタックオーバーフローを起こす

### Step 5: 無限再帰でスタックを溢れさせる

```js
// stack-overflow.js
let depth = 0;

function recurse() {
  depth++;
  recurse(); // return しないので実行コンテキストが積まれ続ける
}

try {
  recurse();
} catch (e) {
  console.log('error name:', e.name);
  console.log('error message:', e.message);
  console.log('reached depth:', depth);
}
```

**実行結果（環境により数値は変わる）:**

```
error name: RangeError
error message: Maximum call stack size exceeded
reached depth: 13921
```

**なぜこうなるか:**

> コールスタックには有限のメモリしか割り当てられていない。関数が `return` しないまま呼び出され続けると、実行コンテキストがどんどん積まれてスタック容量を超え、`RangeError: Maximum call stack size exceeded` になる。これが **スタックオーバーフロー**（同名の有名サイトの由来）。
>
> 到達できる深さ（数千〜数万）はエンジン・OS・ローカル変数のサイズによって変わる。**これが「再帰で書くとクラッシュする」現象の正体**。末尾呼び出し最適化 (TCO) はスペック上存在するが、V8 は未実装なので Node.js では効かない。

**やってみよう（予測 → 検証）:**

> `recurse` の中で大きな配列 `const big = new Array(10000).fill(0)` を宣言してから再帰したら、到達できる `depth` は増える？減る？

---

## 4. 非同期とコールスタックの関係

### Step 6: setTimeout はスタックに積まれない

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

**React との比較:**

> React の `useEffect` の挙動に似ている。`useEffect(() => { ... })` の中のコードはレンダリングのコールスタックが空になってから実行される。「今のスタックが終わってから動く」という感覚は React 経験者には馴染みがあるはず。

**やってみよう（予測 → 検証）:**

> `setTimeout` の中からエラーを投げたら、スタックトレースに `async` 関数は出るでしょうか？予測してから試してください。

---

## 5. 理解チェック — 面接で聞かれたら

### Q1: コールスタックとは何か説明してください。

**模範回答（30 秒版）:**

> コールスタックは、実行中の関数呼び出しを記録する LIFO(後入れ先出し) のスタック構造です。関数を呼ぶたびに「実行コンテキスト」という箱が積まれ、`return` するとポップされます。JavaScript はシングルスレッドなので、このスタックは 1 本しかありません。
>
> **英語キーフレーズ:** "LIFO stack of execution contexts", "single-threaded", "pushed on call, popped on return"

**深掘りされた場合（1 分版）:**

> 実行コンテキストには、変数環境・レキシカル環境（スコープチェーン）・`this` バインディングが含まれます。スタックトレースはこのスタックの瞬間のスナップショットで、一番上が現在実行中の関数、下が呼び出し元の履歴です。スタック容量には上限があるため、無限再帰を書くと `RangeError: Maximum call stack size exceeded` が発生します。

---

### Q2: 実行コンテキストには何が含まれますか？

**模範回答（30 秒版）:**

> 変数環境（`var` と関数宣言）、レキシカル環境（`let`/`const` と外側環境への参照 = スコープチェーン）、そして `this` バインディングの 3 つです。
>
> **英語キーフレーズ:** "Variable Environment", "Lexical Environment", "this binding", "outer environment reference"

**深掘りされた場合（1 分版）:**

> レキシカル環境の「外側への参照」がスコープチェーンを形成します。変数を参照すると、エンジンはまず自分の環境を見て、なければ外側、さらに外側…とたどります。この「外側」は**関数が定義された場所**で決まり、呼び出し元ではありません。これがレキシカルスコープであり、クロージャの基礎です。

---

### Q3: なぜ `setTimeout` のコールバック内のエラーは、元の関数のスタックトレースに出ないのですか？

**模範回答（30 秒版）:**

> `setTimeout` のコールバックは元のコールスタックに積まれていないからです。タイマーは Web API に委ねられ、時間が経つとタスクキューに入り、イベントループが **空になったコールスタック** にあらためて積み直します。そのため呼び出し元の実行コンテキストはすでに消えています。
>
> **英語キーフレーズ:** "Web APIs", "task queue", "event loop", "fresh call stack", "original context already popped"

**深掘りされた場合（1 分版）:**

> この性質のため、非同期コードのデバッグは伝統的に難しいとされてきました。モダンな V8 や Chrome DevTools は「async stack traces」という機能で、Promise チェーンや `await` を越えた呼び出し履歴を再構築してくれます。`async`/`await` を使うと、構文上は同期的に見えつつ、内部的には Promise の resolve でマイクロタスクキュー経由で再開されます。

---

### Q4: スタックオーバーフローを避けるにはどうしますか？

**模範回答（30 秒版）:**

> 再帰をループに書き換えるのが王道です。また、末尾再帰にしてアキュムレータで状態を持つ、あるいは再帰的処理を `setTimeout` / `queueMicrotask` で分割してスタックをリセットする方法もあります。
>
> **英語キーフレーズ:** "convert recursion to iteration", "trampoline pattern", "defer with setTimeout to reset the stack"

**深掘りされた場合（1 分版）:**

> ECMAScript 仕様には Tail Call Optimization (TCO) が定義されていますが、V8（Chrome / Node.js）は実装していません。Safari の JavaScriptCore のみ実装されています。そのため実務では TCO に頼らず、反復やトランポリンパターン（再帰を関数を返す形にして、外側のループで呼ぶ）を使うのが安全です。
