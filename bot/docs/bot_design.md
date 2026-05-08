# じょぎBANK bot 設計ドキュメント

> 配置場所：`bot/docs/bot_design.md`

---

## 1. 概要

じょぎ内専用通貨の発行・管理を担うDiscord bot。  
通貨管理に加え、借金・貸付・資産運用などの銀行機能をスラッシュコマンドで提供する。  
HTTP APIサーバーを兼ねており、他botからの残高操作リクエストを受け付ける。

---

## 2. ディレクトリ構成

```
bot/
├── docs/
│   └── bot_design.md               ← このファイル
├── prisma/
│   ├── schema.prisma               ← DBスキーマ定義（Prisma）
│   └── migrations/                 ← マイグレーションファイル（リポジトリ管理）
├── src/
│   ├── index.ts                    ← エントリーポイント。bot と API サーバーを起動する
│   ├── config.ts                   ← 定数管理（デイリーボーナス額・信用スコア閾値など）
│   │
│   ├── bot/                        ← Discord bot 関連
│   │   ├── client.ts               ← discord.js クライアントの初期化・イベント登録
│   │   ├── deploy.ts               ← スラッシュコマンドを Discord に登録するスクリプト
│   │   ├── commands/               ← スラッシュコマンドの定義と実行ロジック
│   │   │   ├── index.ts            ← コマンド一覧をまとめて export する
│   │   │   ├── balance.ts          ← /balance：残高確認
│   │   │   ├── transfer.ts         ← /transfer：送金（手数料1%を送金者負担で徴収）
│   │   │   ├── history.ts          ← /history：取引履歴
│   │   │   ├── daily.ts            ← /daily：デイリーボーナス
│   │   │   ├── ranking.ts          ← /ranking：資産ランキング
│   │   │   ├── help.ts             ← /help：コマンド一覧 + Web URL 案内
│   │   │   ├── loan/
│   │   │   │   ├── request.ts      ← /loan request：借金申請
│   │   │   │   ├── list.ts         ← /loan list：借入・貸付一覧
│   │   │   │   ├── repay.ts        ← /loan repay：返済
│   │   │   │   └── status.ts       ← /loan status：残債確認
│   │   │   ├── invest/
│   │   │   │   ├── buy.ts          ← /invest buy：株購入
│   │   │   │   ├── sell.ts         ← /invest sell：売却
│   │   │   │   └── chart.ts        ← /invest chart：価格チャート画像生成
│   │   │   ├── deposit.ts          ← /deposit：定期預金
│   │   │   ├── gamble.ts           ← /gamble：ギャンブル（期待値95%・勝利時1.9倍）
│   │   │   └── admin/
│   │   │       ├── grant.ts        ← /admin grant：通貨付与
│   │   │       ├── take.ts         ← /admin take：通貨没収
│   │   │       ├── forgive.ts      ← /admin forgive：借金チャラ
│   │   │       ├── ban.ts          ← /admin ban：利用停止
│   │   │       └── apikey.ts       ← /admin apikey：APIキー発行・無効化
│   │   └── interactions/
│   │       └── loanApproval.ts     ← 借金申請の承認/拒否ボタンのインタラクション処理
│   │
│   ├── api/                        ← HTTP API サーバー（他bot向け）
│   │   ├── server.ts               ← Express サーバーの初期化・ルーティング登録
│   │   ├── middleware/
│   │   │   └── apiKeyAuth.ts       ← X-API-Key ヘッダーを検証する認証ミドルウェア
│   │   └── routes/
│   │       └── balance.ts          ← GET /api/balance/:userId, POST /api/balance/deduct, grant
│   │
│   ├── services/                   ← ビジネスロジック（コマンドと DB の橋渡し）
│   │   ├── userService.ts          ← ユーザー作成・残高操作・BAN管理・信用スコア更新
│   │   ├── transactionService.ts   ← 取引記録・履歴取得
│   │   ├── loanService.ts          ← 借金申請・返済・利息計算・督促・解放条件チェック
│   │   ├── investService.ts        ← 株の売買・ポジション管理
│   │   ├── stockPriceService.ts    ← 株価変動ロジック（cron から呼ばれる）
│   │   ├── depositService.ts       ← 定期預金の預け入れ・満期処理
│   │   └── gambleService.ts        ← ギャンブルの勝敗判定・胴元取り分処理
│   │
│   ├── db/
│   │   └── client.ts               ← Prisma クライアントのシングルトン
│   │
│   ├── cron/
│   │   ├── updateStockPrices.ts    ← 株価を1時間ごとに更新するバッチ
│   │   └── processLoanInterest.ts  ← 借金の経過日数を毎日更新・督促通知を送るバッチ
│   │
│   ├── types/
│   │   └── result.ts               ← Result 型・AppError 型・ErrorCode の定義
│   │
│   └── utils/
│       ├── formatCurrency.ts       ← 通貨のフォーマット（例：1000 → 1,000 JC）
│       ├── embedBuilder.ts         ← Discord の Embed メッセージを生成するヘルパー
│       └── rateLimit.ts            ← コマンド連打制限（メモリ Map で管理）
│
├── .env.example
├── .env                            ← 環境変数（gitignore 対象）
├── tsconfig.json
└── package.json
```

---

## 3. 技術スタック

| 項目               | 採用技術                            |
| ------------------ | ----------------------------------- |
| 言語               | TypeScript                          |
| Discord ライブラリ | discord.js v14                      |
| HTTP サーバー      | Express（または Fastify）           |
| ORM                | Prisma                              |
| DB                 | SQLite（開発） / PostgreSQL（本番） |

---

## 4. 他botとの連携アーキテクチャ

他bot（チンチロ・ガチャ・スロット等）はHTTPリクエストでじょぎBANKの残高を操作する。

```
チンチロbot / ガチャbot / スロットbot / 将来のbot
    │
    │  POST /api/balance/deduct   （消費）
    │  POST /api/balance/grant    （付与）
    │  GET  /api/balance/:userId  （残高取得）
    ▼
じょぎBANK（Discord bot + REST API サーバー）
    │
    ▼
Database
```

### 認証

- 他botごとに **APIキーを発行** し、リクエストヘッダー `X-API-Key` で認証
- `/admin apikey generate bot名` で管理者（開発者）が発行・無効化
- キー漏洩時は該当キーのみ無効化可能

---

## 5. コマンド一覧

### 通貨基本

| コマンド               | 説明                                             |
| ---------------------- | ------------------------------------------------ |
| `/balance`             | 自分の残高確認                                   |
| `/transfer @user 金額` | ユーザー間送金（手数料1%を送金者負担・最低1 JC） |
| `/history`             | 取引履歴（種別フィルタ可）                       |
| `/daily`               | デイリーボーナス受け取り                         |
| `/ranking`             | サークル内資産ランキング                         |
| `/help`                | コマンド一覧 + WebサービスのURLを案内            |

### 借金・貸付

| コマンド                   | 説明                                |
| -------------------------- | ----------------------------------- |
| `/loan request @user 金額` | 借金申請（相手がボタンで承認/拒否） |
| `/loan list`               | 自分の借入・貸付一覧                |
| `/loan repay @user 金額`   | 返済（分割可）                      |
| `/loan status`             | 利息込みの残債確認                  |

### 資産運用

| コマンド                 | 説明                                 |
| ------------------------ | ------------------------------------ |
| `/invest buy 銘柄 枚数`  | 株購入                               |
| `/invest sell 銘柄 枚数` | 売却                                 |
| `/invest chart 銘柄`     | 価格チャート（画像）                 |
| `/deposit 金額 日数`     | 定期預金                             |
| `/gamble 金額`           | ギャンブル（期待値95%・勝利時1.9倍） |

### 管理者（開発者限定）

| コマンド                       | 説明               |
| ------------------------------ | ------------------ |
| `/admin grant @user 金額`      | 通貨付与           |
| `/admin take @user 金額`       | 通貨没収           |
| `/admin forgive @user`         | 借金チャラ         |
| `/admin ban @user`             | コマンド利用停止   |
| `/admin apikey generate bot名` | 他bot用APIキー発行 |

---

## 6. ビジネスロジック

### 借金ルール

| 項目               | 内容                                                                               |
| ------------------ | ---------------------------------------------------------------------------------- |
| 金利方式           | **単利・日利1%**                                                                   |
| 利息計算式         | `利息 = 元本 × 0.01 × 経過日数`                                                    |
| 借金上限           | `信用スコア × 基準額（TODO: 要確認 / 暫定500）`                                    |
| 借金機能の解放条件 | 登録から7日経過かつ信用スコア1以上、または登録時点で一定額以上所持（TODO: 要確認） |
| マイナス残高       | 許容（踏み倒し後）                                                                 |
| 踏み倒し           | 返済期限超過 → 一定期間コマンド全制限（TODO: 要確認）                              |
| 督促               | 返済期限超過でDiscord通知                                                          |
| 保証人             | なし                                                                               |

`loans.status`：`pending` / `active` / `defaulted` / `repaid`

### 信用スコア

| 項目           | 内容                                                                         |
| -------------- | ---------------------------------------------------------------------------- |
| スコア範囲     | 0〜10（整数）                                                                |
| 初期値         | 0                                                                            |
| 加算対象       | 全収入の累計（デイリーボーナス・送金受取・ギャンブル勝利・定期預金満期など） |
| 更新タイミング | 収入発生時にリアルタイム更新                                                 |

詳細は `docs/validation_and_ratelimit.md` 参照。

### レート制限

| コマンド        | 制限     |
| --------------- | -------- |
| `/transfer`     | 1分に1回 |
| `/loan request` | 1分に1回 |
| `/loan repay`   | 1分に1回 |
| `/gamble`       | 制限なし |

### 資産運用ルール

| 種別       | 内容                                                              |
| ---------- | ----------------------------------------------------------------- |
| 株相場     | じょぎ独自銘柄を複数用意。1時間ごとのcronでランダム＋トレンド変動 |
| 定期預金   | 日利0.3%・期間中引き出し不可・満期で自動返還                      |
| ギャンブル | 期待値95%・勝利時1.9倍。胴元取り分5%は消滅（シンク）              |

### 経済シンク

| 種別                 | 内容                                           |
| -------------------- | ---------------------------------------------- |
| ギャンブル胴元取り分 | 期待値95%（5%消滅）・勝利時1.9倍返し           |
| 送金手数料           | 送金額の1%を送金者負担で徴収・消滅（最低1 JC） |

詳細は `docs/economy.md` 参照。

---

## 7. DBスキーマ

### `users`

```sql
discord_id     TEXT PRIMARY KEY
balance        INTEGER            -- マイナス許容
total_income   INTEGER            -- 累計収入（信用スコア算出に使用）
credit_score   INTEGER            -- 0〜10（total_income をもとにリアルタイム更新）
banned_until   TIMESTAMP          -- NULL以外でコマンド全制限中
registered_at  TIMESTAMP          -- 借金機能の解放判定に使用
created_at     TIMESTAMP
```

### `transactions`

```sql
id         INTEGER PRIMARY KEY
from_user  TEXT REFERENCES users
to_user    TEXT REFERENCES users
amount     INTEGER
type       TEXT  -- transfer / daily / loan_repay / invest / gamble / admin
created_at TIMESTAMP
```

### `loans`

```sql
id            INTEGER PRIMARY KEY
lender_id     TEXT REFERENCES users
borrower_id   TEXT REFERENCES users
principal     INTEGER
interest_rate REAL        -- 0.01 固定
days_elapsed  INTEGER     -- バッチで毎日更新
status        TEXT        -- pending / active / defaulted / repaid
due_date      TIMESTAMP
created_at    TIMESTAMP
```

### `stocks`

```sql
id            INTEGER PRIMARY KEY
symbol        TEXT
name          TEXT
current_price INTEGER
updated_at    TIMESTAMP
```

### `stock_holdings`

```sql
id            INTEGER PRIMARY KEY
user_id       TEXT REFERENCES users
stock_id      INTEGER REFERENCES stocks
quantity      INTEGER
avg_buy_price INTEGER
```

### `stock_history`

```sql
id          INTEGER PRIMARY KEY
stock_id    INTEGER REFERENCES stocks
price       INTEGER
recorded_at TIMESTAMP
```

### `deposits`

```sql
id         INTEGER PRIMARY KEY
user_id    TEXT REFERENCES users
amount     INTEGER
rate       REAL       -- 0.003 固定
matured_at TIMESTAMP
status     TEXT       -- active / matured
```

### `api_keys`

```sql
id         INTEGER PRIMARY KEY
key        TEXT UNIQUE
bot_name   TEXT
created_at TIMESTAMP
```

---

## 8. 実装ロードマップ

1. Prismaスキーマ・マイグレーション初期設定
2. 通貨基本コマンド（`/balance` `/transfer` `/daily` `/history` `/ranking`）
3. HTTP APIサーバー（Express + APIキー認証ミドルウェア）
4. 他botとの連携テスト（チンチロ等と最小動作確認）← **ここまで優先**
5. 借金・貸付機能（申請フロー・単利計算・信用スコア・督促バッチ）
6. 資産運用機能（株相場・定期預金・ギャンブル）

---

## 9. 未決定事項

- 通貨の名称・単位
- 初回ウェルカムボーナスの有無・金額
- デイリーボーナスの金額
- 踏み倒し時のコマンド制限期間
- 借金機能即時解放の所持金閾値
- 株銘柄の名称・種類・価格変動ロジックの詳細
