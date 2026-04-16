# 1-2. 値渡しと参照渡し — ハンズオン学習ガイド

> JavaScript では「プリミティブは値がコピーされ、オブジェクトは参照がコピーされる」。この一文の意味をメモリレベルで理解し、React の state 更新で「なぜスプレッド構文が必要なのか」を自信を持って説明できるようになる。

---

## 0. この教材のねらい

**想定所要時間: 約 60 分**

- プリミティブ型とオブジェクト型のメモリ上の違いを図解できるようになる
- 関数に値を渡したときの挙動の違いを正確に説明できる
- オブジェクトのシャローコピーとディープコピーを使い分けられる
- React の state 更新で「なぜスプレッド構文が必要か」を説明できる

---

## 1. プリミティブの値コピー

### Step 1: プリミティブの値コピーと関数渡しを確認する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// プリミティブ型の代入 — 値がコピーされる
let originalNumber = 42;
let copiedNumber = originalNumber;

copiedNumber = 100;

console.log("originalNumber:", originalNumber);
console.log("copiedNumber:", copiedNumber);

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
originalNumber: 42
copiedNumber: 100
関数呼び出し前: 42
関数内の value: 999
関数呼び出し後: 42
```

**なぜこうなるか:**

> プリミティブ型（number, string, boolean, null, undefined, symbol, bigint）は、変数に代入するときも関数に渡すときも **値そのものがコピー** される。メモリ上のイメージ：
>
> ```
> let copiedNumber = originalNumber:
>   originalNumber → [42]
>   copiedNumber   → [42]  ← 別のメモリ領域に 42 がコピーされた
>
> copiedNumber = 100:
>   originalNumber → [42]  ← 影響なし
>   copiedNumber   → [100]
> ```
>
> 2 つの変数は **完全に独立** している。関数の引数も同じく値のコピーなので、関数内で書き換えても呼び出し元には影響しない。

---

## 2. オブジェクトの参照渡しとミューテーション

### Step 2: オブジェクトの参照コピーと関数内ミューテーション vs 再代入を体験する

```js
// オブジェクトの代入 — 参照（メモリアドレス）がコピーされる
const originalUser = { name: "Alice", age: 25 };
const copiedUser = originalUser;

copiedUser.age = 30;
console.log("originalUser:", originalUser);
console.log("同じオブジェクト？:", originalUser === copiedUser);

console.log("---");

// パターン 1: 関数内でプロパティを変更する（ミューテーション）
function addSkill(userObj) {
  userObj.skills.push("TypeScript");
}

const developer = { name: "Bob", skills: ["JavaScript"] };
console.log("呼び出し前:", developer);
addSkill(developer);
console.log("呼び出し後:", developer);

console.log("---");

// パターン 2: 関数内で引数を再代入する（失敗例）
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
originalUser: { name: 'Alice', age: 30 }
同じオブジェクト？: true
---
呼び出し前: { name: 'Bob', skills: [ 'JavaScript' ] }
呼び出し後: { name: 'Bob', skills: [ 'JavaScript', 'TypeScript' ] }
---
呼び出し前: { name: 'Dave', skills: [ 'Go' ] }
関数内（再代入後）: { name: 'Charlie', skills: [ 'Python' ] }
呼び出し後: { name: 'Dave', skills: [ 'Go' ] }
```

**なぜこうなるか（失敗体験を読み解く）:**

> JavaScript は **すべて「値渡し」** だが、オブジェクトの場合は **「参照の値渡し」（pass by sharing）** と呼ばれる。コピーされるのは「オブジェクトそのもの」ではなく「オブジェクトへの参照（メモリアドレス）」。
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
>   userObj  → [参照: 0xGHI] → { name: "Charlie", skills: ["Python"] }
> ```
>
> **ミューテーション**（プロパティの変更）は元のオブジェクトに影響する。**再代入**（`=` で新しいオブジェクトを代入）はローカル変数の参照先が変わるだけで、呼び出し元には影響しない。

**React との比較:**

> React で `const [user, setUser] = useState({ name: "Alice", age: 25 })` としたとき、`user.age = 30` と直接変更しても React は再レンダリングしない。React は `Object.is()` で前後の値を比較するが、同じ参照（同じオブジェクト）なので「変化なし」と判定される。だから `setUser({ ...user, age: 30 })` と新しいオブジェクトを作る必要がある。

---

## 3. 配列の参照共有とシャローコピーの罠

### Step 3: 配列の参照共有とシャローコピーの罠を理解する

```js
// 配列もオブジェクトなので参照がコピーされる
const originalTodos = ["買い物", "洗濯", "料理"];
const sharedTodos = originalTodos;

sharedTodos.push("掃除");
console.log("originalTodos:", originalTodos);
console.log("同じ配列？:", originalTodos === sharedTodos);

console.log("---");

// スプレッド構文でシャローコピーを作る
const numbers2 = [1, 2, [99, 100]];
const shallowCopy = [...numbers2];

// 1 階層目は独立する
shallowCopy.push(4);
console.log("numbers2.length:", numbers2.length);
console.log("shallowCopy.length:", shallowCopy.length);

// しかしネストされた配列は共有されてしまう
shallowCopy[2].push(999);
console.log("numbers2[2]:", numbers2[2]);
console.log("shallowCopy[2]:", shallowCopy[2]);
console.log("ネスト配列は同じ？:", numbers2[2] === shallowCopy[2]);
```

**実行結果:**

```
originalTodos: [ '買い物', '洗濯', '料理', '掃除' ]
同じ配列？: true
---
numbers2.length: 3
shallowCopy.length: 4
numbers2[2]: [ 99, 100, 999 ]
shallowCopy[2]: [ 99, 100, 999 ]
ネスト配列は同じ？: true
```

**なぜこうなるか:**

> 配列もオブジェクトの一種なので、代入すると参照がコピーされる。**シャローコピー** は「1 階層目のプロパティだけをコピー」する。ネストされたオブジェクトや配列は **参照がコピー** されるため、共有される。
>
> ```
> shallowCopy = [...numbers2] のメモリイメージ:
>
>   numbers2    → [ 1, 2, [参照: 0xAAA] ]
>                                ↓
>   shallowCopy → [ 1, 2, [参照: 0xAAA] ]
>                                ↓
>                          [99, 100]  ← 同じ配列を共有
> ```
>
> 配列メソッドにも **ミューテーション（破壊的）** と **非ミューテーション（非破壊的）** の 2 種類がある：
>
> | 破壊的（元の配列を変更） | 非破壊的（新しい配列を返す） |
> |---|---|
> | `push`, `pop`, `shift`, `unshift` | `concat`, `slice`, `map`, `filter` |
> | `sort`, `reverse`, `splice` | `toSorted`, `toReversed`, `toSpliced` (ES2023) |

**React との比較:**

> React の state で配列を扱うとき、`push()` や `sort()` を直接使うと参照が変わらないため再レンダリングされない。代わりに `[...todos, newTodo]`（追加）、`todos.filter(...)` （削除）、`todos.map(...)` （更新）を使って新しい配列を作る。ネストされた要素を更新するときは、その階層もスプレッドする必要がある。

---

## 4. ディープコピーで罠を解決する

### Step 4: structuredClone でディープコピーを使い分ける

```js
// structuredClone() — ディープコピーの標準的な方法（ES2022+）
const original = {
  name: "Alice",
  hobbies: ["読書", "映画"],
  address: { city: "東京", zip: "100-0001" },
};

const deepCopy = structuredClone(original);

// ネストされたオブジェクトも独立しているか確認
deepCopy.hobbies.push("旅行");
deepCopy.address.city = "大阪";

console.log("original.hobbies:", original.hobbies);
console.log("deepCopy.hobbies:", deepCopy.hobbies);
console.log("hobbies は同じ配列？:", original.hobbies === deepCopy.hobbies);
console.log("original.address.city:", original.address.city);
console.log("deepCopy.address.city:", deepCopy.address.city);
```

**実行結果:**

```
original.hobbies: [ '読書', '映画' ]
deepCopy.hobbies: [ '読書', '映画', '旅行' ]
hobbies は同じ配列？: false
original.address.city: 東京
deepCopy.address.city: 大阪
```

**なぜこうなるか:**

> `structuredClone()` は **構造化クローンアルゴリズム** を使い、ネストされたオブジェクトや配列を再帰的にコピーする。ただし **関数、Symbol、DOM ノード** はコピーできない。
>
> 以前主流だった `JSON.parse(JSON.stringify())` には以下の問題がある：
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

---

## 発展課題（任意）

時間に余裕があれば挑戦してください。

### 課題 1: ネストされた config の罠
> `theme` と `sizes.width` のどちらが元のオブジェクトに影響するか予測してください。
>
> ```js
> const config = { theme: "dark", sizes: { width: 100 } };
> const newConfig = { ...config };
> newConfig.theme = "light";
> newConfig.sizes.width = 500;
> console.log(config.theme, config.sizes.width);
> ```

### 課題 2: 関数内での再代入とミューテーションの順序
> `person.name` は "Alice"、"Eve"、"Frank"、"Grace" のどれでしょうか？
>
> ```js
> function updateName(obj) {
>   obj.name = "Eve";
>   obj = { name: "Frank" };
>   obj.name = "Grace";
> }
> const person = { name: "Alice" };
> updateName(person);
> console.log(person.name);
> ```

### 課題 3: shallow vs deep の影響範囲
> シャローコピーとディープコピーのどちらが影響を受けるでしょうか？
>
> ```js
> const original = { a: 1, b: { c: 2 } };
> const shallow = { ...original };
> const deep = structuredClone(original);
> original.b.c = 99;
> console.log(shallow.b.c, deep.b.c);
> ```

---

## 理解チェック — 面接で聞かれたら

### Q1: JavaScript は値渡しですか？参照渡しですか？

**模範回答（30 秒版）:**

> JavaScript は **すべて値渡し** です。プリミティブ型は値そのものがコピーされ、オブジェクト型は「参照の値」がコピーされます。これを "pass by sharing" と呼ぶこともあります。関数内で引数に新しいオブジェクトを再代入しても呼び出し元には影響しませんが、引数のプロパティを変更（ミューテーション）すると呼び出し元のオブジェクトも変更されます。

### Q2: React で state を直接変更してはいけない理由を説明してください。

**模範回答（30 秒版）:**

> React は state の変更を検知するために `Object.is()` で前後の値を比較します。オブジェクトを直接ミューテーションしても参照は変わらないため、React は「変化なし」と判定して再レンダリングしません。だから `setState` にはスプレッド構文（`{ ...user, name: "Bob" }`）などで新しいオブジェクトを渡す必要があります。
