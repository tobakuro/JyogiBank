## アーキテクチャ
- DB操作は必ず `services/` 層を経由する（コマンド・ルートから `prisma` を直接呼ばない）
- bot の HTTP API（`api/`）と Discord コマンド（`bot/commands/`）でロジックを重複させない（共通処理は `services/` に切り出す）
- web の Route Handler はロジックを持たず、jyogibank API への中継のみ行う
- `db/client.ts` の Prisma クライアントはシングルトンとして使い回す

## エラーハンドリング
- サービス層は Result 型（`{ ok: true, data } | { ok: false, error: string }`）を返す
- bot のエラーメッセージはすべて ephemeral（本人にだけ見える）で返す
- HTTP API のエラーレスポンスは `{ error: string }` + 適切なステータスコードに統一する
- システム障害（DB接続失敗など）はログファイルに出力する（Discordチャンネルには通知しない）

## バリデーション
- 金額は必ず整数・1以上を検証する（bot・API 両方で独立して行う）
- Discord ID は文字列として扱い、数値変換しない

## セキュリティ
- APIキー・トークンは `.env` にのみ記載し、コード中にハードコードしない
- ログには機密情報を含めない（APIキー・Discordトークン・残高・DBの接続文字列）
- ログに出力してよいのはエラーの種類・message・stack のみ
- `api_keys` テーブルのキー値をレスポンスやログに含めない

## 未決定事項への対処
- 通貨単位など未確定の定数は `src/config.ts`（bot）/ `src/lib/config.ts`（web）に集約する
- 未決定の数値はコメントで `TODO: 要確認` を付けて仮置きする
