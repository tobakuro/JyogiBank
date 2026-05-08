# web よく使うコマンド

> 作業ディレクトリ：`web/`

---

## 開発

| コマンド        | 内容                                                  |
| --------------- | ----------------------------------------------------- |
| `npm run dev`   | 開発サーバー起動（デフォルト：http://localhost:3000） |
| `npm run build` | 本番用ビルドを生成                                    |
| `npm run start` | ビルド済みの本番サーバーを起動                        |

> `npm run dev` の前に `bot/` 側も起動しておかないと API 呼び出しが失敗する。

---

## 型チェック・Lint・フォーマット

| コマンド               | 内容                                |
| ---------------------- | ----------------------------------- |
| `npm run typecheck`    | 型チェックのみ（出力ファイルなし）  |
| `npm run lint`         | ESLint を実行（プロジェクト全体）   |
| `npm run lint:fix`     | ESLint を実行して自動修正           |
| `npm run format:check` | Prettier でフォーマットのズレを検出 |
| `npm run format`       | Prettier でフォーマットを自動修正   |
