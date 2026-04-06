# 1-3. var / let / const とホイスティング — ハンズオン学習ガイド

> JavaScript の変数宣言キーワード `var` / `let` / `const` のスコープの違い、ホイスティング（巻き上げ）の仕組み、そして TDZ（Temporal Dead Zone）を手を動かして理解する。「なぜ `var` を使うべきでないのか？」を仕組みレベルで説明できるようになる。

---

## 0. この教材のねらい

- `var` / `let` / `const` のスコープの違い（関数スコープ vs ブロックスコープ）を即答できる
- ホイスティングが「宣言の巻き上げ」であり「初期化の巻き上げ」ではないことを説明できる
- TDZ（Temporal Dead Zone）の仕組みと、なぜ `let` / `const` で ReferenceError が出るかを理解する
- `for` ループ + `var` のクロージャ問題を理解し、`let` で解決できる理由を説明できる
- 面接でホイスティングに関する質問に自信を持って答えられる

---

## 1. var / let / const のスコープの違い

### Step 1: var のスコープを確認する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// var は関数スコープ — ブロック {} を無視する
function demonstrateVarScope() {
  if (true) {
    var message = "varで宣言しました";
  }
  // if ブロックの外からアクセスできるか？
  console.log("ifブロックの外:", message);
}

demonstrateVarScope();
```

**実行結果:**

```
ifブロックの外: varで宣言しました
```

**なぜこうなるか:**

> `var` は **関数スコープ** を持つ。つまり `var` で宣言した変数は、`if` / `for` / `while` などのブロック `{}` を貫通して、関数全体からアクセスできる。上の例では `message` は `demonstrateVarScope` 関数の中ならどこからでも見える。

**React との比較:**

> React のコンポーネント内で `var` を使うとレンダリング関数全体にスコープが広がるため、意図しない変数の上書きが起きやすい。これが React プロジェクトで `let` / `const` のみを使う理由の一つ。

---

### Step 2: let / const のブロックスコープを確認する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
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

  try {
    console.log("ifブロックの外 (const):", blockConst);
  } catch (error) {
    console.log("constのエラー:", error.message);
  }
}

demonstrateBlockScope();
```

**実行結果:**

```
ifブロックの中 (let): letで宣言しました
ifブロックの中 (const): constで宣言しました
letのエラー: blockLet is not defined
constのエラー: blockConst is not defined
```

**なぜこうなるか:**

> `let` と `const` は **ブロックスコープ** を持つ。`{}` で囲まれた範囲を出ると変数は存在しないものとして扱われる。ブロックの外からアクセスすると `ReferenceError` が発生する。

```
var のスコープ:
┌─ function ──────────────────┐
│  ┌─ if ──────────┐          │
│  │  var message   │ ← ここで宣言 │
│  └────────────────┘          │
│  console.log(message) ← OK! │
└──────────────────────────────┘

let / const のスコープ:
┌─ function ──────────────────────┐
│  ┌─ if ──────────────┐          │
│  │  let blockLet     │ ← ここだけ │
│  │  const blockConst │ ← ここだけ │
│  └───────────────────┘          │
│  console.log(blockLet) ← Error! │
└─────────────────────────────────┘
```

**やってみよう（予測 → 検証）:**

> `for` ループの中で `var` と `let` を使い分けるとどうなるでしょうか？以下のコードの出力を予測してから実行してみてください。
>
> ```js
> for (var i = 0; i < 3; i++) {
>   // ループ処理
> }
> console.log("varのi:", i);
>
> for (let j = 0; j < 3; j++) {
>   // ループ処理
> }
> try {
>   console.log("letのj:", j);
> } catch (error) {
>   console.log("letのjエラー:", error.message);
> }
> ```

---

### Step 3: const の「再代入禁止」を正確に理解する

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
console.log("変更前:", userProfile);

userProfile.age = 26; // プロパティの変更は OK
userProfile.email = "taro@example.com"; // プロパティの追加も OK
console.log("変更後:", userProfile);

// 配列の中身も変更できる
const scores = [80, 90, 70];
scores.push(100); // 要素の追加は OK
console.log("配列変更後:", scores);

// ただし、変数そのものの再代入はできない
try {
  userProfile = { name: "花子" }; // 別のオブジェクトを代入しようとするとエラー
} catch (error) {
  console.log("オブジェクト再代入エラー:", error.message);
}
```

**実行結果:**

```
再代入エラー: Assignment to constant variable.
変更前: { name: '太郎', age: 25 }
変更後: { name: '太郎', age: 26, email: 'taro@example.com' }
配列変更後: [ 80, 90, 70, 100 ]
オブジェクト再代入エラー: Assignment to constant variable.
```

**なぜこうなるか:**

> `const` が守るのは「変数の束縛（バインディング）」であって「値そのもの」ではない。つまり `const` は「この変数名は常にこのメモリアドレスを指す」ことを保証する。オブジェクトや配列の場合、アドレスが同じまま中身を変えることは許される。

```
const userProfile = { name: "太郎" }

userProfile → [アドレス: 0x1234] → { name: "太郎" }

userProfile.name = "花子"  ← アドレスは同じ。OK!
userProfile → [アドレス: 0x1234] → { name: "花子" }

userProfile = { name: "花子" }  ← アドレスが変わる。Error!
userProfile → [アドレス: 0x5678] ← 禁止！
```

**React との比較:**

> React の `useState` で返る state も同じ考え方。`const [items, setItems] = useState([])` としたとき、`items.push(newItem)` は配列の中身を変えるが React は再レンダリングしない。React が変化を検知するには `setItems([...items, newItem])` のように新しい配列（新しいアドレス）を作る必要がある。`const` の「バインディング vs 値」の理解はここに直結する。

---

## 2. ホイスティング（巻き上げ）の仕組み

### Step 4: var のホイスティングを体験する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// var は宣言がスコープの先頭に巻き上げられる
console.log("宣言前にアクセス:", hoistedVar);

var hoistedVar = "初期化された値";

console.log("宣言後にアクセス:", hoistedVar);
```

**実行結果:**

```
宣言前にアクセス: undefined
宣言後にアクセス: 初期化された値
```

**なぜこうなるか:**

> JavaScript エンジンはコードを実行する前に **コンパイルフェーズ** を行い、すべての `var` 宣言をスコープの先頭に「巻き上げ」る。ただし、巻き上げられるのは **宣言だけ** で、**代入は元の位置に残る**。つまりエンジンは上のコードを以下のように解釈している：

```js
// エンジンの解釈（イメージ）
var hoistedVar;                  // ← 宣言だけ巻き上げ。値は undefined
console.log("宣言前にアクセス:", hoistedVar); // undefined
hoistedVar = "初期化された値";      // ← 代入はここで実行
console.log("宣言後にアクセス:", hoistedVar); // "初期化された値"
```

---

### Step 5: let / const のホイスティングと TDZ を体験する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// let もホイスティングされるが、TDZ（Temporal Dead Zone）がある
try {
  console.log("letの宣言前:", hoistedLet);
} catch (error) {
  console.log("letのエラー:", error.message);
}

let hoistedLet = "letの値";
console.log("letの宣言後:", hoistedLet);
```

**実行結果:**

```
letのエラー: Cannot access 'hoistedLet' before initialization
constのエラー: Cannot access 'hoistedConst' before initialization
letの宣言後: letの値
```

> ※ 実際には上のコードでは `const` のテストは含まれていません。正しい出力は：

```
letのエラー: Cannot access 'hoistedLet' before initialization
letの宣言後: letの値
```

**なぜこうなるか:**

> `let` / `const` も実は **ホイスティングされる**（宣言はスコープの先頭で認識される）。しかし `var` と違い、宣言位置に到達するまで **TDZ（Temporal Dead Zone：一時的デッドゾーン）** に入る。TDZ 内でアクセスすると `ReferenceError` が発生する。

```
スコープの先頭
│  ┌── TDZ（Temporal Dead Zone）──┐
│  │  ここでアクセス → ReferenceError  │
│  │  変数は「存在するが初期化されていない」│
│  └─────────────────────────────┘
│  let hoistedLet = "letの値";  ← ここで TDZ 終了
│  ここでアクセス → OK!
│
スコープの終わり
```

**やってみよう（予測 → 検証）:**

> TDZ はホイスティングの証拠です。もし `let` がホイスティングされなければ、以下のコードはどう動くでしょうか？予測してから実行してみてください。
>
> ```js
> let outerValue = "外側";
>
> function checkTDZ() {
>   // もし let がホイスティングされないなら、
>   // ここでは outerValue（外側）が見えるはず
>   try {
>     console.log("innerValueの値:", innerValue);
>   } catch (error) {
>     console.log("エラー:", error.message);
>   }
>   let innerValue = "内側";
> }
>
> checkTDZ();
> ```
>
> ヒント：`let innerValue` がホイスティングされるなら、`innerValue` は TDZ に入り `ReferenceError` になる。ホイスティングされないなら、スコープチェーンをたどって外側の変数を探しに行く（ただしこの例では `outerValue` と `innerValue` は別名なので、単純に `not defined` になる）。

---

### Step 6: 関数宣言のホイスティングと関数式の違いを確認する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// 関数宣言は「まるごと」ホイスティングされる
console.log("関数宣言の結果:", greet("太郎"));

function greet(name) {
  return `こんにちは、${name}さん！`;
}

// 関数式は変数のホイスティングルールに従う
try {
  console.log("関数式(var)の結果:", greetVar("花子"));
} catch (error) {
  console.log("関数式(var)のエラー:", error.message);
}

var greetVar = function(name) {
  return `こんにちは、${name}さん！`;
};

// アロー関数も関数式と同じ
try {
  console.log("アロー関数(const)の結果:", greetArrow("次郎"));
} catch (error) {
  console.log("アロー関数(const)のエラー:", error.message);
}

const greetArrow = (name) => `こんにちは、${name}さん！`;
```

**実行結果:**

```
関数宣言の結果: こんにちは、太郎さん！
関数式(var)のエラー: greetVar is not a function
アロー関数(const)のエラー: Cannot access 'greetArrow' before initialization
```

**なぜこうなるか:**

> | 宣言方法 | ホイスティングされるもの | 宣言前のアクセス |
> |---|---|---|
> | `function greet()` | 関数全体（宣言＋本体） | 呼び出せる |
> | `var greetVar = function()` | `var greetVar` のみ（値は `undefined`） | `undefined()` → TypeError |
> | `const greetArrow = () =>` | 宣言のみ（TDZ に入る） | ReferenceError |

> `var greetVar` は `undefined` にホイスティングされるため、`undefined()` を呼ぼうとして「is not a function」エラーになる。`const greetArrow` は TDZ 内なので「before initialization」エラーになる。エラーメッセージが違うことに注目。

---

## 3. 実践的な落とし穴 — for ループ + var 問題

### Step 7: var + setTimeout のクロージャ問題を体験する

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

> **var 版：** `var count` は関数スコープなので、ループ全体で **1 つの `count`** を共有する。`setTimeout` のコールバックが実行される頃にはループが終わっており、`count` は `3` になっている。3 つのコールバックが同じ `count`（= 3）を参照するため、全部 `3` が出力される。
>
> **let 版：** `let index` はブロックスコープなので、ループの **各イテレーションごとに新しい `index`** が作られる。各コールバックが自分専用の `index` を持っているため、`0, 1, 2` が正しく出力される。

```
var 版のメモリイメージ:
┌──────────────────────┐
│ count: 3（ループ後）   │ ← 全コールバックがこの 1 つを共有
├──────────────────────┤
│ callback 0 → count   │
│ callback 1 → count   │
│ callback 2 → count   │
└──────────────────────┘

let 版のメモリイメージ:
┌──────────┐ ┌──────────┐ ┌──────────┐
│ index: 0 │ │ index: 1 │ │ index: 2 │
├──────────┤ ├──────────┤ ├──────────┤
│ callback │ │ callback │ │ callback │
└──────────┘ └──────────┘ └──────────┘
↑ 各イテレーションが独自の index を持つ
```

**React との比較:**

> React でイベントハンドラをループ内で定義するときも同じ問題が起きうる。ただし React の `map` を使う場合、コールバック関数の引数として値が渡されるため `var` 問題は起きにくい。それでも原理を理解していないと、カスタムフック内などで同様のバグを作り込む可能性がある。

**やってみよう（予測 → 検証）:**

> `var` 版を `let` を使わずに修正する古典的な方法があります。以下のコードの出力を予測してから実行してみてください。
>
> ```js
> // IIFE（即時実行関数式）でスコープを作る方法
> for (var count = 0; count < 3; count++) {
>   (function(capturedCount) {
>     setTimeout(function() {
>       console.log("IIFE count:", capturedCount);
>     }, 100);
>   })(count);
> }
> ```
>
> ヒント：IIFE は `var` しかなかった時代に「ブロックスコープ」を擬似的に作るテクニックだった。

---

## 4. 宣言スタイルのベストプラクティス

### Step 8: 現代の変数宣言ルールを整理する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// 現代の JavaScript では const をデフォルトで使う
// 再代入が必要な場合のみ let を使う
// var は使わない

// ✅ const をデフォルトにする
const API_BASE_URL = "https://api.example.com";
const maxRetries = 3;
const userList = []; // 配列の中身は変えられる

// ✅ 再代入が必要な場合は let
let currentPage = 1;
let isLoading = false;
let errorMessage = null;

// ループカウンタも let
for (let step = 0; step < maxRetries; step++) {
  console.log(`リトライ ${step + 1} / ${maxRetries}`);
}

// 条件で値が変わる場合も let（ただし三項演算子で const にできるならそちらを優先）
const statusCode = 404;

// ❌ let を使わなくてもよいパターン
let statusText;
if (statusCode === 200) {
  statusText = "OK";
} else if (statusCode === 404) {
  statusText = "Not Found";
} else {
  statusText = "Unknown";
}
console.log("letパターン:", statusText);

// ✅ const + 三項演算子やオブジェクトマップでまとめる
const statusMap = { 200: "OK", 404: "Not Found" };
const betterStatusText = statusMap[statusCode] || "Unknown";
console.log("constパターン:", betterStatusText);
```

**実行結果:**

```
リトライ 1 / 3
リトライ 2 / 3
リトライ 3 / 3
letパターン: Not Found
constパターン: Not Found
```

**なぜこうなるか:**

> `const` をデフォルトにすることで「この変数は再代入されない」という意図を明示できる。コードを読む人は `let` を見た瞬間「この変数はどこかで値が変わるんだな」と身構えることができる。`var` は関数スコープ + ホイスティングで予期しないバグを生みやすいため、現代の JavaScript では使わない。

> **判断フローチャート:**
> ```
> 変数を宣言する
> │
> ├── 再代入する？ ─── No ──→ const を使う
> │
> └── Yes ──→ let を使う
>
> ※ var は使わない
> ```

---

## 理解チェック — 面接で聞かれたら

### Q1: var / let / const の違いを説明してください

**模範回答（30 秒版）:**

> `var` は関数スコープで、宣言がホイスティングされて `undefined` で初期化されます。`let` と `const` はブロックスコープで、ホイスティングはされますが TDZ（Temporal Dead Zone）があるため宣言前にアクセスすると ReferenceError になります。`const` はさらに再代入が禁止されますが、オブジェクトのプロパティ変更は可能です。

**英語キーフレーズ:**
> "var is function-scoped, let and const are block-scoped. All three are hoisted, but let and const have a Temporal Dead Zone. const prevents reassignment but not mutation."

**深掘りされた場合（1 分版）:**

> ホイスティングの仕組みをもう少し詳しく言うと、JavaScript エンジンはコードを実行する前にコンパイルフェーズで変数宣言をスコープに登録します。`var` の場合は同時に `undefined` で初期化されますが、`let` / `const` は初期化が実際の宣言行まで遅延されます。この「宣言は認識されているが初期化されていない」期間が TDZ です。TDZ があることで、宣言前のアクセスがエラーになり、バグの早期発見につながります。実務では `const` をデフォルトにし、再代入が必要な場合のみ `let` を使い、`var` は使わないのがモダンなスタイルです。

### Q2: ホイスティングとは何ですか？ let もホイスティングされますか？

**模範回答（30 秒版）:**

> ホイスティングとは、変数や関数の宣言がスコープの先頭に巻き上げられる JavaScript の仕組みです。`var` は宣言と同時に `undefined` で初期化されるため、宣言前に `undefined` としてアクセスできます。`let` / `const` もホイスティングされますが、TDZ により宣言行まで初期化されないため、宣言前にアクセスすると ReferenceError になります。

**英語キーフレーズ:**
> "Hoisting moves declarations to the top of their scope during the compile phase. var is initialized to undefined, while let/const enter the Temporal Dead Zone until the declaration is reached."

### Q3: for ループで var を使うとなぜクロージャの問題が起きるのですか？

**模範回答（30 秒版）:**

> `var` は関数スコープなので、`for` ループのイテレーション全体で 1 つの変数を共有します。`setTimeout` などの非同期コールバックがその変数をキャプチャすると、実行時にはループが終了した後の最終値を参照してしまいます。`let` はブロックスコープなので各イテレーションが独自の変数を持ち、この問題は起きません。

**英語キーフレーズ:**
> "var creates a single binding shared across all iterations. let creates a new binding per iteration, so each closure captures its own copy of the variable."

**深掘りされた場合（1 分版）:**

> ES6 以前は `var` しかなかったため、IIFE（即時実行関数式）でスコープを作って回避していました。`(function(i) { setTimeout(..., i); })(count)` のように関数呼び出しで値を引数としてコピーします。`let` の登場でこのパターンは不要になりましたが、レガシーコードの保守ではまだ見かけることがあります。この問題は JavaScript のクロージャとスコープの理解を試す典型的な面接質問です。
