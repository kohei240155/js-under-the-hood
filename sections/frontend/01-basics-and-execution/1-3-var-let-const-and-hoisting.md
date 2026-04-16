# 1-3. var / let / const とホイスティング — ハンズオン学習ガイド

> JavaScript の変数宣言キーワード `var` / `let` / `const` のスコープの違い、ホイスティング（巻き上げ）の仕組み、そして TDZ（Temporal Dead Zone）を手を動かして理解する。「なぜ `var` を使うべきでないのか？」を仕組みレベルで説明できるようになる。

---

## 0. この教材のねらい

**想定所要時間: 約 60 分**

- `var` / `let` / `const` のスコープの違い（関数スコープ vs ブロックスコープ）を即答できる
- ホイスティングが「宣言の巻き上げ」であり「初期化の巻き上げ」ではないことを説明できる
- TDZ（Temporal Dead Zone）の仕組みと、なぜ `let` / `const` で ReferenceError が出るかを理解する
- `for` ループ + `var` のクロージャ問題を理解し、`let` で解決できる理由を説明できる

---

## 1. var / let / const のスコープの違い

### Step 1: var と let/const のスコープを比較する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// var は関数スコープ — ブロック {} を無視する
function demonstrateVarScope() {
  if (true) {
    var message = "varで宣言しました";
  }
  console.log("ifブロックの外:", message); // アクセスできてしまう
}

demonstrateVarScope();

// let / const はブロックスコープ — {} に閉じ込められる
function demonstrateBlockScope() {
  if (true) {
    let blockLet = "letで宣言しました";
    const blockConst = "constで宣言しました";
    console.log("ifブロックの中 (let):", blockLet);
    console.log("ifブロックの中 (const):", blockConst);
  }

  try {
    console.log("ifブロックの外 (let):", blockLet);
  } catch (error) {
    console.log("letのエラー:", error.message);
  }
}

demonstrateBlockScope();
```

**実行結果:**

```
ifブロックの外: varで宣言しました
ifブロックの中 (let): letで宣言しました
ifブロックの中 (const): constで宣言しました
letのエラー: blockLet is not defined
```

**なぜこうなるか:**

> `var` は **関数スコープ** を持ち、`if` / `for` などのブロック `{}` を貫通する。一方、`let` / `const` は **ブロックスコープ** を持ち、`{}` の外に出ると変数は存在しないものとして扱われ、`ReferenceError` になる。

```
var のスコープ:                    let / const のスコープ:
┌─ function ──────────────┐       ┌─ function ────────────────┐
│  ┌─ if ──────────┐      │       │  ┌─ if ──────────────┐    │
│  │  var message   │     │       │  │  let blockLet     │    │
│  └────────────────┘     │       │  └───────────────────┘    │
│  console.log(message)OK │       │  console.log(blockLet)Err │
└──────────────────────────┘      └────────────────────────────┘
```

**React との比較:**

> React のコンポーネント内で `var` を使うとレンダリング関数全体にスコープが広がるため、意図しない変数の上書きが起きやすい。これが React プロジェクトで `let` / `const` のみを使う理由の一つ。

---

### Step 2: const の「再代入禁止」と「ミュータビリティ」を区別する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// const は「再代入」を禁止する。「中身の変更」は禁止しない
const userName = "太郎";

try {
  userName = "花子"; // 再代入を試みる
} catch (error) {
  console.log("再代入エラー:", error.message);
}

// オブジェクトの中身は変更できる
const userProfile = { name: "太郎", age: 25 };
userProfile.age = 26;
userProfile.email = "taro@example.com";
console.log("変更後:", userProfile);

// 配列の中身も変更できる
const scores = [80, 90, 70];
scores.push(100);
console.log("配列変更後:", scores);

// ただし、変数そのものの再代入はできない
try {
  userProfile = { name: "花子" };
} catch (error) {
  console.log("オブジェクト再代入エラー:", error.message);
}
```

**実行結果:**

```
再代入エラー: Assignment to constant variable.
変更後: { name: '太郎', age: 26, email: 'taro@example.com' }
配列変更後: [ 80, 90, 70, 100 ]
オブジェクト再代入エラー: Assignment to constant variable.
```

**なぜこうなるか:**

> `const` が守るのは「変数の束縛（バインディング）」であって「値そのもの」ではない。`const` は「この変数名は常にこのメモリアドレスを指す」ことを保証する。アドレスが同じまま中身を変えることは許される。

```
const userProfile = { name: "太郎" }
userProfile → [アドレス: 0x1234] → { name: "太郎" }

userProfile.name = "花子"  ← アドレスは同じ。OK!
userProfile = { name: "花子" }  ← アドレスが変わる。Error!
```

**React との比較:**

> React の `useState` で返る state も同じ考え方。`const [items, setItems] = useState([])` としたとき、`items.push(newItem)` は配列の中身を変えるが React は再レンダリングしない。React が変化を検知するには `setItems([...items, newItem])` のように新しい配列（新しいアドレス）を作る必要がある。

---

## 2. ホイスティングと TDZ

### Step 3: ホイスティングと TDZ をまとめて理解する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// var は宣言が巻き上げられ、undefined で初期化される
console.log("varの宣言前:", hoistedVar);
var hoistedVar = "varの値";
console.log("varの宣言後:", hoistedVar);

// let もホイスティングされるが、TDZ により宣言前のアクセスはエラー
try {
  console.log("letの宣言前:", hoistedLet);
} catch (error) {
  console.log("letのエラー:", error.message);
}
let hoistedLet = "letの値";
console.log("letの宣言後:", hoistedLet);

// 関数宣言は「まるごと」ホイスティングされる
console.log("関数宣言:", greet("太郎"));
function greet(name) {
  return `こんにちは、${name}さん！`;
}

// 関数式 (var) は undefined にホイスティング → TypeError
try {
  console.log("関数式(var):", greetVar("花子"));
} catch (error) {
  console.log("関数式(var)エラー:", error.message);
}
var greetVar = function(name) { return `こんにちは、${name}`; };

// アロー関数 (const) は TDZ → ReferenceError
try {
  console.log("アロー関数(const):", greetArrow("次郎"));
} catch (error) {
  console.log("アロー関数(const)エラー:", error.message);
}
const greetArrow = (name) => `こんにちは、${name}`;
```

**実行結果:**

```
varの宣言前: undefined
varの宣言後: varの値
letのエラー: Cannot access 'hoistedLet' before initialization
letの宣言後: letの値
関数宣言: こんにちは、太郎さん！
関数式(var)エラー: greetVar is not a function
アロー関数(const)エラー: Cannot access 'greetArrow' before initialization
```

**なぜこうなるか:**

> JavaScript エンジンはコードを実行する前に **コンパイルフェーズ** を行い、すべての宣言をスコープの先頭に「巻き上げ」る。ただし、巻き上げられるのは **宣言だけ** で、**代入は元の位置に残る**。
>
> `var` は宣言と同時に `undefined` で初期化されるが、`let` / `const` は **TDZ（Temporal Dead Zone：一時的デッドゾーン）** に入り、宣言行に到達するまでアクセスできない。これが「失敗体験」としての `ReferenceError: Cannot access ... before initialization` の正体。

```
スコープの先頭
│  ┌── TDZ（Temporal Dead Zone）──┐
│  │  ここでアクセス → ReferenceError  │
│  │  変数は「存在するが初期化されていない」│
│  └─────────────────────────────┘
│  let hoistedLet = "letの値";  ← ここで TDZ 終了
│  ここでアクセス → OK!
```

| 宣言方法 | ホイスティングされるもの | 宣言前のアクセス |
|---|---|---|
| `function greet()` | 関数全体（宣言＋本体） | 呼び出せる |
| `var greetVar = function()` | `var greetVar` のみ（値は `undefined`） | `undefined()` → TypeError |
| `const greetArrow = () =>` | 宣言のみ（TDZ に入る） | ReferenceError |

---

## 3. 実践的な落とし穴 — for ループ + var 問題

### Step 4: var + setTimeout のクロージャ問題を理解する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// var を使った for ループ — 予想外の結果になる
console.log("=== var版 ===");
for (var count = 0; count < 3; count++) {
  setTimeout(function() {
    console.log("var count:", count);
  }, 100);
}

// let を使った for ループ — 期待通りの結果
console.log("=== let版 ===");
for (let index = 0; index < 3; index++) {
  setTimeout(function() {
    console.log("let index:", index);
  }, 200);
}
```

**実行結果:**

```
=== var版 ===
=== let版 ===
var count: 3
var count: 3
var count: 3
let index: 0
let index: 1
let index: 2
```

**なぜこうなるか:**

> **var 版：** `var count` は関数スコープなので、ループ全体で **1 つの `count`** を共有する。`setTimeout` のコールバックが実行される頃にはループが終わっており、`count` は `3` になっている。
>
> **let 版：** `let index` はブロックスコープなので、ループの **各イテレーションごとに新しい `index`** が作られる。各コールバックが自分専用の `index` を持っているため、`0, 1, 2` が正しく出力される。

```
var 版:                          let 版:
┌──────────────────────┐         ┌──────────┐ ┌──────────┐ ┌──────────┐
│ count: 3（ループ後）   │        │ index: 0 │ │ index: 1 │ │ index: 2 │
├──────────────────────┤         ├──────────┤ ├──────────┤ ├──────────┤
│ callback 0 → count   │         │ callback │ │ callback │ │ callback │
│ callback 1 → count   │         └──────────┘ └──────────┘ └──────────┘
│ callback 2 → count   │
└──────────────────────┘
```

**ベストプラクティスまとめ:**

- `const` をデフォルトにする（再代入されないことを明示）
- 再代入が必要な場合のみ `let` を使う（ループカウンタなど）
- `var` は使わない（関数スコープ + ホイスティングでバグの温床）

---

## 発展課題（任意）

時間に余裕があれば挑戦してください。

### 課題 1: for ループのスコープ
> `for (var i = 0; i < 3; i++) {}` の後に `console.log(i)` するとどうなる？ `let j` に置き換えると？ 予測してから実行して理由を説明してみよう。

### 課題 2: TDZ はホイスティングの証拠
> 関数の中で外側に `let outerValue` がある状態で、内側で `console.log(innerValue); let innerValue = "内側";` と書くとどうなる？ ホイスティングがなければ外側を見に行くはず——実際は？

### 課題 3: IIFE で var 問題を解決
> `let` を使わず `(function(captured) { setTimeout(() => console.log(captured), 100); })(count)` で書き換えると `0, 1, 2` が出る。なぜか説明してみよう。

---

## 理解チェック — 面接で聞かれたら

### Q1: var / let / const の違いを説明してください

**模範回答（30 秒版）:**

> `var` は関数スコープで、宣言がホイスティングされて `undefined` で初期化されます。`let` と `const` はブロックスコープで、ホイスティングはされますが TDZ（Temporal Dead Zone）があるため宣言前にアクセスすると ReferenceError になります。`const` はさらに再代入が禁止されますが、オブジェクトのプロパティ変更は可能です。

### Q2: for ループで var を使うとなぜクロージャの問題が起きるのですか？

**模範回答（30 秒版）:**

> `var` は関数スコープなので、`for` ループのイテレーション全体で 1 つの変数を共有します。`setTimeout` などの非同期コールバックがその変数をキャプチャすると、実行時にはループが終了した後の最終値を参照してしまいます。`let` はブロックスコープなので各イテレーションが独自の変数を持ち、この問題は起きません。
