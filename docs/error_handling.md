# エラーハンドリング設計

> 配置場所：`docs/error_handling.md`

---

## 1. 基本方針

| 層 | 方針 |
|---|---|
| サービス層 | Result 型で呼び出し元に伝達する |
| bot コマンド層 | Result を受け取りエラーは ephemeral で返す |
| HTTP API 層 | `{ error: string }` + ステータスコードで返す |
| システム障害 | ログファイルに出力する（Discord通知しない） |

---

## 2. Result 型の定義

```ts
// src/types/result.ts（bot・web 共通）
export type Result<T> =
  | { ok: true; data: T }
  | { ok: false; error: AppError };

export type AppError = {
  code: ErrorCode;
  message: string; // ユーザー向けの日本語メッセージ
};

export const ErrorCode = {
  // ユーザー起因
  INSUFFICIENT_BALANCE:    "INSUFFICIENT_BALANCE",
  LOAN_LIMIT_EXCEEDED:     "LOAN_LIMIT_EXCEEDED",
  USER_NOT_FOUND:          "USER_NOT_FOUND",
  INVALID_AMOUNT:          "INVALID_AMOUNT",
  ALREADY_CLAIMED_DAILY:   "ALREADY_CLAIMED_DAILY",
  USER_BANNED:             "USER_BANNED",
  REPAY_EXCEEDS_DEBT:      "REPAY_EXCEEDS_DEBT",
  LOAN_ALREADY_PENDING:    "LOAN_ALREADY_PENDING",
  DEPOSIT_NOT_MATURED:     "DEPOSIT_NOT_MATURED",
  STOCK_NOT_HELD:          "STOCK_NOT_HELD",
  // システム起因
  DB_ERROR:                "DB_ERROR",
  EXTERNAL_API_ERROR:      "EXTERNAL_API_ERROR",
} as const;

export type ErrorCode = typeof ErrorCode[keyof typeof ErrorCode];
```

---

## 3. エラーコード一覧

| コード | 原因 | ユーザー向けメッセージ例 |
|---|---|---|
| `INSUFFICIENT_BALANCE` | 残高不足 | 残高が不足しています |
| `LOAN_LIMIT_EXCEEDED` | 借金上限超過 | 借金上限に達しています |
| `USER_NOT_FOUND` | 存在しないユーザー | 指定したユーザーが見つかりません |
| `INVALID_AMOUNT` | 不正な金額 | 金額は1以上の整数で指定してください |
| `ALREADY_CLAIMED_DAILY` | デイリーボーナス取得済み | 本日のボーナスは取得済みです |
| `USER_BANNED` | BAN中ユーザー | 現在利用が制限されています |
| `REPAY_EXCEEDS_DEBT` | 返済額が残債超過 | 返済額が残債を超えています |
| `LOAN_ALREADY_PENDING` | 承認待ちローンが既存 | 既に申請中の借金があります |
| `DEPOSIT_NOT_MATURED` | 定期預金が満期前 | 定期預金はまだ満期を迎えていません |
| `STOCK_NOT_HELD` | 未保有銘柄の売却 | 保有していない銘柄は売却できません |
| `DB_ERROR` | DB障害 | システムエラーが発生しました。しばらく経ってから再試行してください |
| `EXTERNAL_API_ERROR` | API障害 | システムエラーが発生しました。しばらく経ってから再試行してください |

---

## 4. 各層での実装パターン

### サービス層

```ts
// services/userService.ts
export const transfer = async (
  fromId: string,
  toId: string,
  amount: number
): Promise<Result<{ newBalance: number }>> => {
  const user = await prisma.user.findUnique({ where: { discordId: fromId } });
  if (!user) {
    return { ok: false, error: { code: ErrorCode.USER_NOT_FOUND, message: "指定したユーザーが見つかりません" } };
  }
  if (user.balance < amount) {
    return { ok: false, error: { code: ErrorCode.INSUFFICIENT_BALANCE, message: "残高が不足しています" } };
  }
  // ...送金処理
  return { ok: true, data: { newBalance: user.balance - amount } };
};
```

### bot コマンド層

```ts
// bot/commands/transfer.ts
const result = await userService.transfer(fromId, toId, amount);

if (!result.ok) {
  return interaction.reply({
    embeds: [errorEmbed(result.error.message)],
    ephemeral: true, // 本人にだけ見える
  });
}

return interaction.reply({
  embeds: [successEmbed(`${amount} JC を送金しました`)],
});
```

### HTTP API 層

```ts
// api/routes/balance.ts
const result = await userService.deduct(userId, amount);

if (!result.ok) {
  const status =
    result.error.code === ErrorCode.USER_NOT_FOUND          ? 404 :
    result.error.code === ErrorCode.INSUFFICIENT_BALANCE    ? 400 : 500;
  return res.status(status).json({ error: result.error.message });
}

return res.status(200).json({ balance: result.data.newBalance });
```

### システム障害（DB エラー）

```ts
// services/userService.ts
try {
  const user = await prisma.user.findUnique(...);
} catch (e) {
  // ログに出力（message・stack のみ。接続文字列等は含めない）
  logger.error("DB error in userService.transfer", {
    message: e instanceof Error ? e.message : String(e),
    stack:   e instanceof Error ? e.stack   : undefined,
  });
  return { ok: false, error: { code: ErrorCode.DB_ERROR, message: "システムエラーが発生しました。しばらく経ってから再試行してください" } };
}
```

---

## 5. Embed ヘルパー

```ts
// utils/embedBuilder.ts
import { EmbedBuilder } from "discord.js";

export const errorEmbed = (message: string) =>
  new EmbedBuilder()
    .setColor(0xe74c3c)   // 赤
    .setTitle("エラー")
    .setDescription(message);

export const successEmbed = (message: string) =>
  new EmbedBuilder()
    .setColor(0x2ecc71)   // 緑
    .setDescription(message);
```

---

## 6. ログ出力ルール

- 出力してよい情報：エラーコード・`error.message`・`error.stack`・発生した関数名
- 出力してはいけない情報：APIキー・Discordトークン・ユーザー残高・DBの接続文字列
- ログレベル：ユーザー起因エラーは `warn`、システム障害は `error`
