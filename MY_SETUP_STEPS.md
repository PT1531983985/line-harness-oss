# MY_SETUP_STEPS

このリポジトリの現状に合わせたセットアップ手順である。`README.md` のクイックスタートだけでは不足があり、実コード上は `D1` に加えて `R2(IMAGES)` と `WORKER_URL` も必要である。

## 1. 前提条件

- `Node.js 20+`
- `pnpm 9+`
- `Cloudflare` アカウント
- `LINE Developers` アカウント
- `wrangler` が使用可能であること

依存関係をインストールする。

```bash
git clone https://github.com/Shudesu/line-harness-oss.git
cd line-harness-oss
pnpm install
```

## 2. LINE 側の準備

### 2.1 Messaging API チャネル

`LINE Developers Console` で `Messaging API` チャネルを作成し、以下を控える。

- `LINE_CHANNEL_SECRET`
- `LINE_CHANNEL_ACCESS_TOKEN`

### 2.2 LINE Login チャネル

`LINE Login` チャネルを作成し、以下を控える。

- `LINE_LOGIN_CHANNEL_ID`
- `LINE_LOGIN_CHANNEL_SECRET`

### 2.3 LIFF アプリ

`LIFF` アプリを作成し、`LIFF ID` を取得する。Worker 側では `LIFF_URL` を使うため、値は以下の形式にする。

```text
https://liff.line.me/{LIFF_ID}
```

## 3. Cloudflare リソース作成

ログイン後、`D1` と `R2` を作成する。

```bash
npx wrangler login
npx wrangler d1 create line-harness
npx wrangler r2 bucket create line-harness-images
```

## 4. `wrangler.toml` の更新

`apps/worker/wrangler.toml` のプレースホルダを実値に置き換える。

更新対象:

- `account_id`
- `[[d1_databases]].database_id`
- `[[r2_buckets]].bucket_name`

必要であれば `name` もデプロイ先の Worker 名に合わせて変更する。

## 5. D1 スキーマ適用

本番用 D1 にスキーマを投入する。

```bash
npx wrangler d1 execute line-harness --file=packages/db/schema.sql --remote
```

ローカル開発用の D1 も作成する。

```bash
pnpm db:migrate:local
```

## 6. Worker シークレット設定

少なくとも以下を設定する。`WORKER_URL` は実コードで参照されるため必須と考えるべきである。

```bash
npx wrangler secret put LINE_CHANNEL_SECRET
npx wrangler secret put LINE_CHANNEL_ACCESS_TOKEN
npx wrangler secret put API_KEY
npx wrangler secret put LINE_LOGIN_CHANNEL_ID
npx wrangler secret put LINE_LOGIN_CHANNEL_SECRET
npx wrangler secret put LIFF_URL
npx wrangler secret put WORKER_URL
```

用途に応じて追加:

- `STRIPE_WEBHOOK_SECRET`
- `X_HARNESS_URL`

## 7. Worker デプロイ

```bash
pnpm deploy:worker
```

デプロイ後に `https://your-worker.workers.dev` のような URL が得られる。この URL を `WORKER_URL` として再設定する。

```bash
npx wrangler secret put WORKER_URL
```

## 8. Web 管理画面の設定

`apps/web/.env.local` を作成し、以下を設定する。

```env
NEXT_PUBLIC_API_URL=https://your-worker.workers.dev
```

ローカルで管理画面を起動する。

```bash
pnpm dev:web
```

## 9. ローカル開発起動

Worker と Web をそれぞれ起動する。

```bash
pnpm dev:worker
pnpm dev:web
```

起動先:

- Worker: `http://localhost:8787`
- Web: `http://localhost:3001`

## 10. LINE Webhook 設定

Messaging API の Webhook URL を以下に設定する。

```text
https://your-worker.workers.dev/webhook
```

合わせて以下を無効化する。

- `Auto-reply messages`
- `Greeting messages`

## 11. 動作確認

API キーで疎通確認を行う。

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://your-worker.workers.dev/api/friends/count
```

## 12. 補足

ルートには `pnpm deploy:setup` という対話式セットアップ導線があるが、そのままでは `packages/create-line-harness/dist/index.js` 前提であり、先にビルドが必要である。また、現状コードを見る限り `WORKER_URL` を自動で補完しないため、初回は手動セットアップの方が確実である。

必要なら先に以下を実行してから使う。

```bash
pnpm --filter create-line-harness build
pnpm deploy:setup
```
