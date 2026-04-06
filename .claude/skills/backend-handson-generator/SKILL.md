---
name: backend-handson-generator
description: >
  バックエンド・API設計・システムデザインの学習トピック・目次・セクション見出しを受け取り、写経するだけで身になるハンズオン形式の学習教材を生成するスキル。
  以下のようなトリガーで必ず使用すること：
  ユーザーがNode.js/Express/PostgreSQL/Redis/MongoDB/Docker/REST API/GraphQL/認証/システムデザイン/マイクロサービス/メッセージキュー/キャッシュ/データベース設計/スケーラビリティ関連の「学習トピック」「目次」「セクション見出し」「カリキュラム項目」を入力して「教材を作って」「ハンズオンにして」「写経できるようにして」「学習ガイドを作って」「hands-on」「教材生成」などと言った場合。
  短い一行のトピック（例：「JWT認証」「CAP定理」「レート制限」）でも、長い目次リストでも対応する。
  バックエンド/インフラ/分散システム関連のプログラミング学習教材の生成リクエスト全般に使う。英語・日本語どちらの入力にも対応。
  フロントエンド/JavaScript/React のトピックには使わない（それらは frontend-handson-generator を使う）。
---

# バックエンド・API設計・システムデザイン ハンズオン教材ジェネレーター

ユーザーが学習トピックや目次を入力したら、**写経するだけで概念が身につく**ハンズオン形式の教材を生成する。

---

## 教材設計の原則

この教材は「読む教材」ではなく「手を動かす教材」。以下の原則を守る：

1. **コードファースト** — 説明の前にまずコードを見せる。理論は後から補足する
2. **段階的に構築** — 最初から完成形を見せず、Step 1 → Step 2 → ... と積み上げる
3. **実務との対比** — 「実際のプロダクションではどうするか」「フレームワーク（Express / Prisma / etc.）が裏側でやっていること」の対比を随所に入れる
4. **なぜそうなるか** — コードの動作結果だけでなく「なぜこの設計判断をするのか」のメンタルモデルを言語化する
5. **失敗を体験させる** — 正しいコードだけでなく、わざと壊れるコード（セキュリティホール・パフォーマンス劣化・データ不整合）を書かせて「何が起きるか予測 → 実行 → 答え合わせ」のサイクルを回す
6. **面接で説明できるレベル** — 各セクションの最後に「面接でこう聞かれたらこう答える」テンプレートを置く
7. **図解を積極的に使う** — システムデザインでは ASCII アートやコンポーネント図が理解の鍵になる。アーキテクチャ図・シーケンス図・データフロー図を必ず含める

---

## 出力フォーマット

教材は **Markdown ファイル** (.md) として生成する。以下の構造を必ず守る：

### 全体構造

```
# [トピック名] — ハンズオン学習ガイド

> 一行の概要説明（何を学ぶか、なぜ重要か）

---

## 0. この教材のねらい
- 完成時に何ができるようになるか（3〜5 個の箇条書き）
- 前提知識（必要な事前知識を明記）
- 必要なツール（Node.js / Docker / PostgreSQL / Redis など）

## 1. [最初のサブトピック]
### Step 1: [動詞で始まる見出し]
### Step 2: ...

## 2. [次のサブトピック]
...

## N. 理解チェック — 面接で聞かれたら
```

### 各 Step の内部構造

各 Step は以下の要素を含める。順序を守る：

````markdown
### Step X: [動詞で始まるタイトル]（例：「インデックスの効果を計測する」）

まず、以下のコードを **そのまま写経して** 実行してください。

```js
// ここに写経するコード（Node.js で実行するコード）
// コメントで「何をしているか」を日本語で書く
```

（SQL / Docker / curl / シェルコマンドの場合）

```sql
-- ここに写経する SQL
-- コメントで「何をしているか」を日本語で書く
```

```bash
# ここに写経するシェルコマンド
# コメントで「何をしているか」を日本語で書く
```
````

**実行結果:**

```
期待される出力をここに書く
```

**なぜこうなるか:**

> 1〜3 文で仕組みを説明。アーキテクチャ図が有効な場合は ASCII アートを使う。

**実務ではどうするか:**

> 本番環境ではこの部分は ○○ ライブラリ / サービスが担当する。手動で書いた理由は △△ を理解するため。
> （該当しない場合はこのセクションを省略）

**やってみよう（予測 → 検証）:**

> 上のコードの ○○ を △△ に変えたら、出力はどうなるでしょうか？
> まず予測を書いてから実行してみてください。

````

### システムデザイン問題専用のフォーマット

システムデザイン（カリキュラム セクション 5）のトピックでは、上記 Step 構造に加えて以下のフォーマットを使う：

````markdown
### Step X: 要件を整理する

**機能要件:**
1. ...
2. ...

**非機能要件:**
- DAU: ○○
- QPS (読み取り): ○○
- QPS (書き込み): ○○
- レイテンシ: ○○
- データ保持期間: ○○

### Step Y: 概算する（Back-of-the-Envelope）

```
QPS = DAU × (1日あたりの平均リクエスト数) / 86,400
    = ○○ × ○○ / 86,400
    = ○○ QPS

ストレージ = 1日の新規データ量 × 保持期間
           = ○○ KB × ○○ × 365日 × 5年
           = ○○ TB
```

### Step Z: ハイレベル設計を描く

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  Client  │────▶│ Load Balancer│────▶│ API Server│
└──────────┘     └──────────────┘     └─────┬─────┘
                                            │
                                      ┌─────▼─────┐
                                      │   Cache    │
                                      │  (Redis)   │
                                      └─────┬─────┘
                                            │
                                      ┌─────▼─────┐
                                      │ Database   │
                                      │ (PostgreSQL)│
                                      └───────────┘
```

> 各コンポーネントの役割を 1〜2 行で説明する

### Step W: 深掘り — [特定のコンポーネント]を詳細設計する

（ここでコードやスキーマを実際に書く）
````

---

## コード品質のルール

- すべてのコードブロックに **実行可能なコード** を書く（擬似コード禁止）
- Node.js で実行できるコードはそのまま `node` で実行可能にする
- SQL は PostgreSQL 構文で書く（MySQL との違いがある場合はコメントで補足）
- Docker / docker-compose の設定はそのまま `docker compose up` で動く完全なファイルを提供する
- curl コマンドはそのままターミナルに貼って実行可能にする
- 変数名・関数名・テーブル名・カラム名は **意味のある英語名** を使う（`a`, `b`, `x` は禁止。`userId`, `orderTotal`, `fetchProducts` など）
- コメントは日本語で書く
- `console.log` の出力には必ず **ラベル** を付ける（`console.log('result:', value)` のように）
- SQL の出力例は整形されたテーブル形式で書く
- 環境変数は `.env.example` として提供し、実際のシークレットは含めない

---

## トピックの深さの判定

ユーザーが入力したトピックの粒度に応じて教材の深さを調整する：

**細かいトピック**（例：「JWT認証」「インデックス最適化」「Token Bucket」「CAP定理」）
→ 1 トピックで 5〜8 Step の詳細教材を作る

**中程度のトピック**（例：「2-3. インデックスとクエリ最適化」のようなセクション見出し）
→ 3〜5 Step の教材を作る

**大きなトピック**（例：「2. データベース設計とクエリ最適化」のようなチャプター見出し）
→ 各サブトピックごとに 2〜3 Step + 総合演習の構成にする

**システムデザイン問題**（例：「URL短縮サービスを設計して」）
→ 要件整理 → 概算 → ハイレベル設計 → 深掘り 2〜3 箇所の構成にする。コードとアーキテクチャ図を必ず含める

**複数トピックのリスト**（例：目次全体を貼り付けた場合）
→ 1 つずつ順番に教材を生成する。1 回のレスポンスでは **1 セクション分** を生成し、「次のセクションに進みますか？」と確認する

---

## 面接対策セクションのフォーマット

各教材の最後に必ず以下を含める：

````markdown
## 理解チェック — 面接で聞かれたら

### Q1: [想定される面接質問]
**模範回答（30 秒版）:**
> 簡潔な回答をここに書く

**深掘りされた場合（1 分版）:**
> 具体例やコード・図を交えた詳細回答

**英語キーフレーズ:**
> - "The trade-off here is between consistency and availability..."
> - "I would use a write-ahead log to ensure durability..."

### Q2: ...
````

回答は **北米テック企業のバックエンド / フルスタック面接** を想定し、英語で答える場合のキーワード・フレーズも併記する。

### システムデザイン面接のフォローアップ質問

システムデザイン問題の教材では、面接でよくある深掘り質問も含める：

````markdown
### よくあるフォローアップ質問

**Q: "What happens when [component X] goes down?"**
> 回答テンプレート

**Q: "How would you scale this to 10x traffic?"**
> 回答テンプレート

**Q: "What are the trade-offs of your design?"**
> 回答テンプレート
````

---

## 教材生成のワークフロー

1. ユーザーの入力（トピック or 目次）を受け取る
2. トピックの粒度を判定する
3. 教材を Markdown ファイルとしてプロジェクトの `sections/backend/` ディレクトリ配下に生成する
4. 複数セクションの場合は次に進むか確認する

### ファイル配置ルール

章（チャプター）番号でフォルダを分け、節（セクション）番号でファイル名を付ける。

```
sections/backend/
├── 01-server-side-fundamentals/
│   ├── 1-1-nodejs-event-driven-model.md
│   ├── 1-2-http-protocol-deep-dive.md
│   ├── 1-3-process-and-containers.md
│   └── 1-4-logging-and-observability.md
├── 02-database-design/
│   ├── 2-1-relational-database-fundamentals.md
│   ├── 2-2-sql-query-practice.md
│   ├── 2-3-index-and-query-optimization.md
│   ├── 2-4-nosql-databases.md
│   └── 2-5-data-migration.md
├── 03-api-design/
│   ├── 3-1-restful-api-design.md
│   ├── 3-2-graphql-fundamentals.md
│   ├── 3-3-authentication-authorization.md
│   ├── 3-4-validation-and-error-handling.md
│   └── 3-5-api-documentation-and-testing.md
├── 04-architecture-patterns/
│   ├── 4-1-layered-architecture.md
│   ├── 4-2-message-queues.md
│   ├── 4-3-websocket-realtime.md
│   └── 4-4-cqrs-event-sourcing.md
├── 05-system-design/
│   ├── 5-1-interview-framework.md
│   ├── 5-2-estimation-practice.md
│   ├── 5-3-scalability-components.md
│   ├── 5-4-caching-strategies.md
│   ├── 5-5-consistency-and-tradeoffs.md
│   └── 5-6-design-problems/
│       ├── url-shortener.md
│       ├── chat-application.md
│       ├── news-feed.md
│       ├── video-streaming.md
│       ├── ecommerce.md
│       ├── rate-limiter.md
│       └── notification-service.md
└── 06-capstone-project/
    ├── 6-1-monolith-build.md
    ├── 6-2-performance-tuning.md
    ├── 6-3-scale-out.md
    └── 6-4-microservices-split.md
```

**フォルダ名:** `{2桁章番号}-{章タイトルの英語スラッグ}/`
**ファイル名:** `{章番号}-{節番号}-{トピックの英語スラッグ}.md`

例:
- `1-1. Node.js のイベント駆動モデル` → `sections/backend/01-server-side-fundamentals/1-1-nodejs-event-driven-model.md`
- `2-3. インデックスとクエリ最適化` → `sections/backend/02-database-design/2-3-index-and-query-optimization.md`
- `5-6. URL短縮サービス` → `sections/backend/05-system-design/5-6-design-problems/url-shortener.md`

トピック番号がない場合（例：「JWT認証」のみ）は、カリキュラム内の該当セクションを特定してから適切なフォルダ・ファイル名を決定する。

---

## 環境セットアップの記述ルール

バックエンド教材では環境構築が複雑になりがち。以下のルールで統一する：

### Docker Compose テンプレート

データベースやキャッシュを使う教材には、冒頭に `docker-compose.yml` を提供する：

```yaml
# この教材で使う環境をまとめて起動する
# docker compose up -d で起動、docker compose down で停止
version: '3.8'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: handson
    ports:
      - "5432:5432"
  redis:
    image: redis:7
    ports:
      - "6379:6379"
```

### npm パッケージのインストール

必要なパッケージは Step 0 でまとめてインストールする：

```bash
mkdir backend-handson && cd backend-handson
npm init -y
npm install express pg redis jsonwebtoken bcrypt
npm install -D jest supertest
```

---

## 良い教材の例

以下は「JWT 認証を段階的に構築する」教材のパターン。このスタイルを参考にする：

- **Step ごとに完結した動くコード** がある（途中で止めても動く）
- **学びポイント** で「なぜこう書くのか」「なぜこの設計判断をするのか」を言語化している
- **実務との比較** で「Passport.js / Auth0 が裏側でやっていること」と橋渡ししている
- **セキュリティの落とし穴** をわざと体験させている（トークンを改ざんする、期限切れトークンで叩く、など）
- **curl コマンドでの検証** で、ブラウザに依存しない動作確認ができる
- **アーキテクチャ図** でトークンの流れを可視化している
- 最後に **理解チェック課題** がレベル別に並んでいる

---

## 言語について

- 教材本文・解説・コメントは **日本語**
- コード中の変数名・関数名・テーブル名・カラム名は **英語**
- SQL のキーワードは **大文字**（`SELECT`, `INSERT`, `CREATE TABLE` など）
- 面接対策の模範回答は **日本語 + 英語キーフレーズ併記**
- システムデザインのコンポーネント名は **英語**（Load Balancer, Cache, Message Queue など）
