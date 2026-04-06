# 1-2. 値渡しと参照渡し — ハンズオン学習ガイド

> JavaScript では「プリミティブは値がコピーされ、オブジェクトは参照がコピーされる」。この一文の意味をメモリレベルで理解し、React の state 更新で「なぜスプレッド構文が必要なのか」を自信を持って説明できるようになる。

---

## 0. この教材のねらい

- プリミティブ型とオブジェクト型のメモリ上の違いを図解できるようになる
- 関数に値を渡したときの挙動の違いを正確に説明できる
- オブジェクトのシャローコピーとディープコピーを使い分けられる
- React の state 更新で「なぜスプレッド構文が必要か」を説明できる
- 面接で「JavaScript は値渡し？参照渡し？」に正確に回答できる

---

## 1. プリミティブ型の「値渡し」を確認する

### Step 1: プリミティブの代入と独立性を確認する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// プリミティブ型の代入 — 値がコピーされる
let originalNumber = 42;
let copiedNumber = originalNumber;

// コピー先を変更しても、コピー元は変わらない
copiedNumber = 100;

console.log("originalNumber:", originalNumber);
console.log("copiedNumber:", copiedNumber);

// 文字列でも同じ
let originalString = "hello";
let copiedString = originalString;

copiedString = "world";

console.log("originalString:", originalString);
console.log("copiedString:", copiedString);
```

**実行結果:**

```
originalNumber: 42
copiedNumber: 100
originalString: hello
copiedString: world
```

**なぜこうなるか:**

> プリミティブ型（number, string, boolean, null, undefined, symbol, bigint）は、変数に代入するとき **値そのものがコピー** される。メモリ上のイメージ：
>
> ```
> 代入前:
>   originalNumber → [42]
>
> copiedNumber = originalNumber の後:
>   originalNumber → [42]
>   copiedNumber   → [42]  ← 別のメモリ領域に 42 がコピーされた
>
> copiedNumber = 100 の後:
>   originalNumber → [42]  ← 影響なし
>   copiedNumber   → [100]
> ```
>
> 2 つの変数は **完全に独立** している。片方を変えてももう片方には影響しない。

**やってみよう（予測 → 検証）:**

> 以下のコードの出力を予測してから実行してみてください。
>
> ```js
> let price = 1000;
> let discountedPrice = price;
> discountedPrice = discountedPrice * 0.8;
> console.log("price:", price);
> console.log("discountedPrice:", discountedPrice);
> ```
>
> `discountedPrice` を変更しても `price` はいくらでしょうか？

---

### Step 2: 関数にプリミティブを渡したときの挙動を確認する

```js
// 関数にプリミティブを渡す — 値のコピーが渡される
function tryToChange(value) {
  value = 999;
  console.log("関数内の value:", value);
}

const myNumber = 42;
console.log("関数呼び出し前:", myNumber);

tryToChange(myNumber);

console.log("関数呼び出し後:", myNumber);
```

**実行結果:**

```
関数呼び出し前: 42
関数内の value: 999
関数呼び出し後: 42
```

**なぜこうなるか:**

> 関数の引数は **値のコピー** が渡される。関数内の `value` はローカル変数であり、呼び出し元の `myNumber` とは別のメモリ領域を指している。
>
> ```
> tryToChange(myNumber) の呼び出し時:
>   myNumber → [42]
>   value    → [42]  ← 42 のコピーが作られた
>
> value = 999 の後:
>   myNumber → [42]  ← 影響なし
>   value    → [999]
> ```

**やってみよう（予測 → 検証）:**

> 以下のコードの出力を予測してください。
>
> ```js
> function increment(num) {
>   num++;
>   return num;
> }
>
> let count = 5;
> let result = increment(count);
> console.log("count:", count);
> console.log("result:", result);
> ```
>
> `count` の値は変わっているでしょうか？

---

## 2. オブジェクト型の「参照の値渡し」を確認する

### Step 3: オブジェクトの代入が参照のコピーであることを確認する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// オブジェクトの代入 — 参照（メモリアドレス）がコピーされる
const originalUser = { name: "Alice", age: 25 };
const copiedUser = originalUser;

// copiedUser のプロパティを変更する
copiedUser.age = 30;

console.log("originalUser:", originalUser);
console.log("copiedUser:", copiedUser);

// 同じオブジェクトを指しているか確認
console.log("同じオブジェクト？:", originalUser === copiedUser);
```

**実行結果:**

```
originalUser: { name: 'Alice', age: 30 }
copiedUser: { name: 'Alice', age: 30 }
同じオブジェクト？: true
```

**なぜこうなるか:**

> オブジェクトを変数に代入すると、**オブジェクトそのもの** ではなく **そのオブジェクトへの参照（メモリアドレス）** がコピーされる。
>
> ```
> const originalUser = { name: "Alice", age: 25 } の後:
>   originalUser → [参照: 0xABC] → { name: "Alice", age: 25 }
>
> const copiedUser = originalUser の後:
>   originalUser → [参照: 0xABC] ─┐
>                                  ├→ { name: "Alice", age: 25 }
>   copiedUser   → [参照: 0xABC] ─┘
>
> copiedUser.age = 30 の後:
>   originalUser → [参照: 0xABC] ─┐
>                                  ├→ { name: "Alice", age: 30 }  ← 同じオブジェクトが変更された
>   copiedUser   → [参照: 0xABC] ─┘
> ```
>
> 2 つの変数は **同じオブジェクトを指している** ため、どちらから変更しても反映される。

**React との比較:**

> React で `const [user, setUser] = useState({ name: "Alice", age: 25 })` としたとき、`user.age = 30` と直接変更しても React は再レンダリングしない。React は `Object.is()` で前後の値を比較するが、同じ参照（同じオブジェクト）なので「変化なし」と判定される。だから `setUser({ ...user, age: 30 })` と新しいオブジェクトを作る必要がある。

**やってみよう（予測 → 検証）:**

> 以下のコードの出力を予測してください。
>
> ```js
> const teamA = { score: 0 };
> const teamB = teamA;
> teamB.score = 10;
> console.log("teamA.score:", teamA.score);
> ```
>
> `teamA` のスコアはいくつでしょうか？

---

### Step 4: 関数にオブジェクトを渡したときの挙動 — ミューテーション vs 再代入

```js
// パターン 1: 関数内でプロパティを変更する（ミューテーション）
function addSkill(userObj) {
  userObj.skills.push("TypeScript");
  console.log("関数内:", userObj);
}

const developer = { name: "Bob", skills: ["JavaScript"] };
console.log("呼び出し前:", developer);

addSkill(developer);

console.log("呼び出し後:", developer);

console.log("---");

// パターン 2: 関数内で引数を再代入する
function replaceUser(userObj) {
  userObj = { name: "Charlie", skills: ["Python"] };
  console.log("関数内（再代入後）:", userObj);
}

const engineer = { name: "Dave", skills: ["Go"] };
console.log("呼び出し前:", engineer);

replaceUser(engineer);

console.log("呼び出し後:", engineer);
```

**実行結果:**

```
呼び出し前: { name: 'Bob', skills: [ 'JavaScript' ] }
関数内: { name: 'Bob', skills: [ 'JavaScript', 'TypeScript' ] }
呼び出し後: { name: 'Bob', skills: [ 'JavaScript', 'TypeScript' ] }
---
呼び出し前: { name: 'Dave', skills: [ 'Go' ] }
関数内（再代入後）: { name: 'Charlie', skills: [ 'Python' ] }
呼び出し後: { name: 'Dave', skills: [ 'Go' ] }
```

**なぜこうなるか:**

> これが JavaScript の引数渡しの核心。**JavaScript はすべて「値渡し」** だが、オブジェクトの場合は **「参照の値渡し」（pass by sharing）** と呼ばれる。
>
> ```
> パターン 1（ミューテーション）:
>   developer → [参照: 0xABC] ─┐
>                               ├→ { name: "Bob", skills: [...] }  ← push で変更される
>   userObj   → [参照: 0xABC] ─┘
>
> パターン 2（再代入）:
>   engineer → [参照: 0xDEF] → { name: "Dave", skills: ["Go"] }
>   userObj  → [参照: 0xDEF]
>
>   userObj = { ... } の後:
>   engineer → [参照: 0xDEF] → { name: "Dave", skills: ["Go"] }  ← 影響なし
>   userObj  → [参照: 0xGHI] → { name: "Charlie", skills: ["Python"] }  ← 新しいオブジェクト
> ```
>
> **ミューテーション**（プロパティの変更）は元のオブジェクトに影響する。**再代入**（`=` で新しいオブジェクトを代入）はローカル変数の参照先が変わるだけで、呼び出し元には影響しない。

**やってみよう（予測 → 検証）:**

> 以下のコードの出力を予測してください。
>
> ```js
> function updateName(obj) {
>   obj.name = "Eve";
>   obj = { name: "Frank" };
>   obj.name = "Grace";
> }
>
> const person = { name: "Alice" };
> updateName(person);
> console.log("person.name:", person.name);
> ```
>
> `person.name` は "Alice"、"Eve"、"Frank"、"Grace" のどれでしょうか？

---

## 3. 配列の参照共有と罠

### Step 5: 配列の参照共有とミューテーションメソッドを確認する

```js
// 配列もオブジェクトなので参照がコピーされる
const originalTodos = ["買い物", "洗濯", "料理"];
const sharedTodos = originalTodos;

// push はミューテーション — 元の配列に影響する
sharedTodos.push("掃除");

console.log("originalTodos:", originalTodos);
console.log("sharedTodos:", sharedTodos);
console.log("同じ配列？:", originalTodos === sharedTodos);

console.log("---");

// ミューテーションするメソッド vs しないメソッドの比較
const numbers = [3, 1, 4, 1, 5];

// sort() はミューテーションする（元の配列を変更する）
const sortedRef = numbers.sort();
console.log("numbers（sort後）:", numbers);
console.log("sortedRef:", sortedRef);
console.log("同じ配列？:", numbers === sortedRef);

console.log("---");

// toSorted() はミューテーションしない（新しい配列を返す）— ES2023
const numbers2 = [3, 1, 4, 1, 5];
const sortedCopy = numbers2.toSorted();
console.log("numbers2（toSorted後）:", numbers2);
console.log("sortedCopy:", sortedCopy);
console.log("同じ配列？:", numbers2 === sortedCopy);
```

**実行結果:**

```
originalTodos: [ '買い物', '洗濯', '料理', '掃除' ]
sharedTodos: [ '買い物', '洗濯', '料理', '掃除' ]
同じ配列？: true
---
numbers（sort後）: [ 1, 1, 3, 4, 5 ]
sortedRef: [ 1, 1, 3, 4, 5 ]
同じ配列？: true
---
numbers2（toSorted後）: [ 3, 1, 4, 1, 5 ]
sortedCopy: [ 1, 1, 3, 4, 5 ]
同じ配列？: false
```

**なぜこうなるか:**

> 配列もオブジェクトの一種なので、代入すると参照がコピーされる。配列メソッドには **ミューテーション（破壊的）** と **非ミューテーション（非破壊的）** の 2 種類がある：
>
> | ミューテーション（元の配列を変更） | 非ミューテーション（新しい配列を返す） |
> |----------------------------------|------------------------------------|
> | `push()`, `pop()` | `concat()`, `slice()` |
> | `shift()`, `unshift()` | `map()`, `filter()` |
> | `sort()`, `reverse()` | `toSorted()`, `toReversed()` |
> | `splice()` | `toSpliced()` |
> | `fill()` | `with()` |
>
> ES2023 で追加された `toSorted()`, `toReversed()`, `toSpliced()`, `with()` は、既存のミューテーションメソッドの非破壊版。

**React との比較:**

> React の state で配列を扱うとき、`push()` や `sort()` を直接使うと参照が変わらないため再レンダリングされない。代わりに `[...todos, newTodo]`（追加）、`todos.filter(...)` （削除）、`todos.map(...)` （更新）を使って新しい配列を作る。ES2023 の `toSorted()` は React の state 更新と特に相性が良い。

**やってみよう（予測 → 検証）:**

> 以下のコードの出力を予測してください。
>
> ```js
> const listA = [1, 2, 3];
> const listB = listA;
> const listC = [...listA];
>
> listA.push(4);
>
> console.log("listA:", listA);
> console.log("listB:", listB);
> console.log("listC:", listC);
> ```
>
> `listB` と `listC` にはそれぞれ `4` が追加されているでしょうか？

---

## 4. コピーの方法を学ぶ

### Step 6: シャローコピーの 3 つの方法を実験する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// シャローコピーの 3 つの方法
const original = { name: "Alice", age: 25, hobbies: ["読書", "映画"] };

// 方法 1: スプレッド構文（最も一般的）
const copy1 = { ...original };

// 方法 2: Object.assign
const copy2 = Object.assign({}, original);

// 方法 3: 配列の場合は Array.from や [...array]
const originalArray = [1, 2, 3];
const copy3 = [...originalArray];
const copy4 = Array.from(originalArray);

// コピーは独立しているか確認
copy1.name = "Bob";
copy1.age = 30;

console.log("original.name:", original.name);
console.log("copy1.name:", copy1.name);
console.log("独立している？:", original !== copy1);

console.log("---");

// しかしシャローコピーの罠がある
copy1.hobbies.push("旅行");

console.log("original.hobbies:", original.hobbies);
console.log("copy1.hobbies:", copy1.hobbies);
console.log("hobbies は同じ配列？:", original.hobbies === copy1.hobbies);
```

**実行結果:**

```
original.name: Alice
copy1.name: Bob
独立している？: true
---
original.hobbies: [ '読書', '映画', '旅行' ]
copy1.hobbies: [ '読書', '映画', '旅行' ]
hobbies は同じ配列？: true
```

**なぜこうなるか:**

> **シャローコピー** は「1 階層目のプロパティだけをコピーする」。ネストされたオブジェクトや配列は **参照がコピー** されるため、共有されてしまう。
>
> ```
> スプレッド構文 { ...original } のメモリイメージ:
>
>   original → { name: "Alice", age: 25, hobbies: [参照: 0xAAA] }
>                                                       ↓
>   copy1    → { name: "Alice", age: 25, hobbies: [参照: 0xAAA] }
>                                                       ↓
>                                                 ["読書", "映画"]  ← 同じ配列を共有
>
> copy1.name = "Bob" → copy1 の name だけ変わる（プリミティブなので独立）
> copy1.hobbies.push("旅行") → 共有している配列が変更される
> ```

**React との比較:**

> React で `setUser({ ...user, name: "Bob" })` としたとき、`hobbies` 配列は元の `user` と同じ参照を共有している。`hobbies` も更新したい場合は `setUser({ ...user, hobbies: [...user.hobbies, "旅行"] })` のようにネストされた部分もスプレッドする必要がある。

**やってみよう（予測 → 検証）:**

> 以下のコードの出力を予測してください。
>
> ```js
> const config = { theme: "dark", sizes: { width: 100, height: 200 } };
> const newConfig = { ...config };
> newConfig.theme = "light";
> newConfig.sizes.width = 500;
>
> console.log("config.theme:", config.theme);
> console.log("config.sizes.width:", config.sizes.width);
> ```
>
> `theme` と `sizes.width` のどちらが元のオブジェクトに影響するでしょうか？

---

### Step 7: ディープコピーでネストの罠を解決する

```js
// structuredClone() — ディープコピーの標準的な方法（ES2022+）
const original = {
  name: "Alice",
  age: 25,
  hobbies: ["読書", "映画"],
  address: {
    city: "東京",
    zip: "100-0001",
  },
};

const deepCopy = structuredClone(original);

// ネストされたオブジェクトも独立しているか確認
deepCopy.hobbies.push("旅行");
deepCopy.address.city = "大阪";

console.log("original.hobbies:", original.hobbies);
console.log("deepCopy.hobbies:", deepCopy.hobbies);
console.log("hobbies は同じ配列？:", original.hobbies === deepCopy.hobbies);

console.log("---");

console.log("original.address.city:", original.address.city);
console.log("deepCopy.address.city:", deepCopy.address.city);

console.log("---");

// structuredClone の制限 — 関数はコピーできない
try {
  const withFunction = {
    name: "Alice",
    greet: function () {
      return "Hello!";
    },
  };
  const copied = structuredClone(withFunction);
} catch (error) {
  console.log("エラー:", error.message);
}

console.log("---");

// JSON.parse(JSON.stringify()) — 古い方法（制限あり）
const originalWithDate = {
  name: "Alice",
  createdAt: new Date("2024-01-01"),
  value: undefined,
  regexp: /hello/g,
};

const jsonCopy = JSON.parse(JSON.stringify(originalWithDate));

console.log("=== JSON.parse(JSON.stringify()) の制限 ===");
console.log("original.createdAt:", originalWithDate.createdAt, "型:", typeof originalWithDate.createdAt);
console.log("jsonCopy.createdAt:", jsonCopy.createdAt, "型:", typeof jsonCopy.createdAt);
console.log("original.value:", originalWithDate.value);
console.log("jsonCopy.value:", jsonCopy.value);
console.log("original.regexp:", originalWithDate.regexp);
console.log("jsonCopy.regexp:", jsonCopy.regexp);
```

**実行結果:**

```
original.hobbies: [ '読書', '映画' ]
deepCopy.hobbies: [ '読書', '映画', '旅行' ]
hobbies は同じ配列？: false
---
original.address.city: 東京
deepCopy.address.city: 大阪
---
エラー: function () {
      return "Hello!";
    } could not be cloned.
---
=== JSON.parse(JSON.stringify()) の制限 ===
original.createdAt: 2024-01-01T00:00:00.000Z 型: object
jsonCopy.createdAt: 2024-01-01T00:00:00.000Z 型: string
original.value: undefined
jsonCopy.value: undefined
original.regexp: /hello/g
jsonCopy.regexp: {}
```

**なぜこうなるか:**

> `structuredClone()` は **構造化クローンアルゴリズム** を使い、ネストされたオブジェクトや配列を再帰的にコピーする。ただし **関数、Symbol、DOM ノード** はコピーできない。
>
> 以前主流だった `JSON.parse(JSON.stringify())` は以下の問題がある：
>
> | 型 | `structuredClone()` | `JSON.parse(JSON.stringify())` |
> |----|--------------------|-----------------------------|
> | Date | Date オブジェクトを維持 | 文字列になる |
> | undefined | 維持 | プロパティが消える |
> | RegExp | 維持 | 空オブジェクト `{}` になる |
> | Map / Set | 維持 | 空オブジェクト `{}` になる |
> | 循環参照 | 対応 | エラー |
> | 関数 | エラー | プロパティが消える |
>
> 2024 年以降、ディープコピーには `structuredClone()` を使うのが標準。

**やってみよう（予測 → 検証）:**

> 以下のコードの出力を予測してください。
>
> ```js
> const original = { a: 1, b: { c: 2 } };
> const shallow = { ...original };
> const deep = structuredClone(original);
>
> original.b.c = 99;
>
> console.log("shallow.b.c:", shallow.b.c);
> console.log("deep.b.c:", deep.b.c);
> ```
>
> シャローコピーとディープコピーのどちらが影響を受けるでしょうか？

---

## 5. 総合演習 — 予測チャレンジ

### Step 8: 参照の罠を見抜く — 予測チャレンジ

以下の 10 問を **まず紙やコメントで予測してから** 実行してください。正答率 8/10 以上を目指しましょう。

```js
// 値渡しと参照渡し 予測チャレンジ 10 問
// 各問題の横に予測を書いてから実行しよう

// Q1: プリミティブの再代入
let score = 100;
let highScore = score;
score = 200;
console.log("Q1: highScore =", highScore);

// Q2: オブジェクトのプロパティ変更
const cat = { name: "Tama" };
const sameCat = cat;
sameCat.name = "Mike";
console.log("Q2: cat.name =", cat.name);

// Q3: const でもプロパティは変更できる？
const frozenObj = { value: 1 };
frozenObj.value = 2;
console.log("Q3: frozenObj.value =", frozenObj.value);

// Q4: 配列のスプレッドコピー
const arrA = [1, 2, 3];
const arrB = [...arrA];
arrB.push(4);
console.log("Q4: arrA.length =", arrA.length);

// Q5: ネストされた配列のスプレッドコピー
const matrix = [[1, 2], [3, 4]];
const matrixCopy = [...matrix];
matrixCopy[0].push(99);
console.log("Q5: matrix[0] =", matrix[0]);

// Q6: 関数内での再代入
function reset(obj) {
  obj = { count: 0 };
}
const counter = { count: 42 };
reset(counter);
console.log("Q6: counter.count =", counter.count);

// Q7: 関数内でのミューテーション
function addItem(arr) {
  arr.push("new");
}
const items = ["a", "b"];
addItem(items);
console.log("Q7: items =", items);

// Q8: === でのオブジェクト比較
const objX = { id: 1 };
const objY = { id: 1 };
const objZ = objX;
console.log("Q8: objX === objY:", objX === objY, "| objX === objZ:", objX === objZ);

// Q9: 配列の === 比較
const listX = [1, 2, 3];
const listY = [1, 2, 3];
console.log("Q9: listX === listY:", listX === listY);

// Q10: structuredClone のネスト独立性
const nested = { a: { b: { c: 1 } } };
const cloned = structuredClone(nested);
cloned.a.b.c = 999;
console.log("Q10: nested.a.b.c =", nested.a.b.c);
```

**実行結果:**

```
Q1: highScore = 100
Q2: cat.name = Mike
Q3: frozenObj.value = 2
Q4: arrA.length = 3
Q5: matrix[0] = [ 1, 2, 99 ]
Q6: counter.count = 42
Q7: items = [ 'a', 'b', 'new' ]
Q8: objX === objY: false | objX === objZ: true
Q9: listX === listY: false
Q10: nested.a.b.c = 1
```

**なぜこうなるか:**

> - **Q1:** プリミティブの値コピー → `highScore` は代入時の `100` のまま
> - **Q2:** 同じオブジェクトへの参照 → `sameCat.name` の変更は `cat` にも反映
> - **Q3:** `const` は**再代入を禁止する**だけで、プロパティの変更は許可される。プロパティも凍結するには `Object.freeze()` を使う
> - **Q4:** スプレッドで新しい配列を作成 → `arrA` と `arrB` は独立
> - **Q5:** スプレッドはシャローコピー → ネストされた `[1, 2]` は共有される
> - **Q6:** 関数内で引数を再代入 → ローカル変数の参照先が変わるだけ
> - **Q7:** 関数内で引数のミューテーション → 元の配列に影響
> - **Q8:** `===` はオブジェクトの参照を比較 → 同じ内容でも別のオブジェクトなら `false`
> - **Q9:** 配列も同様 → 同じ内容でも別の配列なら `false`
> - **Q10:** `structuredClone` はディープコピー → ネストされたオブジェクトも独立

---

## 理解チェック — 面接で聞かれたら

### Q1: JavaScript は値渡しですか？参照渡しですか？

**模範回答（30 秒版）:**

> JavaScript は **すべて値渡し** です。プリミティブ型は値そのものがコピーされ、オブジェクト型は「参照の値」がコピーされます。これを "pass by sharing" と呼ぶこともあります。関数内で引数に新しいオブジェクトを再代入しても呼び出し元には影響しませんが、引数のプロパティを変更（ミューテーション）すると呼び出し元のオブジェクトも変更されます。

**深掘りされた場合（1 分版）:**

> 「参照渡し（pass by reference）」とは、C++ の参照引数のように関数内で引数を再代入すると呼び出し元の変数自体が変わることを指します。JavaScript ではそうならないため、厳密には参照渡しではありません。ただし、オブジェクトの場合は参照（メモリアドレス）が値としてコピーされるため、プロパティのミューテーションは呼び出し元に影響します。この挙動を正確に表現するなら "pass by sharing" や "call by object sharing" と呼ぶのが適切です。実務では、意図しないミューテーションを防ぐために `structuredClone()` でディープコピーを作るか、イミュータブルなパターン（スプレッド構文など）を使います。

> **英語キーフレーズ:** "JavaScript is always pass by value. For objects, what gets copied is the reference value — not the object itself. This is sometimes called 'pass by sharing.' Reassigning the parameter inside a function doesn't affect the caller, but mutating the object's properties does."

---

### Q2: `===` でオブジェクトを比較すると何が起きますか？

**模範回答（30 秒版）:**

> `===` はオブジェクト同士を比較する場合、**参照の同一性** を比較します。つまり「同じメモリ上のオブジェクトを指しているか」を見ます。プロパティの内容が完全に同じでも、別々に作られたオブジェクトなら `false` になります。内容を比較したい場合は `JSON.stringify()` を使うか、再帰的に比較する関数を自前で作る必要があります。

**深掘りされた場合（1 分版）:**

> `{ id: 1 } === { id: 1 }` は `false` です。これはオブジェクトリテラルを書くたびに新しいオブジェクトがメモリ上に作られるためです。React の `useMemo` や `useCallback` が必要になるのもこの理由で、毎回新しいオブジェクトや関数を作ると `Object.is()` で「変化あり」と判定され、不要な再レンダリングが発生します。深い比較が必要な場合は Lodash の `_.isEqual()` や、自前の再帰比較関数を使います。また、テストフレームワーク（Jest など）の `toEqual()` は深い比較を行い、`toBe()` は参照の同一性を比較します。

> **英語キーフレーズ:** "The strict equality operator compares object identity, not structural equality. Two objects with identical properties are not equal unless they are the same reference. This is why React needs useMemo and useCallback — to preserve referential identity and avoid unnecessary re-renders."

---

### Q3: React で state を直接変更してはいけない理由を説明してください。

**模範回答（30 秒版）:**

> React は state の変更を検知するために `Object.is()` で前後の値を比較します。オブジェクトを直接ミューテーションしても参照は変わらないため、React は「変化なし」と判定して再レンダリングしません。だから `setState` にはスプレッド構文などで新しいオブジェクトを渡す必要があります。

**深掘りされた場合（1 分版）:**

> React の reconciliation アルゴリズムは、パフォーマンスのために `Object.is()` で高速に変更を検知します。これは深い比較（deep equality）ではなく浅い参照比較です。`user.name = "Bob"` と直接変更すると、`Object.is(prevUser, nextUser)` は `true`（同じ参照）を返すため、React は state が変わっていないと判断します。`setUser({ ...user, name: "Bob" })` と書くことで新しいオブジェクトが作られ、参照が変わるため React が再レンダリングをトリガーします。これがイミュータブル更新パターンと呼ばれるもので、Redux や Zustand などの状態管理ライブラリもこの原則に基づいています。Immer ライブラリを使うと、ミュータブルな書き方をしながら内部的にイミュータブルな更新を行ってくれます。

> **英語キーフレーズ:** "React uses Object.is() for shallow reference comparison to detect state changes. Direct mutation doesn't change the reference, so React doesn't know the state has changed. We create new objects with the spread operator to trigger a re-render. This is called the immutable update pattern."
