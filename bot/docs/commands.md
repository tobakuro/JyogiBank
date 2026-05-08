# bot よく使うコマンド

> 作業ディレクトリ：`bot/`

---

## 開発

| コマンド | 内容 |
|---|---|
| `npm run dev` | 開発サーバー起動（ts-node でホットリロードなし） |
| `npm run build` | TypeScript をコンパイルして `dist/` に出力 |
| `npm run start` | ビルド済みの `dist/index.js` を実行 |

---

## 型チェック・Lint・フォーマット

| コマンド | 内容 |
|---|---|
| `npm run typecheck` | 型チェックのみ（出力ファイルなし） |
| `npm run lint` | ESLint を実行（`src/` 対象） |
| `npm run lint:fix` | ESLint を実行して自動修正 |
| `npm run format:check` | Prettier でフォーマットのズレを検出 |
| `npm run format` | Prettier でフォーマットを自動修正 |

---

## DB（Prisma）

| コマンド | 内容 |
|---|---|
| `npx prisma migrate dev --name <name>` | マイグレーションを作成・適用（開発用） |
| `npx prisma migrate deploy` | マイグレーションを適用（本番用） |
| `npx prisma db push` | スキーマを直接 DB に反映（マイグレーション不要な試作時） |
| `npx prisma generate` | Prisma クライアントを再生成（`src/generated/prisma/`） |
| `npx prisma studio` | ブラウザで DB の中身を確認できる GUI を起動 |

---

## Discord コマンド登録

| コマンド | 内容 |
|---|---|
| `npx ts-node src/bot/deploy.ts` | スラッシュコマンドを Discord サーバーに登録 |

> `.env` の `GUILD_ID` を設定しておくこと。本番ではグローバルコマンドとして登録する。
