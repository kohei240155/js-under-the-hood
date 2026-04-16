# Node.js のイベント駆動モデル — ハンズオン学習ガイド

> Node.js が「シングルスレッドなのに大量の同時接続を捌ける」理由を、イベントループ・ノンブロッキング I/O・libuv のスレッドプールを実際に手を動かして体感する。

---

## 0. この教材のねらい

**想定所要時間: 約 60 分**

- ブロッキング処理とノンブロッキング処理の違いを、レイテンシの数値として計測できる
- `setTimeout` / `setImmediate` / `process.nextTick` / `Promise.then` の実行順序を説明できる
- CPU バウンドな処理がイベントループを止める様子を再現し、Worker Threads で回避できる
- 面接で「なぜ Node.js は C10K 問題に強いのか」を 30 秒で答えられる

**前提知識:** JavaScript の基本（コールバック、Promise、async/await）、ターミナル操作

**必要なツール:** Node.js 20 以上、ターミナル 2 枚（サーバー起動用と curl 用）

```bash
mkdir nodejs-event-loop && cd nodejs-event-loop
npm init -y
```

---

## 1. シングルスレッドで複数接続を捌く

### Step 1: ブロッキング vs ノンブロッキングのレイテンシを比較する

まず、ブロッキング版を `blocking-server.js` として保存してください。

```js
// blocking-server.js
// 1 リクエストごとに 3 秒間 CPU をブン回す
const http = require('http');

function heavyBlockingWork() {
  const end = Date.now() + 3000;
  while (Date.now() < end) { /* CPU 占有 */ }
}

const server = http.createServer((req, res) => {
  console.log('received:', req.url, 'at', new Date().toISOString());
  heavyBlockingWork(); // ← イベントループが完全に止まる
  res.end('done\n');
});

server.listen(3000, () => console.log('blocking server on :3000'));
```

ターミナル 1 で `node blocking-server.js`、ターミナル 2 で並列に curl：

```bash
time curl -s localhost:3000/a & time curl -s localhost:3000/b & wait
```

**実行結果（ブロッキング）:**

```
curl -s localhost:3000/a  0% cpu 3.012 total
curl -s localhost:3000/b  0% cpu 6.021 total  ← 2本目は待たされる
```

次に、ノンブロッキング版を `nonblocking-server.js` として保存して `Ctrl+C` した後に起動：

```js
// nonblocking-server.js
const http = require('http');

function nonBlockingWait() {
  // setTimeout はタイマーを libuv に登録するだけで即座に制御を返す
  return new Promise((resolve) => setTimeout(resolve, 3000));
}

const server = http.createServer(async (req, res) => {
  console.log('received:', req.url, 'at', new Date().toISOString());
  await nonBlockingWait(); // ← 他のリクエストを処理できる
  res.end('done\n');
});

server.listen(3000, () => console.log('non-blocking server on :3000'));
```

同じ並列 curl を実行：

**実行結果（ノンブロッキング）:**

```
curl -s localhost:3000/a  0% cpu 3.015 total
curl -s localhost:3000/b  0% cpu 3.016 total  ← 両方とも約3秒で終わる
```

**なぜこうなるか:**

> Node.js はリクエストハンドラを **シングルスレッド** で実行する。`while` ループは CPU を独占するため、2 本目はキューで待たされる（合計 6 秒）。一方 `setTimeout` は libuv のタイマーに「3 秒後に呼んでね」と登録して即制御を返すので、両方のタイマーが並行に進む（両方 3 秒）。
>
> ```
> ブロッキング:                       ノンブロッキング:
> Req A: [■■■■■ CPU ■■■■■] done    Req A: [register timer ─ wait ─][resolve] done
> Req B: [   待機 ][■■■■■ CPU ■■■■■] Req B: [register timer ─ wait ─][resolve] done
>        0s        3s         6s              0s              3s
> ```

**実務ではどうするか:**

> 実プロダクトでは「3 秒待つ」ではなく DB クエリ・外部 API・ファイル読み込みが I/O に相当する。`pg` / `redis` / `fetch` などのライブラリはすべて内部でノンブロッキング I/O を使うので、同じ仕組みで数千接続を 1 プロセスで捌ける。

---

## 2. イベントループのフェーズ

### Step 2: 実行順序を予測する

以下を `phases.js` として保存し、**実行する前に出力順を予測してから** 動かしてください。

```js
// phases.js
console.log('1: sync start');

setTimeout(() => console.log('2: setTimeout 0ms'), 0);
setImmediate(() => console.log('3: setImmediate'));
Promise.resolve().then(() => console.log('4: promise.then (microtask)'));
process.nextTick(() => console.log('5: process.nextTick'));

console.log('6: sync end');
```

**実行結果:**

```
1: sync start
6: sync end
5: process.nextTick
4: promise.then (microtask)
2: setTimeout 0ms
3: setImmediate
```

**なぜこうなるか:**

> Node.js のイベントループには厳密な優先順位がある：
>
> 1. **同期コード** — 最初に一気に流れる（1, 6）
> 2. **microtask キュー** — 同期コード完了直後に全部吐き出す。`process.nextTick` が最優先、次に `Promise.then`（5 → 4）
> 3. **Timers フェーズ** — `setTimeout` / `setInterval`（2）
> 4. **Check フェーズ** — `setImmediate`（3）
>
> ```
> ┌─────────────┐
> │  sync code  │ 1, 6
> └──────┬──────┘
>        ▼
> ┌─────────────────────────┐
> │ microtask queue         │
> │  ├─ process.nextTick(5) │ ← 最優先
> │  └─ Promise.then    (4) │
> └──────┬──────────────────┘
>        ▼
> ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌─────────┐
> │ Timers  │───▶│ Poll(I/O)│───▶│ Check   │───▶│ Close   │
> │ setT(2) │    │          │    │ setI(3) │    │         │
> └─────────┘    └──────────┘    └─────────┘    └─────────┘
>      ▲                                              │
>      └────────────── loop ─────────────────────────┘
> ```

---

## 3. CPU バウンド処理を Worker Threads に逃がす

### Step 3: イベントループラグを計測する

「いまイベントループがどれだけ遅れているか」を計測する仕込みを入れます。

```js
// lag-probe.js
// 10ms ごとに setTimeout を仕掛け、本来 10ms で戻るはずの時間が
// 実際には何 ms かかったかを記録する = イベントループラグ
let last = Date.now();
setInterval(() => {
  const now = Date.now();
  const lag = now - last - 10;
  console.log('event loop lag:', lag, 'ms');
  last = now;
}, 10);

// 3 秒待ってからわざと CPU を 2 秒間占有する
setTimeout(() => {
  console.log('>>> starting heavy CPU work');
  const end = Date.now() + 2000;
  while (Date.now() < end) {}
  console.log('>>> heavy work done');
}, 3000);
```

**実行結果:**

```
event loop lag: 0 ms
event loop lag: 1 ms
...
>>> starting heavy CPU work
event loop lag: 1998 ms    ← 2 秒分まるごと遅れた
>>> heavy work done
event loop lag: 0 ms
```

**なぜこうなるか:**

> `while` ループが走っている 2 秒間、イベントループは 1 ステップも進めない。タイマーは登録されているが発火できない。この状態ではサーバーは一切レスポンスを返せず、ヘルスチェックも落ちる。
>
> 本番では `perf_hooks.monitorEventLoopDelay()` でラグを計測し、Datadog / Prometheus に送ってアラートを張る。50ms を超え始めたら危険信号。

---

### Step 4: Worker Threads で CPU 仕事をオフロードする

CPU バウンドな処理を別スレッドに逃がして、メインのイベントループを守ります。

```js
// worker-main.js
const { Worker } = require('worker_threads');
const http = require('http');

function runHeavyInWorker(n) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(
      `
      const { parentPort, workerData } = require('worker_threads');
      // 素朴なフィボナッチ（指数的に膨らむ CPU バウンド処理）
      function fib(n) { return n < 2 ? n : fib(n - 1) + fib(n - 2); }
      parentPort.postMessage(fib(workerData));
      `,
      { eval: true, workerData: n }
    );
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}

const server = http.createServer(async (req, res) => {
  if (req.url === '/heavy') {
    const result = await runHeavyInWorker(42);
    res.end('fib(42) = ' + result + '\n');
  } else if (req.url === '/ping') {
    res.end('pong\n');
  } else {
    res.end('ok\n');
  }
});

server.listen(3000, () => console.log('worker server on :3000'));
```

別ターミナルで、重い処理を投げた **直後に** ping を投げます：

```bash
time curl -s localhost:3000/heavy & sleep 0.1 && time curl -s localhost:3000/ping & wait
```

**実行結果:**

```
pong                         ← ping は即座に返る！
curl -s localhost:3000/ping  0% cpu 0.115 total
fib(42) = 267914296
curl -s localhost:3000/heavy 0% cpu 1.840 total
```

**なぜこうなるか:**

> `fib(42)` の計算は OS レベルの別スレッドで走るので、メインのイベントループは `/ping` を即座に処理できる。`await` で Worker の完了を待つが、これはノンブロッキングなので他のリクエストの邪魔をしない。
>
> ```
> Main:   [/heavy受信→worker起動→/ping処理→pong返却→Worker結果受信→/heavy返却]
> Worker:                    [fib(42) 計算 ...............]
> ```

**実務ではどうするか:**

> 画像リサイズ、PDF 生成、暗号化、パースヘビーな JSON 加工など CPU バウンドな処理は Worker Threads または別サービス（ジョブキュー + ワーカープロセス）に逃がす。毎リクエストで Worker を生成するのはオーバーヘッドが大きいので、実務では `piscina` などのワーカープールライブラリを使う。

---

## 発展課題（任意）

時間に余裕があれば挑戦してください。

### 課題 1: 並列 curl を3本に増やす
> Step 1 のブロッキング版で curl を **3 本** 同時に投げたら、一番遅いものは何秒になる？ ノンブロッキング版ではどうなる？ 予測してから実行してみよう。

### 課題 2: I/O コールバック内での setTimeout vs setImmediate
> 以下のコードでは **必ず `setImmediate` が先** になります。なぜか考えてみよう。ヒント：I/O コールバックは Poll フェーズで実行され、Poll の次は Check（setImmediate）、その次のループで Timers（setTimeout）。
>
> ```js
> const fs = require('fs');
> fs.readFile(__filename, () => {
>   setTimeout(() => console.log('timeout'), 0);
>   setImmediate(() => console.log('immediate'));
> });
> ```

### 課題 3: Worker を外して同じことをすると…
> Step 4 の `runHeavyInWorker` を呼ばずに、ハンドラ内で直接 `fib(42)` を実行するコードに書き換えてみよう。`/heavy` を投げた直後の `/ping` はどうなる？

---

## 理解チェック — 面接で聞かれたら

### Q1: "Node.js はシングルスレッドなのに、なぜ数千接続を同時に捌けるのですか？"

**模範回答（30 秒版）:**

> Node.js は JavaScript の実行こそシングルスレッドですが、I/O 処理は libuv というライブラリが OS のノンブロッキング I/O 機能（epoll / kqueue）を使って非同期に処理します。JavaScript コードが待っている時間はイベントループが他のリクエストを拾えるので、CPU がアイドルにならず、1 プロセスで大量の同時接続を効率的に処理できます。これが C10K 問題に Node.js が強い理由です。

### Q2: "`setTimeout(fn, 0)` と `setImmediate(fn)` と `process.nextTick(fn)` の違いは？"

**模範回答（30 秒版）:**

> 実行タイミングが異なります。`process.nextTick` は現在の操作の直後、どのフェーズよりも先に実行されます。`Promise.then` も microtask キューですが、`nextTick` の方が優先されます。`setTimeout(fn, 0)` は Timers フェーズで、`setImmediate` は Check フェーズで実行されます。I/O コールバック内では `setImmediate` が先、トップレベルでは順序が非決定的です。
