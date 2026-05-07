# じょぎBANK web 設計ドキュメント

> 配置場所：`web/docs/web_design.md`

---

## 1. 概要

じょぎBANK botと同等の操作をブラウザのGUIで行えるWebサービス。  
コマンド操作に慣れない部員でも直感的に使えることを目的とする。  
`/help` コマンドでbotがURLを案内する。

---

## 2. ディレクトリ構成

```
web/
├── docs/
│   └── web_design.md                   ← このファイル
├── public/                             ← 静的ファイル（画像・アイコン等）
├── src/
│   ├── app/                            ← Next.js App Router のページ定義
│   │   ├── layout.tsx                  ← 全ページ共通レイアウト（ヘッダー・フッター）
│   │   ├── page.tsx                    ← / トップ・ログインページ
│   │   ├── dashboard/
│   │   │   └── page.tsx                ← /dashboard：残高・資産サマリー・デイリーボーナス
│   │   ├── transfer/
│   │   │   └── page.tsx                ← /transfer：送金フォーム
│   │   ├── history/
│   │   │   └── page.tsx                ← /history：取引履歴テーブル
│   │   ├── ranking/
│   │   │   └── page.tsx                ← /ranking：資産ランキング
│   │   ├── loan/
│   │   │   └── page.tsx                ← /loan：借金申請・返済・残債確認
│   │   ├── invest/
│   │   │   ├── page.tsx                ← /invest：銘柄一覧・ポートフォリオ
│   │   │   └── [symbol]/
│   │   │       └── page.tsx            ← /invest/[symbol]：銘柄詳細・売買フォーム
│   │   └── api/                        ← Next.js Route Handlers（サーバーサイドAPI）
│   │       ├── auth/
│   │       │   └── [...nextauth]/
│   │       │       └── route.ts        ← NextAuth.js の Discord OAuth エンドポイント
│   │       ├── balance/
│   │       │   └── route.ts            ← GET /api/balance：残高取得（jyogibank API を中継）
│   │       ├── transfer/
│   │       │   └── route.ts            ← POST /api/transfer：送金（jyogibank API を中継）
│   │       ├── history/
│   │       │   └── route.ts            ← GET /api/history：取引履歴（jyogibank API を中継）
│   │       ├── loan/
│   │       │   ├── route.ts            ← GET/POST /api/loan：借金一覧・申請
│   │       │   └── repay/
│   │       │       └── route.ts        ← POST /api/loan/repay：返済
│   │       ├── invest/
│   │       │   ├── route.ts            ← GET /api/invest：保有銘柄一覧
│   │       │   └── [symbol]/
│   │       │       └── route.ts        ← GET/POST /api/invest/[symbol]：銘柄取得・売買
│   │       ├── deposit/
│   │       │   └── route.ts            ← GET/POST /api/deposit：定期預金一覧・預け入れ
│   │       ├── gamble/
│   │       │   └── route.ts            ← POST /api/gamble：ギャンブル実行
│   │       ├── daily/
│   │       │   └── route.ts            ← POST /api/daily：デイリーボーナス受け取り
│   │       └── ranking/
│   │           └── route.ts            ← GET /api/ranking：ランキング取得
│   │
│   ├── components/                     ← 再利用可能な UI コンポーネント
│   │   ├── layout/
│   │   │   ├── Header.tsx              ← ヘッダー（ログイン情報・残高表示）
│   │   │   └── Sidebar.tsx             ← サイドバーナビゲーション
│   │   ├── ui/                         ← 汎用 UI 部品
│   │   │   ├── Button.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Modal.tsx               ← 送金確認などの確認モーダル
│   │   │   └── Badge.tsx               ← ステータス表示バッジ
│   │   ├── balance/
│   │   │   └── BalanceCard.tsx         ← 残高表示カード（リアルタイム更新対応）
│   │   ├── transfer/
│   │   │   └── TransferForm.tsx        ← ユーザー検索 + 金額入力フォーム
│   │   ├── history/
│   │   │   └── HistoryTable.tsx        ← 取引履歴テーブル（フィルタ・ページング付き）
│   │   ├── loan/
│   │   │   ├── LoanRequestForm.tsx     ← 借金申請フォーム
│   │   │   ├── LoanList.tsx            ← 借入・貸付一覧
│   │   │   ├── RepaySlider.tsx         ← 返済額スライダー
│   │   │   └── LoanSimulator.tsx       ← 返済スケジュールシミュレーター
│   │   ├── invest/
│   │   │   ├── StockChart.tsx          ← 株価折れ線チャート（Recharts）
│   │   │   ├── StockList.tsx           ← 銘柄一覧カード
│   │   │   ├── BuySellForm.tsx         ← 売買フォーム
│   │   │   └── PortfolioPieChart.tsx   ← 保有銘柄円グラフ
│   │   ├── deposit/
│   │   │   └── DepositForm.tsx         ← 金額・期間入力フォーム
│   │   └── gamble/
│   │       └── GambleAnimator.tsx      ← ルーレット・演出アニメーション
│   │
│   ├── lib/                            ← ユーティリティ・外部連携
│   │   ├── auth.ts                     ← NextAuth.js の設定（Discord プロバイダー定義）
│   │   ├── apiClient.ts                ← jyogibank API への fetch をラップしたクライアント
│   │   └── formatCurrency.ts           ← 通貨フォーマット（例：1000 → 1,000 JC）
│   │
│   └── types/                          ← 型定義
│       ├── api.ts                      ← API レスポンスの型
│       └── next-auth.d.ts              ← NextAuth.js のセッション型拡張
│
├── .env.example
├── .env                                ← 環境変数（gitignore 対象）
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## 3. 技術スタック

| 項目 | 採用技術 |
|---|---|
| フレームワーク | Next.js（App Router） |
| 認証 | Discord OAuth2 + NextAuth.js |
| スタイリング | Tailwind CSS（推奨） |
| API通信 | じょぎBANK REST API（HTTPリクエスト） |

### 認証フロー

```
ユーザー
  → 「Discordでログイン」ボタン
  → Discord OAuth2 認可画面
  → じょぎBANK web に Discord ID・ユーザー名・アバターが返る
  → NextAuth.js がセッション管理
  → Discord ID を使ってじょぎBANK APIから残高・情報を取得
```

---

## 3. ページ構成

| パス | ページ名 | 説明 |
|---|---|---|
| `/` | トップ / ログイン | 未ログイン時のランディング・Discordログインボタン |
| `/dashboard` | ダッシュボード | 残高・資産サマリー・デイリーボーナス |
| `/transfer` | 送金 | ユーザー検索 → 金額入力 → 送金 |
| `/history` | 取引履歴 | 種別フィルタ・ページング付きテーブル |
| `/ranking` | ランキング | サークル内資産ランキング |
| `/loan` | 借金・貸付 | 申請・返済・残債確認 |
| `/invest` | 資産運用 | 株・定期預金・ギャンブル |
| `/invest/[symbol]` | 銘柄詳細 | インタラクティブチャート・売買フォーム |

---

## 4. 機能一覧

### botと同等の機能（GUI化）

| bot コマンド | web での対応 |
|---|---|
| `/balance` | ダッシュボードに常時表示 |
| `/transfer @user 金額` | ユーザー検索 → 金額入力 → 確認モーダル → 送金 |
| `/history` | テーブル表示・種別フィルタ・ページング |
| `/daily` | ボタン1つ、受取済みはグレーアウト＋次回取得可能時刻を表示 |
| `/ranking` | カード形式のランキングページ |
| `/loan request @user 金額` | フォームで申請、承認通知はDiscordに送る |
| `/loan repay @user 金額` | スライダーで返済額指定、残債リアルタイム更新 |
| `/invest buy / sell` | チャートを見ながら売買ボタン |
| `/deposit 金額 日数` | 金額・期間を選んで預け入れ、満期一覧表示 |
| `/gamble 金額` | アニメーション演出つきのギャンブル画面 |

### Webだからこそできる機能

#### 📊 インタラクティブチャート
- 株価の折れ線チャート（Recharts / Chart.js）
- 自分の資産推移グラフ
- 保有銘柄のポートフォリオ円グラフ
- botの `/invest chart` は静止画1枚が限界だが、WebはZoom・期間切り替えが可能

#### ⚡ リアルタイム残高更新（WebSocket）
- じょぎBANK APIサーバーにWebSocketエンドポイントを追加
- 誰かが送金した瞬間に残高が自動更新される
- botでは都度コマンドを叩かないと確認できないため差別化になる

#### 📅 借金返済シミュレーター
- 「いつまでにいくら返すと利息がいくらか」をグラフで可視化
- 単利計算なのでロジックはシンプル、UIで直感的に伝える

#### 🎰 リッチなギャンブル演出
- ルーレット・スロットリールのアニメーション
- botのテキストレスポンスと全く異なる体験を提供

#### 🔔 Web Push通知
- 返済期限が近い、デイリーボーナスが取れる、ローン申請が届いた等をブラウザ通知
- Discordを見ていない部員にもリーチできる

#### 🏦 経済統計ダッシュボード（管理者向け）
- サーバー全体の通貨総流通量
- 1日の取引量推移
- botでは実装しにくいタイプの機能

---

## 5. API連携仕様

WebからじょぎBANK APIを叩く際は、WebサーバーサイドからAPIキーをヘッダーに付与して中継する（クライアントにAPIキーを露出させない）。

```
ブラウザ（Next.js クライアント）
  → POST /api/transfer  （Next.js Route Handler）
  → POST http://jyogibank-api/api/balance/transfer  （X-API-Key 付き）
  → DB 更新 → レスポンス返却
```

### WebSocket接続（リアルタイム更新を実装する場合）

```
ブラウザ ──── WS ws://jyogibank-api/ws ──── じょぎBANK APIサーバー
```

---

## 6. 実装ロードマップ

1. Next.js プロジェクト初期設定・Tailwind CSS 導入
2. Discord OAuth2 + NextAuth.js による認証
3. ダッシュボード（残高・デイリーボーナス）
4. 送金・取引履歴ページ
5. ランキングページ
6. 借金・貸付ページ（申請フォーム・返済スライダー・シミュレーター）
7. 資産運用ページ（チャート・売買・定期預金）
8. ギャンブルページ（アニメーション演出）
9. リアルタイム更新（WebSocket）
10. Web Push通知
11. 管理者向け経済統計ダッシュボード

---

## 7. 未決定事項

- UIデザインのテーマカラー・雰囲気
- リアルタイム更新（WebSocket）の優先度
- Web Push通知の実装有無
- 管理者ダッシュボードの実装有無・公開範囲
- ホスティング先（Vercel / Cloudflare Pages 等）
