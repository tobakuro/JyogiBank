# じょぎBANK プロジェクト概要

> Discordサークル「じょぎ」内専用通貨・銀行システム

---

## 1. プロジェクト概要

じょぎ内で稼働している娯楽系bot（チンチロ・ガチャ・スロット）がそれぞれ独自ポイントを持っている状態を解消し、**共通通貨で統一する**ことを目的としたプロジェクト。

- **じょぎBANK bot**：Discord上で通貨の発行・管理・借金・資産運用をコマンドで操作できる
- **じょぎBANK web**：botと同等の操作をブラウザのGUIで行えるサービス（コマンド操作に慣れない部員向け）

---

## 2. ディレクトリ構成

```
jyogibank/                        ← プロジェクトルート
├── devbox.json                   ← bot・web 共通の devbox 設定
├── devbox.lock                   ← devbox が自動生成するロックファイル
├── .gitignore
├── docs/
│   ├── overview.md               ← このファイル（共通仕様・システム構成）
│   ├── setup.md                  ← 環境構築手順
│   ├── error_handling.md         ← エラーハンドリング設計
│   ├── validation_and_ratelimit.md ← バリデーション・レート制限・信用スコア設計
│   ├── economy.md                ← 経済設計（インフレ対策・シンク）
│   └── migration.md              ← DB マイグレーション運用方針
├── bot/                          ← Discord bot ＋ HTTP API サーバー
│   ├── docs/
│   │   └── bot_design.md         ← bot 設計ドキュメント
│   └── src/                      ← bot ソースコード（詳細は bot_design.md 参照）
└── web/                          ← Next.js web アプリ
    ├── docs/
    │   └── web_design.md         ← web 設計ドキュメント
    └── src/                      ← web ソースコード（詳細は web_design.md 参照）
```

---

## 3. システム全体構成

```
Discord サーバー（じょぎ）
│
├── チンチロbot ─┐
├── ガチャbot   ─┼─ POST /api/balance/* ─→ じょぎBANK bot
├── スロットbot ─┘        (X-API-Key 認証)         │
│                                                   ▼
└── ユーザー ──── Discord コマンド ──────────────  Database
                                                    ▲
ブラウザ ──── Next.js web ──── REST API ───────────┘
              (Discord OAuth)
```

---

## 4. 共通技術スタック

| 項目 | 内容 |
|---|---|
| 言語 | TypeScript（bot・web 共通） |
| DB | SQLite（開発） / PostgreSQL（本番） |
| 認証（web） | Discord OAuth2 |
| パッケージマネージャー | npm または pnpm |

---

## 5. 共通開発ルール

### ブランチ戦略
- `main`：本番リリース済みコード
- `develop`：開発統合ブランチ
- `feature/xxx`：機能単位で切る（例：`feature/bot-loan`, `feature/web-chart`）
- bot と web で影響範囲が分かれるため、`feature/bot-xxx` / `feature/web-xxx` のプレフィックスを推奨

### コミットメッセージ
```
feat(bot): 借金申請コマンドを追加
fix(web): 残高表示が更新されない問題を修正
docs: overview.md を更新
```

### 環境変数
各サブプロジェクトのルートに `.env` を置く（`.env.example` をリポジトリに含める）。機密情報は絶対にコミットしない。

---

## 6. 参照ドキュメント

- 環境構築手順：`docs/setup.md`
- エラーハンドリング設計：`docs/error_handling.md`
- バリデーション・レート制限・信用スコア設計：`docs/validation_and_ratelimit.md`
- 経済設計（インフレ対策・シンク）：`docs/economy.md`
- DB マイグレーション運用方針：`docs/migration.md`
- bot 設計：`bot/docs/bot_design.md`
- web 設計：`web/docs/web_design.md`

---

## 7. 未決定事項（共通）

- 通貨の名称・単位（例：JC / じょぎコイン）
- 本番環境のホスティング先（bot・web・DB それぞれ）
- デイリーボーナスの金額
- 借金上限のデフォルト値
