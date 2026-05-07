# DB マイグレーション運用方針

> 配置場所：`docs/migration.md`

---

## 1. 基本方針

- マイグレーションファイルは **リポジトリで管理** し、CI で自動適用する
- スキーマ変更は必ず `prisma migrate dev` でマイグレーションファイルを生成してからコミットする
- 生成されたマイグレーションファイルを手動編集しない

---

## 2. ディレクトリ構成

```
bot/
└── prisma/
    ├── schema.prisma              ← スキーマ定義
    └── migrations/
        ├── 20250101000000_init/
        │   └── migration.sql      ← 初期スキーマ
        ├── 20250115000000_add_credit_score/
        │   └── migration.sql      ← 信用スコア追加
        └── migration_lock.toml    ← Prisma が自動生成
```

---

## 3. 開発時の手順

### スキーマを変更したとき

```bash
cd bot

# 1. schema.prisma を編集する

# 2. マイグレーションファイルを生成（開発DBに即時適用される）
npx prisma migrate dev --name 変更内容を英語で簡潔に
# 例: npx prisma migrate dev --name add_credit_score_to_users

# 3. 生成されたファイルをコミットに含める
git add prisma/migrations/
git commit -m "feat(db): usersテーブルにcredit_scoreカラムを追加"
```

### 現在のスキーマをDBに反映するだけのとき（初回セットアップ等）

```bash
npx prisma migrate dev
```

---

## 4. CI での自動適用

`develop` / `main` ブランチへの push 時に自動でマイグレーションを適用する。

```yaml
# .github/workflows/migrate.yml
name: DB Migrate

on:
  push:
    branches: [main, develop]
    paths:
      - "bot/prisma/migrations/**"

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - name: Install dependencies
        run: cd bot && npm install
      - name: Run migrations
        run: cd bot && npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

- `prisma migrate deploy`：生成済みのマイグレーションファイルを順番に適用する（開発用の `migrate dev` とは異なり、対話なしで実行できる）
- `DATABASE_URL` は GitHub Actions の Secrets に登録する

---

## 5. 注意事項

| 項目 | 内容 |
|---|---|
| カラム削除・リネーム | データ損失リスクがあるため、適用前に必ずバックアップを取る |
| マイグレーション失敗時 | CI が失敗してデプロイが止まる。`prisma migrate status` で状態を確認する |
| ロールバック | Prisma は自動ロールバックをサポートしない。必要な場合は手動で逆方向の SQL を実行する |
| 本番 DB の直接変更 | 禁止。必ずマイグレーションファイル経由で変更する |
