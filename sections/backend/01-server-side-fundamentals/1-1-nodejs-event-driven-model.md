# Node.js のイベント駆動モデル — ハンズオン学習ガイド

> Node.js が「シングルスレッドなのに大量の同時接続を捌ける」理由を、イベントループ・ノンブロッキング I/O・libuv のスレッドプールを実際に手を動かして体感する。

---

## 0. この教材のねらい

完成時に以下ができるようになる：

- Node.js のイベントループが「どの順序でタスクを拾うか」を自分で予測できる
- ブロッキング処理とノンブロッキング処理の違いを、レイテンシの数値として計測できる
- `setTimeout` / `setImmediate` / `process.nextTick` / `Promise.then` の実行順序を説明できる
- CPU バウンドな処理がイベントループを止める様子を再現し、Worker Threads で回避できる
- 面接で「なぜ Node.js は C10K 問題に強いのか」を 30 秒で答えられる

**前提知識:**
- JavaScript の基本（関数、コールバック、Promise、async/await）
- ターミナル操作の基本

**必要なツール:**
- Node.js 20 以上（`node -v` で確認）
- ターミナル 2 枚（片方でサーバー起動、片方で curl）

```bash
# 作業ディレクトリを作る
mkdir nodejs-event-loop && cd nodejs-event-loop
npm init -y
```

---

## 1. シングルスレッドで複数接続を捌く体験

### Step 1: まず同期的な「ブロッキング」サーバーを書く

以下のコードをそのまま `blocking-server.js` として保存してください。

```js
// blocking-server.js
// わざと「重い同期処理」を入れたサーバー
// 1 リクエストごとに 3 秒間 CPU をブン回す
const http = require('http');

function heavyBlockingWork() {
  // 3 秒間、同期的にループを回す（I/O ではなく CPU を占有する）
  const end = Date.now() + 3000;
  while (Date.now() < end) {
    // ひたすら待つ
  }
}

const server = http.createServer((req, res) => {
  console.log('received:', req.url, 'at', new Date().toISOString());
  heavyBlockingWork(); // ← ここでイベントループが完全に止まる
  res.end('done\n');
});

server.listen(3000, () => {
  console.log('blocking server listening on :3000');
});
```

ターミナル 1 で起動：

```bash
node blocking-server.js
```

ターミナル 2 で、**2 本同時に** curl を叩きます：

```bash
# & でバックグラウンド起動して並列に投げる
time curl -s localhost:3000/a & time curl -s localhost:3000/b & wait
```

**実行結果:**

```
done
done
curl -s localhost:3000/a  0.01s user 0.01s system 0% cpu 3.012 total
curl -s localhost:3000/b  0.01s user 0.01s system 0% cpu 6.021 total
```

**なぜこうなるか:**

> Node.js はリクエストハンドラを **シングルスレッド** で実行する。1 本目が `while` ループで CPU を占有している間、2 本目のリクエストはイベントループのキューで待たされる。結果、2 本目は合計 6 秒かかる。

```
時刻:  0s ─────────── 3s ─────────── 6s
Req A: [■■■■■ CPU ■■■■■] done
Req B: [   待機   ][■■■■■ CPU ■■■■■] done
```

**やってみよう（予測 → 検証）:**

> curl を **3 本** 同時に投げたら、一番遅いものは何秒になるでしょうか？予測してから実行してみてください。

---

### Step 2: ノンブロッキング I/O に書き換える

同じ「3 秒待つ」でも、CPU を占有せず I/O として扱うとどうなるか見ます。

```js
// nonblocking-server.js
const http = require('http');

function nonBlockingWait() {
  // setTimeout はタイマーを libuv に登録するだけ
  // その間イベントループは他の仕事を進められる
  return new Promise((resolve) => setTimeout(resolve, 3000));
}

const server = http.createServer(async (req, res) => {
  console.log('received:', req.url, 'at', new Date().toISOString());
  await nonBlockingWait(); // ← ここで他のリクエストを処理できる
  res.end('done\n');
});

server.listen(3000, () => {
  console.log('non-blocking server listening on :3000');
});
```

一度ターミナル 1 の blocking サーバーを `Ctrl+C` で止めてから：

```bash
node nonblocking-server.js
```

ターミナル 2 で同じく 2 本並列：

```bash
time curl -s localhost:3000/a & time curl -s localhost:3000/b & wait
```

**実行結果:**

```
done
done
curl -s localhost:3000/a  0.01s user 0.01s system 0% cpu 3.015 total
curl -s localhost:3000/b  0.01s user 0.01s system 0% cpu 3.016 total
```

**なぜこうなるか:**

> `setTimeout` は JavaScript スレッドを占有せず、libuv のタイマーに「3 秒後に呼んでね」と登録して即座に制御を返す。イベントループは空いたので次のリクエストを受け取り、両方のタイマーが並行に進む。結果、2 本とも約 3 秒で終わる。

```
時刻:  0s ─────────── 3s
Req A: [register timer ─── wait ───][resolve] done
Req B: [register timer ─ wait ─────][resolve] done
         ↑ 両方同時に待てる
```

**実務ではどうするか:**

> 実プロダクトでは「3 秒待つ」ではなく DB クエリ・外部 API 呼び出し・ファイル読み込みが I/O に相当する。`pg` / `redis` / `fetch` などのライブラリはすべて内部でノンブロッキング I/O を使うので、同じ仕組みで数千接続を 1 プロセスで捌ける。

---

## 2. イベントループのフェーズを体感する

### Step 3: 実行順序をクイズで確かめる

以下のコードを `phases.js` として保存してください。**実行する前に出力順を予測してから** 動かしてみます。

```js
// phases.js
// 「この順番で出力される」を予測してから node phases.js を実行しよう

console.log('1: sync start');

setTimeout(() => console.log('2: setTimeout 0ms'), 0);

setImmediate(() => console.log('3: setImmediate'));

Promise.resolve().then(() => console.log('4: promise.then (microtask)'));

process.nextTick(() => console.log('5: process.nextTick'));

console.log('6: sync end');
```

予測を書いてから：

```bash
node phases.js
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
> 2. **microtask キュー** — 同期コード完了直後に全部吐き出す。`process.nextTick` が最優先、次に `Promise.then` などの microtask（5 → 4）
> 3. **Timers フェーズ** — `setTimeout` / `setInterval`（2）
> 4. **Check フェーズ** — `setImmediate`（3）

```
┌─────────────┐
│  sync code  │ 1, 6
└──────┬──────┘
       ▼
┌─────────────────────────┐
│ microtask queue         │
│  ├─ process.nextTick(5) │ ← 最優先
│  └─ Promise.then    (4) │
└──────┬──────────────────┘
       ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ Timers        │────▶│ Poll (I/O)    │────▶│ Check         │────▶│ Close         │
│ setTimeout(2) │     │               │     │ setImmediate(3)│     │               │
└───────────────┘     └───────────────┘     └───────────────┘     └───────────────┘
       ▲                                                                │
       └────────────────────────── loop ────────────────────────────────┘
```

**実務ではどうするか:**

> 「`setTimeout(fn, 0)` なのになぜ即実行されない？」という質問は面接の頻出。答えは「最速でも Timers フェーズまで待つ」であり、microtask の方が必ず先に走る。`process.nextTick` の乱用は microtask キューが枯渇せず I/O が飢える原因になるので、ライブラリ作者以外は使わない方が良い。

**やってみよう（予測 → 検証）:**

> 上のコードで `setTimeout(..., 0)` と `setImmediate(...)` を I/O コールバック内（例えば `fs.readFile` の中）に入れると、今度は **必ず `setImmediate` が先** になります。なぜでしょうか？試してから次に進んでください。
>
> ヒント：I/O コールバックは Poll フェーズで実行される。Poll の次は Check（setImmediate）、その次のループで Timers（setTimeout）。

---

### Step 4: I/O コールバック内での順序を確かめる

```js
// io-order.js
const fs = require('fs');

fs.readFile(__filename, () => {
  // ここは Poll フェーズ内で呼ばれる
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
```

```bash
node io-order.js
```

**実行結果:**

```
immediate
timeout
```

**なぜこうなるか:**

> Poll フェーズで I/O コールバックが走った直後の順序は **Check（setImmediate）→ 次ループの Timers（setTimeout）** なので、`setImmediate` が必ず先になる。トップレベルで書いた場合は競合状態になるためしばしば `setTimeout` が先に見えるが、I/O 内では決定論的になる。

---

## 3. CPU バウンド処理でイベントループを止めてみる

### Step 5: 遅延を計測するプローブを仕込む

Node.js 本体が「今イベントループがどれだけ遅れているか」を測るための仕込みをします。

```js
// lag-probe.js
// 10ms ごとに setTimeout を仕掛け、本来 10ms で戻ってくるはずの時間が
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

```bash
node lag-probe.js
```

**実行結果:**

```
event loop lag: 0 ms
event loop lag: 1 ms
event loop lag: 0 ms
...
>>> starting heavy CPU work
event loop lag: 1998 ms    ← 2 秒分まるごと遅れた
>>> heavy work done
event loop lag: 0 ms
event loop lag: 0 ms
```

**なぜこうなるか:**

> `while` ループが走っている 2 秒間、イベントループは 1 ステップも進めない。タイマーは登録されているが発火できない。この状態ではサーバーは一切レスポンスを返せず、ヘルスチェックも落ちる。

**実務ではどうするか:**

> 本番では `perf_hooks.monitorEventLoopDelay()` でラグを計測し、Datadog / Prometheus に送ってアラートを張る。50ms を超え始めたら危険信号。Express / Fastify は健全でも、同期的な JSON.stringify で巨大オブジェクトを変換しているだけで数百 ms 止まることがある。

---

### Step 6: Worker Threads で CPU 仕事をオフロードする

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
      // 素朴なフィボナッチ（計算量が指数的に膨らむ CPU バウンド処理）
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

```bash
node worker-main.js
```

別ターミナルで、重い処理を投げた **直後に** ping を投げます：

```bash
time curl -s localhost:3000/heavy & sleep 0.1 && time curl -s localhost:3000/ping & wait
```

**実行結果:**

```
pong                               ← ping は即座に返る！
curl -s localhost:3000/ping  0% cpu 0.115 total
fib(42) = 267914296
curl -s localhost:3000/heavy  0% cpu 1.840 total
```

**なぜこうなるか:**

> `fib(42)` の計算は別スレッド（OS レベルの別スレッド）で走るので、メインのイベントループは `/ping` を即座に処理できる。`await` で Worker の完了を待つが、これはノンブロッキングなので他のリクエストの邪魔をしない。

```
Main thread (event loop):  [/heavy受信→worker起動→/ping処理→pong返却]
Worker thread:                        [fib(42) 計算 ...............]
                                                                    ↓
Main thread:                                      [Workerからメッセージ→/heavy返却]
```

**実務ではどうするか:**

> 画像リサイズ、PDF 生成、暗号化、パースヘビーな JSON 加工など CPU バウンドな処理は Worker Threads または別サービス（ジョブキュー + ワーカープロセス）に逃がす。毎リクエストで Worker を生成するのはオーバーヘッドが大きいので、実務では `piscina` などのワーカープールライブラリを使う。

**やってみよう（予測 → 検証）:**

> `worker-main.js` の `runHeavyInWorker` を呼ばずに、ハンドラ内で直接 `fib(42)` を実行するコードに書き換えてみてください。/heavy を投げた直後の /ping はどうなるでしょうか？

---

## 4. 理解チェック — 面接で聞かれたら

### Q1: "Node.js はシングルスレッドなのに、なぜ数千接続を同時に捌けるのですか？"

**模範回答（30 秒版）:**

> Node.js は JavaScript の実行こそシングルスレッドですが、I/O 処理は libuv というライブラリが OS のノンブロッキング I/O 機能（epoll / kqueue）を使って非同期に処理します。JavaScript コードが待っている時間はイベントループが他のリクエストを拾えるので、CPU がアイドルにならず、1 プロセスで大量の同時接続を効率的に処理できます。これが C10K 問題に Node.js が強い理由です。

**深掘りされた場合（1 分版）:**

> 具体的にはイベントループが Timers → Pending → Poll → Check → Close の順に各フェーズを回り、Poll フェーズで OS から I/O 完了通知を受け取ってコールバックを実行します。ファイル I/O のように OS が真の非同期 API を提供しないものは libuv の内部スレッドプール（デフォルト 4 スレッド）で処理されます。ただし CPU バウンドな処理は別で、これを同期的に書くとイベントループ全体が止まります。そのため画像処理や暗号化は Worker Threads に逃がすか、ジョブキューで別プロセスに任せます。

**英語キーフレーズ:**
> - "Node.js uses non-blocking I/O via libuv and an event loop to multiplex many connections on a single thread."
> - "CPU-bound work blocks the event loop, so we offload it to worker threads or a job queue."
> - "The thread pool in libuv handles file system operations and DNS lookups that the OS doesn't expose asynchronously."

---

### Q2: "`setTimeout(fn, 0)` と `setImmediate(fn)` と `process.nextTick(fn)` の違いは？"

**模範回答（30 秒版）:**

> 実行タイミングが異なります。`process.nextTick` は現在の操作の直後、どのフェーズよりも先に実行されます。Promise の `.then` と同じ microtask キューですが、`nextTick` の方が優先されます。`setTimeout(fn, 0)` は Timers フェーズで、`setImmediate` は Check フェーズで実行されます。I/O コールバック内では `setImmediate` が先、トップレベルでは順序が非決定的です。

**英語キーフレーズ:**
> - "`process.nextTick` runs before any other I/O event fires — use it sparingly."
> - "`setImmediate` fires in the check phase, right after poll."
> - "Microtasks drain completely between each macrotask."

---

### Q3: "本番環境でイベントループが詰まっているかどうか、どう検知しますか？"

**模範回答（1 分版）:**

> Node.js の `perf_hooks` モジュールが提供する `monitorEventLoopDelay` を使うと、ヒストグラムで遅延を計測できます。これを定期的にメトリクスシステム（Prometheus、Datadog など）に送信し、p99 が 50〜100ms を超えたらアラートを鳴らします。原因としては同期的な巨大 JSON の処理、正規表現の ReDoS、同期的なファイル I/O などが典型です。APM ツールであれば継続プロファイラで「どの関数がイベントループを止めているか」を直接特定できます。

**英語キーフレーズ:**
> - "We track event loop lag as a first-class SLI."
> - "A common culprit is synchronous JSON.stringify over large payloads."
> - "We use a continuous profiler to pinpoint the blocking call site."

---

### Q4: "Node.js の Worker Threads と Cluster モジュールの使い分けは？"

**模範回答（30 秒版）:**

> Worker Threads は CPU バウンドな計算を同一プロセス内の別スレッドに逃がす仕組みで、メモリを `SharedArrayBuffer` で共有できます。Cluster は同じサーバーを複数プロセスで並列起動し、マスターがポートを共有してリクエストを分散する仕組みです。リクエスト処理全体をマルチコアで捌きたいなら Cluster（または本番では PM2・Kubernetes のレプリカ）、特定の重い計算だけを逃がしたいなら Worker Threads を使います。

**英語キーフレーズ:**
> - "Cluster scales request handling across cores; worker threads offload specific CPU tasks."
> - "In containerized deployments, we usually skip cluster and run one process per pod, scaling horizontally."

---

## 次のステップ

- `1-2. HTTP プロトコル詳解` へ進み、Node.js の `http` モジュールが内部で何をしているかを覗く
- 余力があれば `clinic.js` や `0x` を使って実サーバーのフレームグラフを撮ってみる
- `libuv` のソースを軽く読むと「なぜ Timers → Poll → Check の順なのか」が腑に落ちる
