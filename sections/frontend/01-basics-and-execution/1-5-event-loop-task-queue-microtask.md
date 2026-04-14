# 1-5. イベントループ・タスクキュー・マイクロタスク — ハンズオン学習ガイド

> JavaScript はシングルスレッドなのに、なぜ非同期処理が実現できるのか。コールスタック（1-4 で学習済み）の「外側」で何が起きているかを、実際にコードを動かしながら理解する教材。

---

## 0. この教材のねらい

完成時に以下ができるようになる：

- イベントループの「1 サイクル」で何が起きるかを図と言葉で説明できる
- `setTimeout(fn, 0)` がなぜ「即座に実行されない」のかをメカニズムから説明できる
- タスクキュー（マクロタスク）とマイクロタスクキューの優先度の違いを実演できる
- `Promise.then` / `queueMicrotask` / `setTimeout` / `requestAnimationFrame` の実行順を正確に予測できる
- React の `setState` バッチ更新やライフサイクル順序を、イベントループの視点から説明できる

---

## 1. シングルスレッドと非同期の矛盾を体感する

### Step 1: 同期処理でスレッドをブロックする

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// blocking.js
// 重い同期処理でスレッドが止まることを体感する

function heavyTask(label) {
  console.log(`${label}: 開始`);
  const start = Date.now();
  // 約2秒間 CPU を占有するループ
  while (Date.now() - start < 2000) {
    // 何もしない — ただ CPU を回す
  }
  console.log(`${label}: 完了 (${Date.now() - start}ms)`);
}

console.log('main: 開始');
heavyTask('重い処理A');
console.log('main: Aの後');
heavyTask('重い処理B');
console.log('main: すべて完了');
```

`node blocking.js` で実行してください。

**実行結果:**

```
main: 開始
重い処理A: 開始
重い処理A: 完了 (2000ms)
main: Aの後
重い処理B: 開始
重い処理B: 完了 (2000ms)
main: すべて完了
```

**なぜこうなるか:**

> JavaScript はシングルスレッド。コールスタック上の関数が完了するまで、次の行に進めない。`heavyTask` が while ループで CPU を占有している間、他のコードは一切実行されない。ブラウザでこれをやると画面が完全にフリーズする。

### Step 2: setTimeout で「後回し」にする

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// non-blocking.js
// setTimeout で重い処理を「スケジュール」するとどうなるか

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
> 6. console.log('2: setTimeout ...')
> ```

**やってみよう（予測 → 検証）:**

> `setTimeout` の遅延を `0` から `1000` に変えたら、出力の **順序** は変わるでしょうか？それとも同じでしょうか？
> まず予測を書いてから実行してみてください。

---

## 2. イベントループの仕組みを分解する

### Step 3: イベントループの全体像を描く

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

`node event-loop-anatomy.js` で実行してください。

**実行結果:**

```
1: script 開始 (同期)
2: script 終了 (同期)
3: Promise.then コールバック (マイクロタスク)
4: queueMicrotask コールバック (マイクロタスク)
5: setTimeout コールバック (マクロタスク)
```

**なぜこうなるか:**

> 1. 同期コード（`console.log('1:...')`, `console.log('2:...')`）がまずコールスタックで実行される
> 2. `setTimeout` のコールバックはタスクキュー（マクロタスク）に入る
> 3. `Promise.then` と `queueMicrotask` のコールバックはマイクロタスクキューに入る
> 4. 同期コードが終わり、コールスタックが空になる
> 5. イベントループがまず **マイクロタスクキューをすべて** 処理（3 → 4）
> 6. マイクロタスクが空になったので、タスクキューから **1 つ** 取り出して実行（5）

### Step 4: マイクロタスクが「全部消化」されることを確認する

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// microtask-drain.js
// マイクロタスクキューは「全部」消化されてからマクロタスクに進む

console.log('1: script 開始');

setTimeout(function macro() {
  console.log('6: setTimeout (マクロタスク)');
}, 0);

Promise.resolve()
  .then(function micro1() {
    console.log('2: Promise.then #1');
    // マイクロタスクの中からさらにマイクロタスクを追加
    return Promise.resolve();
  })
  .then(function micro2() {
    console.log('3: Promise.then #2');
    queueMicrotask(function micro3() {
      console.log('4: queueMicrotask (micro2 の中から追加)');
    });
  })
  .then(function micro4() {
    console.log('5: Promise.then #3');
  });

console.log('--- script 終了 ---');
```

`node microtask-drain.js` で実行してください。

**実行結果:**

```
1: script 開始
--- script 終了 ---
2: Promise.then #1
3: Promise.then #2
4: queueMicrotask (micro2 の中から追加)
5: Promise.then #3
6: setTimeout (マクロタスク)
```

**なぜこうなるか:**

> マイクロタスクの中で新たなマイクロタスクを追加しても、そのマイクロタスクも「今のマイクロタスク消化フェーズ」で処理される。つまり、マイクロタスクキューが完全に空になるまで、マクロタスクキューには手をつけない。
>
> これは **危険でもある**。マイクロタスクが無限にマイクロタスクを生み出すと、マクロタスクに永遠に到達しない（＝ UI がフリーズする）。

**やってみよう（予測 → 検証）:**

> 以下のコードを実行したら何が起きるか予測してから実行してください。**注意: 無限ループになるので `Ctrl+C` で止める準備をしてから実行すること。**

```js
// microtask-infinite.js — 実行後すぐ Ctrl+C で停止
function infiniteMicrotask() {
  queueMicrotask(infiniteMicrotask);
}
infiniteMicrotask();
setTimeout(() => console.log('これは表示されるか？'), 0);
```

> `setTimeout` のコールバックは実行されましたか？ なぜですか？

---

## 3. 実行順序パズルで理解を深める

### Step 5: 複合パターンの実行順を予測する

以下のコードの出力順を **紙に書いてから** 実行してください。これは面接でよく出る典型パターンです。

```js
// execution-order-puzzle.js
// 面接頻出: 実行順序を正確に答えられるか

console.log('A');

setTimeout(function timer1() {
  console.log('B');
  Promise.resolve().then(function innerMicro() {
    console.log('C');
  });
}, 0);

Promise.resolve().then(function micro1() {
  console.log('D');
  setTimeout(function timer2() {
    console.log('E');
  }, 0);
});

setTimeout(function timer3() {
  console.log('F');
}, 0);

Promise.resolve().then(function micro2() {
  console.log('G');
});

console.log('H');
```

`node execution-order-puzzle.js` で実行してください。

**実行結果:**

```
A
H
D
G
B
C
F
E
```

**なぜこうなるか:**

> **フェーズ 1: 同期コード**
> - `A` 出力
> - `timer1` をマクロタスクキューへ登録
> - `micro1` をマイクロタスクキューへ登録
> - `timer3` をマクロタスクキューへ登録
> - `micro2` をマイクロタスクキューへ登録
> - `H` 出力
>
> **フェーズ 2: マイクロタスク消化**
> - `micro1` 実行 → `D` 出力、`timer2` をマクロタスクキューへ登録
> - `micro2` 実行 → `G` 出力
>
> **フェーズ 3: マクロタスク 1 つ目 (timer1)**
> - `B` 出力、`innerMicro` をマイクロタスクキューへ登録
> - → マイクロタスク消化: `C` 出力
>
> **フェーズ 4: マクロタスク 2 つ目 (timer3)**
> - `F` 出力
>
> **フェーズ 5: マクロタスク 3 つ目 (timer2)**
> - `E` 出力

### Step 6: async/await の実行順を予測する

`async/await` は Promise のシンタックスシュガーです。イベントループの観点でどう動くか確認しましょう。

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// async-await-order.js
// async/await をイベントループで分解する

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

`node async-await-order.js` で実行してください。

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

> React のコンポーネント内で `useEffect` のコールバックが実行されるタイミングも、イベントループと密接に関わっている。`useEffect` は描画後（マイクロタスク消化後）に実行される「マクロタスク的」なタイミング。一方 `useLayoutEffect` は描画前（マイクロタスクに近いタイミング）で実行される。この違いはイベントループの仕組みを知っていれば自然に理解できる。

**やってみよう（予測 → 検証）:**

> 以下のように `await` を 2 つに増やしたら、出力順はどうなるか予測してから実行してください。

```js
// double-await.js
async function doubleAwait() {
  console.log('A');
  await Promise.resolve();
  console.log('B');
  await Promise.resolve();
  console.log('C');
}

console.log('1');
doubleAwait();
Promise.resolve().then(() => console.log('D'));
console.log('2');
```

---

## 4. ブラウザ固有のイベントループを体験する

### Step 7: requestAnimationFrame の位置を確認する

以下の HTML ファイルを作成し、ブラウザで開いてください。**Node.js では動きません。**

```html
<!-- event-loop-browser.html -->
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>ブラウザのイベントループ</title>
</head>
<body>
  <h1>コンソールを開いて結果を確認してください</h1>
  <script>
    // ブラウザのイベントループにおける実行順序

    console.log('1: script 開始 (同期)');

    setTimeout(function macroTask() {
      console.log('5: setTimeout (マクロタスク)');
    }, 0);

    requestAnimationFrame(function rafCallback() {
      console.log('4: requestAnimationFrame (描画前)');
    });

    Promise.resolve().then(function microTask() {
      console.log('2: Promise.then (マイクロタスク)');
    });

    queueMicrotask(function anotherMicroTask() {
      console.log('3: queueMicrotask (マイクロタスク)');
    });

    console.log('--- script 終了 (同期) ---');
  </script>
</body>
</html>
```

ブラウザの DevTools コンソールで結果を確認してください。

**実行結果（典型的な順序）:**

```
1: script 開始 (同期)
--- script 終了 (同期) ---
2: Promise.then (マイクロタスク)
3: queueMicrotask (マイクロタスク)
4: requestAnimationFrame (描画前)
5: setTimeout (マクロタスク)
```

**なぜこうなるか:**

> ブラウザのイベントループは Node.js より構造が複雑で、以下の順序で処理される：
>
> ```
> 1. 同期コード実行
> 2. マイクロタスクキューをすべて消化
> 3. レンダリングが必要なら:
>    a. requestAnimationFrame コールバック実行
>    b. スタイル計算・レイアウト・描画
> 4. タスクキュー（マクロタスク）から 1 つ取り出して実行
> 5. → 2 に戻る
> ```
>
> `requestAnimationFrame` は「次の描画の直前」に呼ばれるため、マイクロタスクの後、マクロタスクの前に位置する（ただし描画フレームがスキップされる場合は順序が変わることもある）。

**React との比較:**

> React 18 の `createRoot` による Concurrent Mode では、レンダリングのスケジューリングに `MessageChannel`（マクロタスク）を使っている。React が `requestAnimationFrame` ではなくマクロタスクを選んだのは、rAF がディスプレイのリフレッシュレート（通常 16.6ms）に縛られるのに対し、マクロタスクはより早いタイミングで実行できるため。

### Step 8: Node.js 固有のキューを確認する

Node.js には `process.nextTick` と `setImmediate` という固有の API がある。これらの位置を確認しましょう。

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// node-specific-queues.js
// Node.js 固有のタイマー・マイクロタスク

console.log('1: script 開始');

setTimeout(function timeout() {
  console.log('6: setTimeout');
}, 0);

setImmediate(function immediate() {
  console.log('7: setImmediate');
});

process.nextTick(function nextTick() {
  console.log('3: process.nextTick');
});

Promise.resolve().then(function promiseThen() {
  console.log('4: Promise.then');
});

queueMicrotask(function microTask() {
  console.log('5: queueMicrotask');
});

console.log('2: script 終了');
```

`node node-specific-queues.js` で実行してください。

**実行結果:**

```
1: script 開始
2: script 終了
3: process.nextTick
4: Promise.then
5: queueMicrotask
6: setTimeout
7: setImmediate
```

**なぜこうなるか:**

> Node.js のイベントループでは、マイクロタスクがさらに細分化されている：
>
> ```
> 優先度 高 ──────────────────────── 低
>
> process.nextTick  >  Promise/queueMicrotask  >  setTimeout  >  setImmediate
> (nextTick キュー)    (マイクロタスクキュー)       (timers)      (check)
> ```
>
> `process.nextTick` は **マイクロタスクよりさらに優先度が高い**。Node.js のドキュメントでも `process.nextTick` は「マイクロタスクキューとは別のキュー」として扱われている。

**やってみよう（予測 → 検証）:**

> 以下のコードで `setTimeout` と `setImmediate` の順序は常に一定でしょうか？ 10 回実行して確認してみてください。

```js
// timer-vs-immediate.js
setTimeout(() => console.log('setTimeout'), 0);
setImmediate(() => console.log('setImmediate'));
```

> **ヒント:** トップレベルで呼んだ場合、順序は不定になることがある。なぜか考えてみてください。（`setTimeout(fn, 0)` の実際の最小遅延と、`poll` フェーズの到達タイミングに依存する）

---

## 5. 実践的なパターンと落とし穴

### Step 9: マイクロタスクで UI をブロックする失敗例

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
          counterEl.textContent = count; // DOM を更新しているが…
          queueMicrotask(microLoop);     // マイクロタスクで再帰
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

> React 18 の Concurrent Mode が解決する問題はまさにこれ。大きなレンダリングを小さなチャンクに分割し、各チャンクの間にブラウザに制御を戻す。内部的には `MessageChannel` を使って「マクロタスク単位でチャンクを実行 → ブラウザに描画を許可 → 次のチャンク」というパターンを採用している。`startTransition` で「優先度の低い更新」をマークするのも、このスケジューリングの仕組みの上に成り立っている。

### Step 10: 実行順序の総合問題に挑戦する

最後に、すべての知識を統合する問題です。**紙に出力順を書いてから** 実行してください。

```js
// final-challenge.js
// 総合問題: すべてのキューが混在するケース

async function asyncTask() {
  console.log('C');
  await Promise.resolve();
  console.log('F');
}

console.log('A');

setTimeout(() => {
  console.log('G');
  Promise.resolve().then(() => console.log('H'));
}, 0);

Promise.resolve().then(() => {
  console.log('D');
  setTimeout(() => console.log('I'), 0);
});

asyncTask();

queueMicrotask(() => console.log('E'));

console.log('B');
```

`node final-challenge.js` で実行してください。

**実行結果:**

```
A
C
B
D
F
E
G
H
I
```

**解説（フェーズごとに追う）:**

> **フェーズ 1: 同期コード**
> - `A` 出力
> - `setTimeout` → `G` のコールバックをマクロタスクキューへ
> - `Promise.resolve().then` → `D` のコールバックをマイクロタスクキューへ
> - `asyncTask()` 呼び出し → `C` 出力（`await` まで同期）。`await` 後の `F` をマイクロタスクキューへ
> - `queueMicrotask` → `E` のコールバックをマイクロタスクキューへ
> - `B` 出力
>
> **フェーズ 2: マイクロタスク消化**
> - `D` 出力（`setTimeout` で `I` をマクロタスクキューへ登録）
> - `F` 出力
> - `E` 出力
>
> **フェーズ 3: マクロタスク**
> - `G` 出力（`Promise.then` で `H` をマイクロタスクキューへ登録）
> - → マイクロタスク消化: `H` 出力
> - `I` 出力

---

## 理解チェック — 面接で聞かれたら

### Q1: 「JavaScript のイベントループを説明してください」

**模範回答（30 秒版）:**

> JavaScript はシングルスレッドで、1 つのコールスタックしか持ちません。イベントループは、コールスタックが空になったときにキューからタスクを取り出して実行する仕組みです。キューには優先度があり、マイクロタスク（Promise.then など）はマクロタスク（setTimeout など）より先に処理されます。マイクロタスクは全部消化してからマクロタスクに進むのが重要な特徴です。
>
> **英語キーフレーズ:** "The event loop continuously checks if the call stack is empty, then processes all microtasks before picking the next macrotask."

**深掘りされた場合（1 分版）:**

> イベントループの 1 サイクルは次のように動きます。まずコールスタック上の同期コードが実行されます。スタックが空になると、マイクロタスクキュー（Promise.then, queueMicrotask, MutationObserver）を **すべて** 消化します。マイクロタスクの中で追加されたマイクロタスクも同じフェーズで処理されます。マイクロタスクキューが空になったら、ブラウザの場合はレンダリングの機会があり（requestAnimationFrame はここ）、その後マクロタスクキュー（setTimeout, setInterval, I/O）から **1 つ** 取り出して実行します。そしてまたマイクロタスク消化に戻ります。Node.js ではさらに `process.nextTick` がマイクロタスクより優先され、libuv のフェーズ（timers → poll → check → close）という独自のサイクルがあります。
>
> **英語キーフレーズ:** "Microtasks are drained completely between each macrotask. This asymmetry is why Promise callbacks always run before setTimeout callbacks queued at the same time."

### Q2: 「`setTimeout(fn, 0)` は本当に 0ms 後に実行されますか？」

**模範回答（30 秒版）:**

> いいえ。`setTimeout(fn, 0)` は「最短で次のマクロタスクのタイミングで実行」という意味です。コールスタックが空になり、マイクロタスクがすべて消化された後に初めて実行されます。また、ブラウザには最小遅延（ネストが深い場合は 4ms）の制限があるため、実際には 0ms にはなりません。
>
> **英語キーフレーズ:** "`setTimeout(fn, 0)` doesn't mean 'execute in 0ms' — it means 'schedule as a macrotask, to be executed after the current synchronous code and all pending microtasks finish.'"

### Q3: 「マイクロタスクとマクロタスクの違いは何ですか？ なぜ区別されているのですか？」

**模範回答（30 秒版）:**

> マイクロタスク（Promise.then, queueMicrotask）は現在のタスクの終了直後に、全件消化されるまで実行されます。マクロタスク（setTimeout, I/O）はイベントループの 1 サイクルで 1 つずつ処理されます。区別する理由は、Promise チェーンのような「一連の非同期処理」を途中で他のタスクに割り込ませず、一貫して完了させたいから。一方で全部をマイクロタスクにしてしまうと UI 更新がブロックされるリスクがあるため、重い処理はマクロタスクに分割すべきです。
>
> **英語キーフレーズ:** "Microtasks provide a way to execute follow-up logic immediately after the current task completes, before the browser re-renders. Macrotasks yield control back to the event loop, allowing rendering and user input to be processed between tasks."

### Q4: 「React の `setState` のバッチ更新はイベントループとどう関係しますか？」

**模範回答（30 秒版）:**

> React 18 以降では、`setState` 呼び出しはデフォルトで自動バッチ処理されます。同じイベントハンドラ内の複数の `setState` は 1 回の再レンダリングにまとめられます。これはイベントループのマイクロタスクの仕組みと似ており、React は内部的にスケジューラを使って更新をキューに入れ、適切なタイミングでまとめて処理しています。React 18 では `setTimeout` やネイティブイベントハンドラ内の `setState` もバッチ処理される（React 17 ではされなかった）ようになりましたが、これは React のスケジューラがイベントループの仕組みをより巧みに活用するようになったためです。
>
> **英語キーフレーズ:** "React 18's automatic batching leverages the event loop by queueing state updates and flushing them in a single render pass, similar to how microtasks are batched before yielding to the browser."
