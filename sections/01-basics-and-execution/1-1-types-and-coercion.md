# 1-1. 型と型変換の罠 — ハンズオン学習ガイド

> JavaScript の 7 つのプリミティブ型、`typeof` の罠、`==` vs `===` の挙動、明示的型変換の境界ケースを手を動かして理解する。面接で「なぜ `typeof null` は `"object"` なのか？」と聞かれても自信を持って答えられるようになる。

---

## 0. この教材のねらい

- `typeof` で全プリミティブ型の判定結果を即答できるようになる
- `==` と `===` の違いを「暗黙の型変換が起きるかどうか」で説明できる
- `Number()` / `parseInt()` / `String()` / `Boolean()` の境界ケースを把握する
- 型変換が原因のバグを見抜けるようになる
- 面接で型に関する質問に 30 秒で的確に答えられる

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

**なぜこうなるか:**

> 6 つは型名がそのまま返る。しかし `typeof null` だけは `"object"` を返す。これは JavaScript 初期の実装バグが仕様として残ったもの。初期の JS エンジンでは値を「型タグ + 実データ」のビット列で表現しており、オブジェクトの型タグは `000` だった。`null` は「ヌルポインタ = 全ビット 0」で表現されていたため、型タグ部分も `000` = オブジェクトと判定されてしまった。

**やってみよう（予測 → 検証）:**

> `typeof` はプリミティブ型以外にも使える。以下のコードの出力を予測してから実行してみてください。
>
> ```js
> console.log("typeof function:", typeof function() {});
> console.log("typeof array:", typeof [1, 2, 3]);
> console.log("typeof object:", typeof { name: "JS" });
> ```
>
> ヒント：配列は `typeof` では区別できない。

---

### Step 2: null を安全に判定する方法を確認する

`typeof null === "object"` という罠があるため、null チェックには別の方法が必要です。以下を写経してください。

```js
// null を正しく判定する 3 つの方法
const targetNull = null;
const targetObject = { name: "JS" };

// 方法 1: 厳密等価で直接比較（最もシンプル）
console.log("null === null:", targetNull === null);
console.log("object === null:", targetObject === null);

// 方法 2: typeof + null チェックの組み合わせ
function getType(value) {
  if (value === null) return "null";
  return typeof value;
}

console.log("getType(null):", getType(targetNull));
console.log("getType({}):", getType(targetObject));
console.log("getType(42):", getType(42));
console.log("getType(undefined):", getType(undefined));

// 方法 3: Object.prototype.toString（最も正確）
console.log("toString null:", Object.prototype.toString.call(null));
console.log("toString array:", Object.prototype.toString.call([1, 2]));
console.log("toString object:", Object.prototype.toString.call({ a: 1 }));
console.log("toString number:", Object.prototype.toString.call(42));
```

**実行結果:**

```
null === null: true
object === null: false
getType(null): null
getType({}): object
getType(42): number
getType(undefined): undefined
toString null: [object Null]
toString array: [object Array]
toString object: [object Object]
toString number: [object Number]
```

**なぜこうなるか:**

> `===` は型変換をしないため `null` は `null` としか一致しない。`Object.prototype.toString.call()` は内部の `[[Class]]` スロットを読むため、あらゆる型を正確に判定できる。配列の判定には `Array.isArray()` もよく使われる。

**React との比較:**

> React + TypeScript のプロジェクトでは、型の判定はコンパイル時に TypeScript が行う。しかしランタイムで API レスポンスの型を検証する場合（例：`if (data === null)` のガード）は、ここで学んだ知識がそのまま必要になる。

**やってみよう（予測 → 検証）:**

> 以下のコードの出力を予測してください。
>
> ```js
> console.log("typeof NaN:", typeof NaN);
> console.log("NaN === NaN:", NaN === NaN);
> console.log("Number.isNaN(NaN):", Number.isNaN(NaN));
> ```
>
> `NaN` は「数値ではない」という意味なのに、`typeof` は何を返すでしょうか？

---

## 2. == と === の暗黙の型変換を実験する

### Step 3: `==`（緩い比較）の 10 パターンを実験する

まず、以下のコードを **そのまま写経して** 実行してください。実行前に各行の結果を予測してみましょう。

```js
// == （緩い比較）の暗黙の型変換 10 パターン
const patterns = [
  { left: "0",       right: false,     label: '"0" == false' },
  { left: "",        right: false,     label: '"" == false' },
  { left: "",        right: 0,         label: '"" == 0' },
  { left: "1",       right: true,      label: '"1" == true' },
  { left: null,      right: undefined, label: 'null == undefined' },
  { left: null,      right: 0,         label: 'null == 0' },
  { left: [],        right: 0,         label: '[] == 0' },
  { left: [],        right: "",        label: '[] == ""' },
  { left: [1],       right: 1,         label: '[1] == 1' },
  { left: [1, 2],    right: "1,2",     label: '[1,2] == "1,2"' },
];

patterns.forEach(({ left, right, label }) => {
  console.log(`${label.padEnd(22)} → ${left == right}`);
});
```

**実行結果:**

```
"0" == false           → true
"" == false            → true
"" == 0                → true
"1" == true            → true
null == undefined      → true
null == 0              → false
[] == 0                → true
[] == ""               → true
[1] == 1               → true
[1,2] == "1,2"         → true
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

**やってみよう（予測 → 検証）:**

> 以下の結果を予測してから実行してみてください。
>
> ```js
> console.log("[] == false:", [] == false);
> console.log("![] == false:", ![] == false);
> ```
>
> `[]` は truthy なのに `[] == false` が `true` になるのはなぜでしょうか？

---

### Step 4: `===`（厳密比較）で同じ 10 パターンを試す

同じパターンを `===` で比較して違いを確認しましょう。

```js
// === （厳密比較）の 10 パターン — 型変換なし
const patterns = [
  { left: "0",       right: false,     label: '"0" === false' },
  { left: "",        right: false,     label: '"" === false' },
  { left: "",        right: 0,         label: '"" === 0' },
  { left: "1",       right: true,      label: '"1" === true' },
  { left: null,      right: undefined, label: 'null === undefined' },
  { left: null,      right: 0,         label: 'null === 0' },
  { left: [],        right: 0,         label: '[] === 0' },
  { left: [],        right: "",        label: '[] === ""' },
  { left: [1],       right: 1,         label: '[1] === 1' },
  { left: [1, 2],    right: "1,2",     label: '[1,2] === "1,2"' },
];

patterns.forEach(({ left, right, label }) => {
  console.log(`${label.padEnd(24)} → ${left === right}`);
});
```

**実行結果:**

```
"0" === false            → false
"" === false             → false
"" === 0                 → false
"1" === true             → false
null === undefined       → false
null === 0               → false
[] === 0                 → false
[] === ""                → false
[1] === 1                → false
[1,2] === "1,2"          → false
```

**なぜこうなるか:**

> `===` は型変換を一切行わない。左辺と右辺の型が異なれば即座に `false`。そのため上の 10 パターンは **すべて `false`** になる。
>
> ```
> ルールは単純：
> 1. 型が違う → false
> 2. 型が同じ → 値を比較
> ```
>
> これが「常に `===` を使うべき」と言われる理由。`==` の挙動を完全に把握していない限り、意図しないバグを生みやすい。

**React との比較:**

> React の `useState` で状態を更新するとき、React は前後の値を `Object.is()` で比較して再レンダリングの要否を判定する。`Object.is()` は `===` とほぼ同じだが、`NaN === NaN` が `false` なのに対し `Object.is(NaN, NaN)` は `true` を返す点が異なる。

**やってみよう（予測 → 検証）:**

> ```js
> console.log("Object.is(NaN, NaN):", Object.is(NaN, NaN));
> console.log("Object.is(0, -0):", Object.is(0, -0));
> console.log("0 === -0:", 0 === -0);
> ```
>
> `Object.is()` と `===` の違いは上の 2 ケースだけ。React の再レンダリング判定に影響するのはどちらでしょうか？

---

## 3. 明示的型変換の挙動を一覧にする

### Step 5: `Number()` と `parseInt()` の境界ケースを実験する

```js
// Number() の変換ルール — 境界ケースを網羅
const numberTestCases = [
  ["空文字列 ''", ""],
  ["スペース '  '", "  "],
  ["文字列 '42'", "42"],
  ["文字列 '42abc'", "42abc"],
  ["文字列 '0x1A'", "0x1A"],
  ["true", true],
  ["false", false],
  ["null", null],
  ["undefined", undefined],
  ["空配列 []", []],
  ["配列 [1]", [1]],
  ["配列 [1,2]", [1, 2]],
];

console.log("=== Number() の変換結果 ===");
numberTestCases.forEach(([description, value]) => {
  const result = Number(value);
  console.log(`Number(${description.padEnd(18)}) → ${result}`);
});

console.log("\n=== parseInt() の変換結果 ===");
const parseIntTestCases = [
  ["'42'", "42"],
  ["'42abc'", "42abc"],
  ["'abc42'", "abc42"],
  ["'0x1A'", "0x1A"],
  ["''", ""],
  ["'  42  '", "  42  "],
  ["'3.14'", "3.14"],
];

parseIntTestCases.forEach(([description, value]) => {
  const result = parseInt(value);
  console.log(`parseInt(${description.padEnd(12)}) → ${result}`);
});
```

**実行結果:**

```
=== Number() の変換結果 ===
Number(空文字列 ''          ) → 0
Number(スペース '  '        ) → 0
Number(文字列 '42'          ) → 42
Number(文字列 '42abc'       ) → NaN
Number(文字列 '0x1A'        ) → 26
Number(true               ) → 1
Number(false              ) → 0
Number(null               ) → 0
Number(undefined          ) → NaN
Number(空配列 []            ) → 0
Number(配列 [1]             ) → 1
Number(配列 [1,2]           ) → NaN
```

```
=== parseInt() の変換結果 ===
parseInt('42'       ) → 42
parseInt('42abc'    ) → 42
parseInt('abc42'    ) → NaN
parseInt('0x1A'     ) → 26
parseInt(''         ) → NaN
parseInt('  42  '   ) → 42
parseInt('3.14'     ) → 3
```

**なぜこうなるか:**

> `Number()` と `parseInt()` の最大の違い：
>
> | 特性 | `Number()` | `parseInt()` |
> |------|-----------|-------------|
> | 空文字列 | `0` | `NaN` |
> | 途中に文字 | `NaN` | 数字部分まで読む |
> | 小数 | 小数として扱う | 整数部分のみ |
> | 配列 | ToPrimitive → 文字列化 | 文字列化して先頭から |
>
> `Number("")` が `0` になるのは直感に反するため、バグの温床になりやすい。フォームの入力値を数値化するときは `parseInt()` か `Number()` かを意識的に選ぶ必要がある。

**やってみよう（予測 → 検証）:**

> ```js
> console.log("Number([]):", Number([]));
> console.log("Number([null]):", Number([null]));
> console.log("Number([undefined]):", Number([undefined]));
> ```
>
> `[]` が `0` になるのに `[null]` や `[undefined]` はどうなるでしょうか？

---

### Step 6: `String()` と `Boolean()` の変換ルールを実験する

```js
// String() の変換ルール
console.log("=== String() の変換結果 ===");
const stringTestCases = [
  [42, "42"],
  [0, "0"],
  [-0, "-0"],
  [NaN, "NaN"],
  [Infinity, "Infinity"],
  [true, "true"],
  [false, "false"],
  [null, "null"],
  [undefined, "undefined"],
  [[], "[]"],
  [[1, 2], "[1,2]"],
  [{ name: "JS" }, "{ name: 'JS' }"],
];

stringTestCases.forEach(([value, expectedHint]) => {
  console.log(`String(${String(expectedHint).padEnd(18)}) → "${String(value)}"`);
});

// Boolean() の変換ルール — 8 つの falsy 値を覚える
console.log("\n=== Boolean() の変換結果（falsy 値） ===");
const falsyValues = [
  ["false", false],
  ["0", 0],
  ["-0", -0],
  ['""', ""],
  ["null", null],
  ["undefined", undefined],
  ["NaN", NaN],
  ["0n", 0n],
];

falsyValues.forEach(([label, value]) => {
  console.log(`Boolean(${label.padEnd(12)}) → ${Boolean(value)}`);
});

// 驚きの truthy 値
console.log("\n=== 驚きの truthy 値 ===");
const surprisingTruthy = [
  ['"0"（文字列のゼロ）', "0"],
  ['"false"（文字列のfalse）', "false"],
  ["[]（空配列）", []],
  ["{}（空オブジェクト）", {}],
];

surprisingTruthy.forEach(([label, value]) => {
  console.log(`Boolean(${label.padEnd(28)}) → ${Boolean(value)}`);
});
```

**実行結果:**

```
=== String() の変換結果 ===
String(42                ) → "42"
String(0                 ) → "0"
String(-0                ) → "0"
String(NaN               ) → "NaN"
String(Infinity          ) → "Infinity"
String(true              ) → "true"
String(false             ) → "false"
String(null              ) → "null"
String(undefined         ) → "undefined"
String([]                ) → ""
String([1,2]             ) → "1,2"
String({ name: 'JS' }   ) → "[object Object]"

=== Boolean() の変換結果（falsy 値） ===
Boolean(false       ) → false
Boolean(0           ) → false
Boolean(-0          ) → false
Boolean(""          ) → false
Boolean(null        ) → false
Boolean(undefined   ) → false
Boolean(NaN         ) → false
Boolean(0n          ) → false

=== 驚きの truthy 値 ===
Boolean("0"（文字列のゼロ）             ) → true
Boolean("false"（文字列のfalse）        ) → true
Boolean([]（空配列）                     ) → true
Boolean({}（空オブジェクト）              ) → true
```

**なぜこうなるか:**

> `Boolean()` のルールは覚え方がシンプル：**falsy 値は 8 つだけ**、それ以外はすべて truthy。
>
> ```
> falsy 値: false, 0, -0, "", null, undefined, NaN, 0n
> それ以外 → すべて truthy（"0", "false", [], {} を含む）
> ```
>
> 特に `"0"` が truthy なのは罠。サーバーから文字列 `"0"` が返ってきたとき、`if (value)` で判定すると truthy になってしまう。

**React との比較:**

> JSX で条件付きレンダリングするとき `{count && <Component />}` と書くと、`count` が `0` のとき `false` ではなく `0` そのものがレンダリングされる。これは `0` が falsy だけど JSX 上は数値として表示されるため。`{count > 0 && <Component />}` と書くのが安全。

**やってみよう（予測 → 検証）:**

> ```js
> console.log("String(-0):", String(-0));
> console.log("-0 + '':", -0 + "");
> console.log("JSON.stringify(-0):", JSON.stringify(-0));
> console.log("String([] + []):", String([] + []));
> console.log("String([] + {}):", String([] + {}));
> console.log("String({} + []):", String({} + []));
> ```
>
> `-0` の文字列化は `"0"` になるが、本当に `-0` を保持したい場合はどうすればよいでしょうか？

---

## 4. 総合演習

### Step 7: 型変換の罠を見抜く — 予測チャレンジ

以下の 10 問を **まず紙やコメントで予測してから** 実行してください。正答率 8/10 以上を目指しましょう。

```js
// 型変換 予測チャレンジ 10 問
// 各問題の横に予測を書いてから実行しよう
const challenges = [
  { expr: '[] + []', result: [] + [] },
  { expr: '[] + {}', result: [] + {} },
  { expr: '{} + []', result: String({} + []) },
  { expr: 'true + true', result: true + true },
  { expr: '"3" + 1', result: "3" + 1 },
  { expr: '"3" - 1', result: "3" - 1 },
  { expr: '"3" * "2"', result: "3" * "2" },
  { expr: 'null + 1', result: null + 1 },
  { expr: 'undefined + 1', result: undefined + 1 },
  { expr: '"" - 1', result: "" - 1 },
];

console.log("=== 型変換 予測チャレンジ ===\n");
challenges.forEach(({ expr, result }, index) => {
  // NaN は JSON.stringify で null になるので String() を使う
  const display = typeof result === "string" ? `"${result}"` : String(result);
  console.log(`Q${index + 1}: ${expr.padEnd(20)} → ${display} (型: ${typeof result})`);
});
```

**実行結果:**

```
=== 型変換 予測チャレンジ ===

Q1: [] + []              → "" (型: string)
Q2: [] + {}              → "[object Object]" (型: string)
Q3: {} + []              → "[object Object]" (型: string)
Q4: true + true          → 2 (型: number)
Q5: "3" + 1              → "31" (型: string)
Q6: "3" - 1              → 2 (型: number)
Q7: "3" * "2"            → 6 (型: number)
Q8: null + 1             → 1 (型: number)
Q9: undefined + 1        → NaN (型: number)
Q10: "" - 1              → -1 (型: number)
```

**なぜこうなるか:**

> `+` 演算子の罠：**片方が文字列なら文字列結合**、それ以外は数値加算。しかし `-`, `*`, `/` は**常に数値演算**。
>
> ```
> "3" + 1  → "31"  （+ は文字列結合を優先）
> "3" - 1  → 2     （- は数値演算のみ）
> ```
>
> これが `+` 演算子が最も罠の多い演算子と言われる理由。
>
> `[] + []` は `[].toString() + [].toString()` → `"" + ""` → `""`
>
> `null + 1` は `Number(null) + 1` → `0 + 1` → `1`
>
> `undefined + 1` は `Number(undefined) + 1` → `NaN + 1` → `NaN`

**やってみよう（予測 → 検証）:**

> 最後のチャレンジ。以下の出力を予測してください。
>
> ```js
> console.log("Q11:", 1 < 2 < 3);
> console.log("Q12:", 3 > 2 > 1);
> ```
>
> 数学的にはどちらも `true` ですが、JavaScript では…？
> ヒント：比較演算子は左から右に評価され、結果は `boolean` になります。

---

## 理解チェック — 面接で聞かれたら

### Q1: `typeof null` が `"object"` を返すのはなぜですか？

**模範回答（30 秒版）:**

> JavaScript の初期実装で、値は「型タグ + データ」のビット列で表現されていました。オブジェクトの型タグは `000` でしたが、`null` はヌルポインタ（全ビット 0）で表現されていたため、型タグも `000` と判定され `"object"` が返ります。これは仕様のバグですが、既存コードの互換性のために修正されていません。

**深掘りされた場合（1 分版）:**

> この挙動は ES1（1997 年）から存在し、TC39 で修正が提案されたこと（`typeof null === "null"`）もありますが、Web 上の膨大な既存コードが壊れるため却下されました。実務では `value === null` で判定するか、TypeScript の型システムで null チェックを行います。`Object.prototype.toString.call(null)` は `"[object Null]"` を正しく返すため、ライブラリレベルの型判定にはこちらが使われます。

> **英語キーフレーズ:** "It's a historical bug from the first implementation of JavaScript. The internal type tag for objects was `000`, and `null` was represented as a null pointer — all zeros — so its type tag was also `000`. It was never fixed for backward compatibility."

---

### Q2: `==` と `===` の違いを説明してください。実務ではどちらを使いますか？

**模範回答（30 秒版）:**

> `==` は比較前に暗黙の型変換（Abstract Equality Comparison）を行い、`===` は型変換を行いません。型が異なれば `===` は即座に `false` を返します。実務では予測しづらい型変換を避けるため、原則 `===` を使います。唯一の例外は `value == null` で、これは `null` と `undefined` の両方をまとめてチェックできる便利なイディオムです。

**深掘りされた場合（1 分版）:**

> ESLint の `eqeqeq` ルールで `===` を強制するのが一般的です。`==` の型変換ルールは仕様書の Abstract Equality Comparison アルゴリズムに定義されており、boolean を先に数値化してから比較する、オブジェクトは ToPrimitive で変換する、など直感に反するステップがあります。例えば `[] == false` は `true` になりますが、`[]` は truthy です。このような混乱を避けるため `===` をデフォルトにします。ただし `value == null` は `value === null || value === undefined` と同義で、TypeScript の strict null checks と併用して nullish ガードに使うことがあります。

> **英語キーフレーズ:** "The double equals performs type coercion before comparison, while triple equals checks both type and value without any conversion. In practice, I always use strict equality to avoid unexpected coercion, with the one exception of `== null` to check for both null and undefined."

---

### Q3: JavaScript の falsy な値をすべて挙げてください。

**模範回答（30 秒版）:**

> `false`, `0`, `-0`, `""`（空文字列）, `null`, `undefined`, `NaN`, `0n`（BigInt のゼロ）の 8 つです。それ以外はすべて truthy で、`"0"`（文字列のゼロ）、`[]`（空配列）、`{}`（空オブジェクト）も truthy になります。

**深掘りされた場合（1 分版）:**

> よくある罠として、React の JSX で `{count && <Component />}` と書くと、`count` が `0` のとき `0` がそのまま画面に表示されます。`0` は falsy なので右辺は評価されませんが、`&&` 演算子は falsy な左辺の値そのものを返すため、JSX は数値 `0` をテキストノードとしてレンダリングします。`{count > 0 && <Component />}` と書くのが安全です。また、サーバーから `"0"` や `"false"` という文字列が返ってきた場合、これらは truthy なので `if (value)` では期待通りに動きません。

> **英語キーフレーズ:** "There are exactly eight falsy values: `false`, `0`, `-0`, empty string, `null`, `undefined`, `NaN`, and `0n`. Everything else is truthy, including empty arrays and objects. A common React pitfall is using `count && <Component />` where `count` is zero — it renders `0` instead of nothing."
