# LINE Harness 手動セットアップ RUNBOOK

`npx create-line-harness` を使わず、同じ構築フローを手作業（スクリプト）で実行するための手順。
CLI が内部でやっている wrangler 操作を `scripts-local/deploy.sh` に分解してある。

---

## 役割分担

| 担当 | 作業 |
|------|------|
| 人間 | ① `wrangler login` ② `setup.local.env` に LINE 5項目を記入（シークレット登録） |
| Claude | `bash scripts-local/deploy.sh` を一括実行（D1〜deploy〜`.mcp.json` 更新） |
| 人間 | 仕上げ：LINE コンソールで Webhook URL / LIFF エンドポイント設定 |

シークレットは `setup.local.env`（`.gitignore` 済み）にのみ書く。チャット上では扱わない。

---

## 事前準備（人間）

```bash
cd line-harness-oss
npx wrangler login          # ブラウザ認証（デプロイ先アカウント）
pnpm install                # 済みならスキップ
cp setup.local.env.example setup.local.env
$EDITOR setup.local.env     # 下記5項目 + PROJECT を記入
```

### setup.local.env に入れる値（LINE Developers）

| 変数 | 取得元 |
|------|--------|
| `PROJECT` | 任意。Worker / D1 / Pages の名前（英小文字・数字・ハイフン） |
| `LINE_CHANNEL_ID` | Messaging API チャネルの Channel ID（数字） |
| `LINE_CHANNEL_ACCESS_TOKEN` | Messaging API 設定 → チャネルアクセストークン（長期） |
| `LINE_CHANNEL_SECRET` | Channel ID と同じページの Channel Secret |
| `LINE_LOGIN_CHANNEL_ID` | 別途作成する LINE Login チャネルの ID |
| `LIFF_ID` | LINE Login チャネル → LIFF タブで作成（`123-xxxx` 形式） |

---

## 実行（Claude）

```bash
bash scripts-local/deploy.sh
```

冪等。失敗しても再実行で続きから（既存リソースはスキップ、`API_KEY` は固定保存）。

### スクリプトが行う 11 ステップ

| # | 内容 | 主なコマンド |
|---|------|------|
| 0 | ログイン確認 / API_KEY 生成 | `wrangler whoami`, `openssl rand` |
| 1 | D1 作成 | `wrangler d1 create $PROJECT` |
| 2 | スキーマ適用 | `d1 execute --file bootstrap.sql` + 差分 migration |
| 3 | R2 バケット作成 | `wrangler r2 bucket create $PROJECT-images` |
| 4 | Bot Basic ID 取得 | LINE API `/v2/bot/info` |
| 5 | Worker ビルド & deploy | deps build → `vite build` → `wrangler deploy` |
| 6 | Worker シークレット | `wrangler secret bulk`（LINE トークン等） |
| 7 | line_accounts 登録 | `d1 execute`（ON CONFLICT で upsert） |
| 8 | 管理画面 deploy | `next build` → `pages deploy out` |
| 9 | 認証設定 | `secret bulk`（ADMIN_ORIGIN / CORS / Cookie） |
| 10 | 最終 wrangler.toml & 再deploy | vars 入り toml 生成 → `wrangler deploy` |
| 11 | `.mcp.json` 更新 | 親プロジェクトの `.mcp.json` を新 URL/KEY に |

---

## 仕上げ（人間 — LINE コンソール）

スクリプト完了時に URL が表示される。

1. **Messaging API → Webhook URL** を設定: `<WORKER_URL>/webhook`（Webhook 利用 ON）
2. **LIFF → エンドポイント URL** を `<WORKER_URL>` に変更し、**公開**状態にする
3. 管理画面 `<ADMIN_URL>` に **API_KEY** でログイン
4. Claude Code を再起動 → 新しい line-harness MCP が有効化

---

## 生成物 / 状態ファイル

| ファイル | 用途 | git |
|----------|------|-----|
| `setup.local.env` | 入力したシークレット | 無視 |
| `scripts-local/.deploy-state.env` | API_KEY / 各 URL の保存（再実行用） | 無視 |
| `apps/worker/wrangler.toml` | 実 account/D1 ID 入りに更新される | 追跡（本番設定として commit 可） |

---

## トラブル時

- 途中失敗 → そのまま `bash scripts-local/deploy.sh` を再実行
- 全部やり直したい → Cloudflare 側の Worker / D1 / R2 / Pages を削除し、
  `scripts-local/.deploy-state.env` を消してから再実行
- CLI 版に戻したい → この fork ディレクトリ内で `npx create-line-harness`
  （cwd に `pnpm-workspace.yaml` があれば fork をそのまま使う）
