# 1-5. イベントループ・タスクキュー・マイクロタスク — ハンズオン学習ガイド

> JavaScript はシングルスレッドなのに、なぜ非同期処理が実現できるのか。コールスタック（1-4 で学習済み）の「外側」で何が起きているかを、実際にコードを動かしながら理解する教材。

---

## 0. この教材のねらい

**想定所要時間: 約 60 分**

- イベントループの「1 サイクル」で何が起きるかを図と言葉で説明できる
- `setTimeout(fn, 0)` がなぜ「即座に実行されない」のかをメカニズムから説明できる
- タスクキュー（マクロタスク）とマイクロタスクキューの優先度の違いを実演できる
- `Promise.then` / `queueMicrotask` / `setTimeout` / `await` の実行順を正確に予測できる

---

## 1. setTimeout は「すぐ」ではない

### Step 1: setTimeout(fn, 0) はなぜ即実行されないのか

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// non-blocking.js
console.log('1: main 開始');

setTimeout(function delayedTask() {
  console.log('2: setTimeout のコールバック実行');
}, 0); // 0ms 指定なのに…？

console.log('3: main 終了');
```

`node non-blocking.js` で実行してください。

**実行結果:**

```
1: main 開始
3: main 終了
2: setTimeout のコールバック実行
```

**なぜこうなるか:**

> `setTimeout(fn, 0)` は「0ms 後に実行」ではなく「**タスクキューに入れて、コールスタックが空になったら実行**」という意味。以下の流れが起きている：
>
> ```
> 1. console.log('1: main 開始')     → コールスタックで即実行
> 2. setTimeout(delayedTask, 0)      → Web API / Node のタイマーに登録。コールバックはキューへ
> 3. console.log('3: main 終了')     → コールスタックで即実行
> 4. --- コールスタックが空になった ---
> 5. イベントループがタスクキューを確認 → delayedTask を取り出して実行
> ```
>
> JavaScript はシングルスレッド。コールスタック上の関数が完了するまで、次のタスクには進めない。`setTimeout` は遅延 API ではなく、**スケジューリング API** と理解するのが正しい。

---

## 2. イベントループの優先順位

### Step 2: マイクロタスクとマクロタスクの優先順位を観察する

写経の前に、イベントループの構造を頭に入れてください。

```
┌──────────────────────────────────────────────┐
│                  コールスタック                  │
│  (同期コードが上から順に積まれて実行される)        │
└──────────────┬───────────────────────────────┘
               │ コールスタックが空になったら
               ▼
┌──────────────────────────────────────────────┐
│             マイクロタスクキュー                  │
│  (Promise.then, queueMicrotask)              │
│  → すべて処理するまで次に進まない               │
└──────────────┬───────────────────────────────┘
               │ マイクロタスクが空になったら
               ▼
┌──────────────────────────────────────────────┐
│         タスクキュー（マクロタスクキュー）          │
│  (setTimeout, setInterval, I/O callbacks)    │
│  → 1 つ取り出して実行 → またマイクロタスク確認    │
└──────────────────────────────────────────────┘
```

> **ポイント:** イベントループの 1 サイクルは「マイクロタスクを **全部** 消化 → マクロタスクを **1 つ** 実行」。この非対称性がすべての鍵。

では、以下のコードを **そのまま写経して** 実行してください。

```js
// event-loop-anatomy.js
// イベントループの1サイクルを観察する

console.log('1: script 開始 (同期)');

setTimeout(function macroTask() {
  console.log('5: setTimeout コールバック (マクロタスク)');
}, 0);

Promise.resolve().then(function microTask() {
  console.log('3: Promise.then コールバック (マイクロタスク)');
});

queueMicrotask(function anotherMicroTask() {
  console.log('4: queueMicrotask コールバック (マイクロタスク)');
});

console.log('2: script 終了 (同期)');
```

**実行結果:**

```
1: script 開始 (同期)
2: script 終了 (同期)
3: Promise.then コールバック (マイクロタスク)
4: queueMicrotask コールバック (マイクロタスク)
5: setTimeout コールバック (マクロタスク)
```

**なぜこうなるか:**

> 1. 同期コード（`1:`, `2:`）がまずコールスタックで実行される
> 2. `setTimeout` のコールバックはタスクキュー（マクロタスク）に入る
> 3. `Promise.then` と `queueMicrotask` のコールバックはマイクロタスクキューに入る
> 4. 同期コードが終わり、コールスタックが空になる
> 5. イベントループがまず **マイクロタスクキューをすべて** 処理（3 → 4）
> 6. マイクロタスクが空になったので、タスクキューから **1 つ** 取り出して実行（5）
>
> さらに重要：マイクロタスクの中で新たなマイクロタスクを追加しても、そのマイクロタスクも「今のマイクロタスク消化フェーズ」で処理される。マイクロタスクキューが完全に空になるまで、マクロタスクキューには手をつけない。

---

## 3. async/await の実行順

### Step 3: async/await の実行順序を予測する

`async/await` は Promise のシンタックスシュガー。イベントループの観点でどう動くか確認しましょう。

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// async-await-order.js
async function asyncFunction() {
  console.log('2: asyncFunction 開始 (同期部分)');

  const result = await Promise.resolve('resolved!');
  // ↑ ここで asyncFunction は一時停止し、残りはマイクロタスクとしてキューに入る

  console.log('5: await の後 (マイクロタスク):', result);
  return result;
}

console.log('1: script 開始');

asyncFunction().then(function thenCallback(value) {
  console.log('6: asyncFunction の .then:', value);
});

Promise.resolve().then(function otherMicro() {
  console.log('4: 別の Promise.then');
});

console.log('3: script 終了');
```

**実行結果:**

```
1: script 開始
2: asyncFunction 開始 (同期部分)
3: script 終了
4: 別の Promise.then
5: await の後 (マイクロタスク): resolved!
6: asyncFunction の .then: resolved!
```

**なぜこうなるか:**

> `async function` の中で `await` に到達すると、その関数は **一時停止** する。`await` より後のコードは **マイクロタスクとしてキューに入る**。つまり：
>
> ```js
> // これは概念的にこう変換される：
> async function asyncFunction() {
>   console.log('2: ...');          // 同期実行
>   const result = await Promise.resolve('resolved!');
>   console.log('5: ...');          // ← ここからマイクロタスク
> }
>
> // ↓ 内部的にはこう動く：
> function asyncFunction() {
>   console.log('2: ...');
>   return Promise.resolve('resolved!').then(result => {
>     console.log('5: ...', result);
>     return result;
>   });
> }
> ```
>
> `4:` が `5:` より先に出力されるのは、`otherMicro` と `await` の後続処理が両方マイクロタスクキューに入るが、`otherMicro` のほうが先にキューに登録されるため。

**React との比較:**

> React の `useEffect` のコールバックが実行されるタイミングも、イベントループと密接に関わっている。`useEffect` は描画後（マイクロタスク消化後）に実行される「マクロタスク的」なタイミング。一方 `useLayoutEffect` は描画前（マイクロタスクに近いタイミング）で実行される。

---

## 4. UI を壊す失敗例

### Step 4: マイクロタスクで UI をブロックする失敗例

以下の HTML ファイルを作成し、ブラウザで開いてください。

```html
<!-- microtask-blocking.html -->
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>マイクロタスクによるUIブロック</title>
</head>
<body>
  <h1>カウンター: <span id="counter">0</span></h1>
  <button id="startBad">悪い例: マイクロタスクで100万回更新</button>
  <button id="startGood">良い例: マクロタスクで分割更新</button>

  <script>
    const counterEl = document.getElementById('counter');

    // 悪い例: マイクロタスクでループ → UI が固まる
    document.getElementById('startBad').addEventListener('click', function onBadClick() {
      console.log('悪い例: 開始');
      let count = 0;

      function microLoop() {
        if (count < 1000000) {
          count++;
          counterEl.textContent = count;
          queueMicrotask(microLoop); // マイクロタスクで再帰
        } else {
          console.log('悪い例: 完了');
        }
      }

      microLoop();
      // マイクロタスクが全部消化されるまでブラウザは描画できない！
      // → 100万回完了するまで画面が固まり、最後に一気に「1000000」と表示される
    });

    // 良い例: setTimeout で分割 → UI が更新される
    document.getElementById('startGood').addEventListener('click', function onGoodClick() {
      console.log('良い例: 開始');
      let count = 0;
      const batchSize = 1000;

      function macroLoop() {
        const end = Math.min(count + batchSize, 1000000);
        while (count < end) {
          count++;
        }
        counterEl.textContent = count;

        if (count < 1000000) {
          setTimeout(macroLoop, 0); // マクロタスクで次のバッチをスケジュール
          // → 各バッチの間にブラウザが描画できる
        } else {
          console.log('良い例: 完了');
        }
      }

      macroLoop();
    });
  </script>
</body>
</html>
```

**動作の違い:**

| | 悪い例（マイクロタスク） | 良い例（マクロタスク） |
|---|---|---|
| 画面更新 | 100万回完了まで固まる | バッチごとに更新される |
| ユーザー操作 | 受け付けない | 応答可能 |
| 理由 | マイクロタスクは全消化まで描画をブロック | setTimeout の間に描画が入り込める |

**React との比較:**

> React 18 の Concurrent Mode が解決する問題はまさにこれ。大きなレンダリングを小さなチャンクに分割し、各チャンクの間にブラウザに制御を戻す。内部的には `MessageChannel`（マクロタスク）を使って「マクロタスク単位でチャンクを実行 → ブラウザに描画を許可 → 次のチャンク」というパターンを採用している。`startTransition` で「優先度の低い更新」をマークするのも、このスケジューリングの仕組みの上に成り立っている。

---

## 発展課題（任意）

時間に余裕があれば挑戦してください。

### 課題 1: setTimeout の遅延を変えても順序は変わるか
> Step 1 のコードで `setTimeout` の遅延を `0` から `1000` に変えたら、出力の **順序** は変わるでしょうか？それとも同じでしょうか？まず予測してから実行してください。

### 課題 2: 実行順序パズル（面接頻出）
> 以下の出力順を **紙に書いてから** 実行してください。
>
> ```js
> console.log('A');
> setTimeout(() => {
>   console.log('B');
>   Promise.resolve().then(() => console.log('C'));
> }, 0);
> Promise.resolve().then(() => {
>   console.log('D');
>   setTimeout(() => console.log('E'), 0);
> });
> setTimeout(() => console.log('F'), 0);
> Promise.resolve().then(() => console.log('G'));
> console.log('H');
> ```

### 課題 3: Node.js の process.nextTick と setImmediate
> Node.js には `process.nextTick`（マイクロタスクより優先）と `setImmediate`（setTimeout の後）という固有 API がある。以下の出力順を予測して実行してみよう。
>
> ```js
> setTimeout(() => console.log('setTimeout'), 0);
> setImmediate(() => console.log('setImmediate'));
> process.nextTick(() => console.log('nextTick'));
> Promise.resolve().then(() => console.log('promise'));
> ```

---

## 理解チェック — 面接で聞かれたら

### Q1: 「JavaScript のイベントループを説明してください」

**模範回答（30 秒版）:**

> JavaScript はシングルスレッドで、1 つのコールスタックしか持ちません。イベントループは、コールスタックが空になったときにキューからタスクを取り出して実行する仕組みです。キューには優先度があり、マイクロタスク（Promise.then など）はマクロタスク（setTimeout など）より先に処理されます。マイクロタスクは全部消化してからマクロタスクに進むのが重要な特徴です。

### Q2: 「`setTimeout(fn, 0)` は本当に 0ms 後に実行されますか？」

**模範回答（30 秒版）:**

> いいえ。`setTimeout(fn, 0)` は「最短で次のマクロタスクのタイミングで実行」という意味です。コールスタックが空になり、マイクロタスクがすべて消化された後に初めて実行されます。また、ブラウザには最小遅延（ネストが深い場合は 4ms）の制限があるため、実際には 0ms にはなりません。
