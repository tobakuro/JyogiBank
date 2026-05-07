# じょぎBANK 環境構築手順

> 配置場所：`docs/setup.md`

---

## 1. 前提条件

以下がインストール済みであること。

- [Devbox](https://www.jetpack.io/devbox/)（インストール方法は後述）
- Git

OS は macOS / Linux / WSL2（Windows）を想定。

> **WSL2 を使用している場合**、systemd が有効になっている必要があります。  
> 有効化されていない場合は `/etc/wsl.conf` に以下を追加して WSL を再起動してください。
> ```ini
> [boot]
> systemd=true
> ```

---

## 2. Devbox のインストール

```bash
curl -fsSL https://get.jetpack.io/devbox | bash
```

インストール確認：

```bash
devbox version
```

---

## 3. リポジトリのクローン

```bash
git clone <リポジトリURL>
cd jyogibank
```

---

## 4. プロジェクト構成

```
jyogibank/
├── devbox.json          ← bot・web 共通の devbox 設定（ここ1つのみ）
├── devbox.lock
├── docs/
│   ├── overview.md
│   └── setup.md         ← このファイル
├── bot/
│   ├── .env.example
│   └── src/
└── web/
    ├── .env.example
    └── src/
```

`devbox.json` はプロジェクトルートに1つだけ置き、bot・web で使うツールをまとめて管理します。  
各サービスの開発コマンド（`npm install` / `npm run dev` 等）は `cd` で移動してから実行します。

---

## 5. devbox.json

```json
{
  "$schema": "https://raw.githubusercontent.com/jetpack-io/devbox/0.10.0/.schema/devbox.schema.json",
  "packages": [
    "nodejs@22",
    "nodePackages.npm@latest"
  ],
  "shell": {
    "scripts": {
      "setup:bot": "cd bot && npm install",
      "setup:web": "cd web && npm install",
      "setup": "devbox run setup:bot && devbox run setup:web"
    }
  }
}
```

`devbox run setup` を1回実行するだけで bot・web 両方の依存パッケージがインストールされます。

---

## 6. 初回セットアップ

```bash
# プロジェクトルートで devbox シェルに入る
devbox shell

# bot・web の依存パッケージを一括インストール
devbox run setup

# 環境変数ファイルをそれぞれ作成
cp bot/.env.example bot/.env
cp web/.env.example web/.env
# 各 .env を編集して値を設定（下記「環境変数」参照）
```

---

## 7. 開発サーバーの起動

bot と web はそれぞれ別ターミナルで起動します。  
**devbox shell はルートで1回入れば、cd 後もそのまま使えます。**

```bash
# ターミナル1：bot（API サーバー + Discord bot）
devbox shell          # ルートでシェルに入る
cd bot
npm run dev           # → http://localhost:3001 で API 待ち受け

# ターミナル2：web（Next.js）
devbox shell          # ルートでシェルに入る
cd web
npm run dev           # → http://localhost:3000
```

---

## 8. 環境変数

### bot/.env

| 変数名 | 説明 | 例 |
|---|---|---|
| `DISCORD_TOKEN` | Discord Bot トークン | - |
| `CLIENT_ID` | Discord アプリケーション ID | - |
| `GUILD_ID` | じょぎ Discord サーバー ID | - |
| `DATABASE_URL` | DB 接続先 | `file:./dev.db` |
| `API_PORT` | HTTP API サーバーのポート番号 | `3001` |

### web/.env

| 変数名 | 説明 | 例 |
|---|---|---|
| `NEXTAUTH_SECRET` | NextAuth.js のセッション署名キー（任意の文字列） | - |
| `NEXTAUTH_URL` | サービスの URL | `http://localhost:3000` |
| `DISCORD_CLIENT_ID` | Discord OAuth アプリのクライアント ID | - |
| `DISCORD_CLIENT_SECRET` | Discord OAuth アプリのシークレット | - |
| `JYOGIBANK_API_URL` | じょぎBANK API のベース URL | `http://localhost:3001` |
| `JYOGIBANK_API_KEY` | Web 用に発行した API キー | - |

---

## 9. Discord Bot・OAuth の準備

### Bot トークンの取得

1. [Discord Developer Portal](https://discord.com/developers/applications) を開く
2. 「New Application」でアプリを作成
3. 「Bot」タブ → 「Reset Token」でトークンを取得 → `bot/.env` の `DISCORD_TOKEN` に設定
4. 「Privileged Gateway Intents」で必要な権限を有効化
5. 「OAuth2 → URL Generator」で Bot をサーバーに招待

### Discord OAuth2 の設定（web 用）

1. 同じアプリの「OAuth2」タブを開く
2. 「Client ID」と「Client Secret」を取得 → `web/.env` に設定
3. 「Redirects」に `http://localhost:3000/api/auth/callback/discord` を追加

---

## 10. トラブルシューティング

### `devbox shell` に入れない（WSL2）

systemd が有効になっていない可能性があります。「1. 前提条件」の WSL2 向け手順を確認してください。

### Node.js のバージョンが合わない

`devbox shell` の中で確認します。

```bash
node -v   # 22.x.x が表示されれば OK
```

異なるバージョンが表示される場合、`devbox shell` の外のシステム Node.js が使われています。必ずルートで `devbox shell` に入ってから作業してください。

### bot の API に web から繋がらない

`web/.env` の `JYOGIBANK_API_URL` が `bot/.env` の `API_PORT` と一致しているか確認してください。
