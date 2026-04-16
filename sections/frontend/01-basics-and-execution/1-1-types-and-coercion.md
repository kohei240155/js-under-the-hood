# 1-1. 型と型変換の罠 — ハンズオン学習ガイド

> JavaScript の 7 つのプリミティブ型、`typeof` の罠、`==` vs `===` の挙動、明示的型変換の境界ケースを手を動かして理解する。面接で「なぜ `typeof null` は `"object"` なのか？」と聞かれても自信を持って答えられるようになる。

---

## 0. この教材のねらい

**想定所要時間: 約 60 分**

- `typeof` で全プリミティブ型の判定結果を即答できるようになる
- `==` と `===` の違いを「暗黙の型変換が起きるかどうか」で説明できる
- `Number()` / `parseInt()` / `String()` / `Boolean()` の境界ケースを把握する
- 型変換が原因のバグを見抜けるようになる

---

## 1. typeof で全プリミティブ型を確認する

### Step 1: 7 つのプリミティブ型の typeof 結果を確認する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// JavaScript の 7 つのプリミティブ型を typeof で確認する
const stringValue = "hello";
const numberValue = 42;
const bigintValue = 9007199254740991n;
const booleanValue = true;
const undefinedValue = undefined;
const symbolValue = Symbol("id");
const nullValue = null;

console.log("typeof string:", typeof stringValue);
console.log("typeof number:", typeof numberValue);
console.log("typeof bigint:", typeof bigintValue);
console.log("typeof boolean:", typeof booleanValue);
console.log("typeof undefined:", typeof undefinedValue);
console.log("typeof symbol:", typeof symbolValue);
console.log("typeof null:", typeof nullValue);
```

**実行結果:**

```
typeof string: string
typeof number: number
typeof bigint: bigint
typeof boolean: boolean
typeof undefined: undefined
typeof symbol: symbol
typeof null: object
```

**なぜこうなるか（失敗体験を読み解く）:**

> 6 つは型名がそのまま返る。しかし `typeof null` だけは `"object"` を返す。これは JavaScript 初期の実装バグが仕様として残ったもの。初期の JS エンジンでは値を「型タグ + 実データ」のビット列で表現しており、オブジェクトの型タグは `000` だった。`null` は「ヌルポインタ = 全ビット 0」で表現されていたため、型タグ部分も `000` = オブジェクトと判定されてしまった。
>
> 実用上は **null チェックには `value === null`**、より厳密な型判定には `Object.prototype.toString.call(value)`（`"[object Null]"` を返す）を使う。

**React との比較:**

> React + TypeScript のプロジェクトでは、型の判定はコンパイル時に TypeScript が行う。しかしランタイムで API レスポンスの型を検証する場合（例：`if (data === null)` のガード）は、ここで学んだ知識がそのまま必要になる。

---

## 2. == と === の暗黙の型変換を実験する

### Step 2: == と === の違いを実験する

まず、以下のコードを **そのまま写経して** 実行してください。実行前に各行の結果を予測してみましょう。

```js
// == （緩い比較）の暗黙の型変換 5 パターン
const patterns = [
  { left: "0",       right: false,     label: '"0" == false' },
  { left: "",        right: 0,         label: '"" == 0' },
  { left: null,      right: undefined, label: 'null == undefined' },
  { left: null,      right: 0,         label: 'null == 0' },
  { left: [],        right: 0,         label: '[] == 0' },
];

console.log("=== == の結果 ===");
patterns.forEach(({ left, right, label }) => {
  console.log(`${label.padEnd(22)} → ${left == right}`);
});

console.log("\n=== === の結果 ===");
patterns.forEach(({ left, right, label }) => {
  console.log(`${label.replace('==', '===').padEnd(24)} → ${left === right}`);
});
```

**実行結果:**

```
=== == の結果 ===
"0" == false           → true
"" == 0                → true
null == undefined      → true
null == 0              → false
[] == 0                → true

=== === の結果 ===
"0" === false            → false
"" === 0                 → false
null === undefined       → false
null === 0               → false
[] === 0                 → false
```

**なぜこうなるか:**

> `==` は比較前に **Abstract Equality Comparison** アルゴリズムで型変換を行う。主なルール：
>
> ```
> 1. string vs number → string を Number() で変換
> 2. boolean vs 他    → boolean を Number() で変換（true→1, false→0）してから再比較
> 3. object vs primitive → object の valueOf() → toString() を試す（ToPrimitive）
> 4. null == undefined → true（特別ルール）
> 5. null/undefined は他の型と == で比較すると false
> ```
>
> たとえば `"0" == false` は: `false → 0` → `"0" == 0` → `0 == 0` → `true`
>
> `[] == 0` は: `[].toString() → ""` → `"" == 0` → `0 == 0` → `true`
>
> 一方 `===` は型変換を一切行わない。型が違えば即座に `false`。これが「常に `===` を使うべき」と言われる理由。

**React との比較:**

> React の `useState` で状態を更新するとき、React は前後の値を `Object.is()` で比較して再レンダリングの要否を判定する。`Object.is()` は `===` とほぼ同じだが、`NaN === NaN` が `false` なのに対し `Object.is(NaN, NaN)` は `true` を返す点が異なる。

---

## 3. 明示的型変換の境界ケース

### Step 3: Number / parseInt / Boolean の境界ケースを掴む

```js
// Number() の境界ケース
console.log("=== Number() の変換 ===");
console.log("Number(''):", Number(""));            // 0 — 罠！
console.log("Number('  '):", Number("  "));        // 0 — 罠！
console.log("Number('42'):", Number("42"));        // 42
console.log("Number('42abc'):", Number("42abc"));  // NaN
console.log("Number(null):", Number(null));        // 0
console.log("Number(undefined):", Number(undefined)); // NaN
console.log("Number([]):", Number([]));            // 0
console.log("Number([1]):", Number([1]));          // 1
console.log("Number([1,2]):", Number([1,2]));      // NaN

// parseInt() は途中まで読む
console.log("\n=== parseInt() の変換 ===");
console.log("parseInt(''):", parseInt(""));        // NaN — Number と違う！
console.log("parseInt('42abc'):", parseInt("42abc")); // 42
console.log("parseInt('abc42'):", parseInt("abc42")); // NaN
console.log("parseInt('3.14'):", parseInt("3.14")); // 3

// Boolean() の falsy 値 — これだけ覚える
console.log("\n=== falsy 値（8 種類） ===");
console.log("Boolean(false):", Boolean(false));         // false
console.log("Boolean(0):", Boolean(0));                 // false
console.log("Boolean(-0):", Boolean(-0));               // false
console.log('Boolean(""):', Boolean(""));               // false
console.log("Boolean(null):", Boolean(null));           // false
console.log("Boolean(undefined):", Boolean(undefined)); // false
console.log("Boolean(NaN):", Boolean(NaN));             // false
console.log("Boolean(0n):", Boolean(0n));               // false

// 驚きの truthy 値
console.log("\n=== 驚きの truthy 値 ===");
console.log('Boolean("0"):', Boolean("0"));         // true — 罠！
console.log('Boolean("false"):', Boolean("false")); // true
console.log("Boolean([]):", Boolean([]));           // true
console.log("Boolean({}):", Boolean({}));           // true
```

**実行結果:**

```
=== Number() の変換 ===
Number(''): 0
Number('  '): 0
Number('42'): 42
Number('42abc'): NaN
Number(null): 0
Number(undefined): NaN
Number([]): 0
Number([1]): 1
Number([1,2]): NaN

=== parseInt() の変換 ===
parseInt(''): NaN
parseInt('42abc'): 42
parseInt('abc42'): NaN
parseInt('3.14'): 3

=== falsy 値（8 種類） ===
Boolean(false): false
Boolean(0): false
Boolean(-0): false
Boolean(""): false
Boolean(null): false
Boolean(undefined): false
Boolean(NaN): false
Boolean(0n): false

=== 驚きの truthy 値 ===
Boolean("0"): true
Boolean("false"): true
Boolean([]): true
Boolean({}): true
```

**なぜこうなるか:**

> **Number() vs parseInt() の差**: `Number("")` は `0`、`parseInt("")` は `NaN`。`Number` は厳密に「全体が数値か」、`parseInt` は「先頭から数字が読めるところまで」。
>
> **Boolean() のルール**: **falsy 値は 8 つだけ**。それ以外はすべて truthy。`"0"`, `"false"`, `[]`, `{}` がすべて truthy なのは罠。サーバーから文字列 `"0"` が返ってきたとき `if (value)` で判定すると truthy になる。

**React との比較:**

> JSX で条件付きレンダリングするとき `{count && <Component />}` と書くと、`count` が `0` のとき `false` ではなく `0` そのものがレンダリングされる。これは `0` が falsy だけど JSX 上は数値として表示されるため。`{count > 0 && <Component />}` と書くのが安全。

---

## 4. 総合演習

### Step 4: 型変換の罠を見抜く

以下の 6 問を **まず予測してから** 実行してください。

```js
// 型変換 予測チャレンジ 6 問
const challenges = [
  { expr: '[] + []', result: [] + [] },
  { expr: '[] + {}', result: [] + {} },
  { expr: 'true + true', result: true + true },
  { expr: '"3" + 1', result: "3" + 1 },
  { expr: '"3" - 1', result: "3" - 1 },
  { expr: 'null + 1', result: null + 1 },
];

console.log("=== 型変換 予測チャレンジ ===");
challenges.forEach(({ expr, result }, index) => {
  const display = typeof result === "string" ? `"${result}"` : String(result);
  console.log(`Q${index + 1}: ${expr.padEnd(20)} → ${display} (型: ${typeof result})`);
});
```

**実行結果:**

```
=== 型変換 予測チャレンジ ===
Q1: [] + []              → "" (型: string)
Q2: [] + {}              → "[object Object]" (型: string)
Q3: true + true          → 2 (型: number)
Q4: "3" + 1              → "31" (型: string)
Q5: "3" - 1              → 2 (型: number)
Q6: null + 1             → 1 (型: number)
```

**なぜこうなるか:**

> `+` 演算子の罠：**片方が文字列なら文字列結合**、それ以外は数値加算。しかし `-`, `*`, `/` は **常に数値演算**。
>
> ```
> "3" + 1  → "31"  （+ は文字列結合を優先）
> "3" - 1  → 2     （- は数値演算のみ）
> ```
>
> これが `+` 演算子が最も罠の多い演算子と言われる理由。
>
> - `[] + []` は `[].toString() + [].toString()` → `"" + ""` → `""`
> - `null + 1` は `Number(null) + 1` → `0 + 1` → `1`

---

## 発展課題（任意）

時間に余裕があれば挑戦してください。

### 課題 1: NaN の特殊性
> 以下の出力を予測してください。なぜ `NaN === NaN` が `false` なのか、`Number.isNaN(NaN)` を使う理由は？
>
> ```js
> console.log("typeof NaN:", typeof NaN);
> console.log("NaN === NaN:", NaN === NaN);
> console.log("Number.isNaN(NaN):", Number.isNaN(NaN));
> ```

### 課題 2: Object.is と === の違い
> 以下を実行し、React の再レンダリング判定に影響するのはどちらか考えてください。
>
> ```js
> console.log(Object.is(NaN, NaN), NaN === NaN);
> console.log(Object.is(0, -0), 0 === -0);
> ```

### 課題 3: 比較演算子の連鎖
> 数学的にはどちらも `true` ですが、JavaScript では？ ヒント：比較演算子は左から右に評価され、結果は boolean になります。
>
> ```js
> console.log(1 < 2 < 3);
> console.log(3 > 2 > 1);
> ```

---

## 理解チェック — 面接で聞かれたら

### Q1: `typeof null` が `"object"` を返すのはなぜですか？

**模範回答（30 秒版）:**

> JavaScript の初期実装で、値は「型タグ + データ」のビット列で表現されていました。オブジェクトの型タグは `000` でしたが、`null` はヌルポインタ（全ビット 0）で表現されていたため、型タグも `000` と判定され `"object"` が返ります。これは仕様のバグですが、Web の既存コードとの互換性のため修正されていません。実務では `value === null` で判定します。

### Q2: `==` と `===` の違いを説明してください。実務ではどちらを使いますか？

**模範回答（30 秒版）:**

> `==` は比較前に暗黙の型変換を行い、`===` は型変換を行いません。型が異なれば `===` は即座に `false` を返します。実務では予測しづらい型変換を避けるため、原則 `===` を使います。唯一の例外は `value == null` で、これは `null` と `undefined` の両方をまとめてチェックできる便利なイディオムです。
