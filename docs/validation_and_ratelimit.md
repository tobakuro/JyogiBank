# バリデーション・レート制限・信用スコア設計

> 配置場所：`docs/validation_and_ratelimit.md`

---

## 1. バリデーション

### 実施場所

bot コマンドと HTTP API の**両方で独立して**実施する。  
API は他 bot からも直接叩かれるため、bot 側だけでは不十分。

### 共通バリデーションルール

| 項目 | ルール |
|---|---|
| 金額 | 整数・1以上・所持金以下（送金・ギャンブル） |
| Discord ID | 文字列として扱う・数値変換しない |
| 送金先ユーザー | 自分自身への送金は不可 |
| 借金申請額 | 1以上・借金上限額以下（信用スコアから算出） |
| 返済額 | 1以上・残債以下 |

### バリデーションエラーは `INVALID_AMOUNT` または `INSUFFICIENT_BALANCE` で返す

```ts
// バリデーションの実施順（共通）
// 1. 型・形式チェック（整数か・1以上か）
// 2. ユーザー存在チェック
// 3. 残高・上限チェック
// 4. BAN・制限チェック
```

---

## 2. レート制限

### 対象コマンドと制限

| コマンド | 制限 | 理由 |
|---|---|---|
| `/transfer` | 1分に1回 | 連打による意図しない二重送金防止 |
| `/loan request` | 1分に1回 | 複数申請の乱発防止 |
| `/loan repay` | 1分に1回 | 二重返済防止 |
| `/gamble` | 制限なし | ゲーム性を損なわないため |
| その他 | 制限なし | 参照系・デイリーは DB 側で制御 |

### 実装方針

レート制限はメモリ上の Map で管理する（外部ストア不要・サークル規模で十分）。

```ts
// utils/rateLimit.ts
const lastUsed = new Map<string, number>(); // key: `${userId}:${command}`

export const checkRateLimit = (
  userId: string,
  command: string,
  intervalMs: number
): { ok: true } | { ok: false; remainingMs: number } => {
  const key = `${userId}:${command}`;
  const now = Date.now();
  const last = lastUsed.get(key) ?? 0;
  const remaining = intervalMs - (now - last);

  if (remaining > 0) {
    return { ok: false, remainingMs: remaining };
  }

  lastUsed.set(key, now);
  return { ok: true };
};
```

コマンド側での使い方：

```ts
// bot/commands/transfer.ts
const rate = checkRateLimit(userId, "transfer", 60_000);
if (!rate.ok) {
  const seconds = Math.ceil(rate.remainingMs / 1000);
  return interaction.reply({
    embeds: [errorEmbed(`あと ${seconds} 秒後に再試行できます`)],
    ephemeral: true,
  });
}
```

---

## 3. 信用スコア・借金上限

### 信用スコアの定義

| 項目 | 内容 |
|---|---|
| スコア範囲 | 0〜10（整数） |
| 初期値 | 0 |
| 加算対象 | 全収入の累計（デイリーボーナス・送金受取・ギャンブル勝利・定期預金満期など） |
| スコア計算 | 累計収入に応じて段階的に上昇（閾値は `config.ts` で管理） |

### 信用スコアの閾値（仮・要調整）

| 累計収入 | 信用スコア |
|---|---|
| 0 | 0 |
| 500以上 | 1 |
| 1,500以上 | 2 |
| 3,000以上 | 3 |
| 5,000以上 | 4 |
| 8,000以上 | 5 |
| 12,000以上 | 6 |
| 17,000以上 | 7 |
| 23,000以上 | 8 |
| 30,000以上 | 9 |
| 40,000以上 | 10 |

> デイリーボーナスの金額が確定したら閾値を見直す。`config.ts` の `CREDIT_THRESHOLDS` で定義する。

### 借金上限の計算式

```
借金上限 = 信用スコア × 基準額（TODO: 要確認 / 暫定 500）

例）
信用0  →    0円（借金不可）
信用3  → 1,500円
信用5  → 2,500円
信用10 → 5,000円
```

### 借金機能の解放条件

以下のいずれかを満たした場合に借金機能が使用可能になる。

| 条件 | 内容 |
|---|---|
| 通常 | 登録から **7日経過** かつ 信用スコア1以上 |
| 例外 | 登録時点で所持金が一定額以上（`TODO: 要確認`）の場合は即時解放 |

### DB への反映

```ts
// users テーブルに以下を追加
total_income   INTEGER  -- 累計収入（収入が発生するたびに加算）
credit_score   INTEGER  -- 0〜10（total_income をもとに都度計算 or バッチ更新）
registered_at  TIMESTAMP
```

### 信用スコアの計算タイミング

収入発生時に `total_income` を加算し、閾値を超えていれば `credit_score` を更新する。  
バッチではなくリアルタイムで更新することで、スコアアップの達成感を即時フィードバックできる。

```ts
// services/userService.ts
export const addIncome = async (userId: string, amount: number) => {
  const user = await prisma.user.update({
    where: { discordId: userId },
    data: { totalIncome: { increment: amount } },
  });
  const newScore = calcCreditScore(user.totalIncome); // 閾値テーブルと照合
  if (newScore !== user.creditScore) {
    await prisma.user.update({
      where: { discordId: userId },
      data: { creditScore: newScore },
    });
    // TODO: スコアアップをDiscordで通知するか検討
  }
};
```

---

## 4. 定数の管理場所

上記の未確定値はすべて `src/config.ts`（bot）に集約する。

```ts
// bot/src/config.ts
export const config = {
  DAILY_BONUS_AMOUNT: 100,           // TODO: 要確認
  LOAN_BASE_AMOUNT: 500,             // 信用スコア1あたりの借金上限額 TODO: 要確認
  LOAN_UNLOCK_DAYS: 7,               // 借金機能解放までの日数
  LOAN_INSTANT_UNLOCK_BALANCE: 5000, // 即時解放の所持金閾値 TODO: 要確認
  CREDIT_THRESHOLDS: [0, 500, 1500, 3000, 5000, 8000, 12000, 17000, 23000, 30000, 40000],
  RATE_LIMIT_MS: {
    transfer: 60_000,
    loan_request: 60_000,
    loan_repay: 60_000,
  },
} as const;
```
