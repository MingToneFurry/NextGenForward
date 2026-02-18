# 部署说明（Cloudflare Workers）

本文配合 `wrangler.toml.example` 使用，目标是一次配置后走 Git 自动部署。

## 1. 准备

1. 准备 Cloudflare 账号和一个 Telegram Bot（拿到 `BOT_TOKEN`）。
2. 准备一个启用 Topics 的 Telegram 超级群（拿到 `SUPERGROUP_ID`，必须以 `-100` 开头）。
3. 仓库中复制配置文件：

```powershell
Copy-Item wrangler.toml.example wrangler.toml
```

4. 按你的环境修改 `wrangler.toml` 中的非敏感参数（如 `SUPERGROUP_ID`、`WORKER_URL`）。

## 2. 创建并绑定 KV（TOPIC_MAP）

可二选一：

1. Dashboard 创建 KV Namespace，然后把 `id`/`preview_id` 填到 `wrangler.toml`。
2. CLI 创建：

```bash
wrangler kv namespace create TOPIC_MAP
wrangler kv namespace create TOPIC_MAP --preview
```

把返回的 namespace id 写入 `wrangler.toml`：

```toml
[[kv_namespaces]]
binding = "TOPIC_MAP"
id = "生产KV_ID"
preview_id = "预览KV_ID"
```

## 3. 配置 Secrets（只做一次）

在 Cloudflare Dashboard：

`Workers & Pages -> 你的 Worker -> Settings -> Variables and Secrets`

至少配置：

- `BOT_TOKEN`

建议配置：

- `WEBHOOK_SECRET`
- `GROK_API_KEY`
- `CF_TURNSTILE_SECRET_KEY`
- `VERIFY_SIGNING_SECRET`

说明：这些是敏感信息，不应写入 `wrangler.toml` 或 Git 仓库。

## 4. 连接 Git 自动部署（推荐）

1. 打开 Cloudflare Dashboard -> `Workers & Pages`。
2. 选择 `Create` -> `Import a repository`（Connect to Git）。
3. 绑定你的 GitHub 仓库和分支。
4. 确认入口文件为 `worker.js`（本项目默认）。
5. 首次部署完成后，后续 push 会自动触发部署。

## 5. 首次部署后设置 Telegram Webhook

不使用 secret token：

```text
https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<你的Worker域名>
```

使用 secret token：

```text
https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<你的Worker域名>/&secret_token=<WEBHOOK_SECRET>
```

检查 webhook：

```text
https://api.telegram.org/bot<BOT_TOKEN>/getWebhookInfo
```

## 6. 验证部署

1. 访问 Worker URL，返回 `OK`。
2. `getWebhookInfo` 中无明显错误（如 `last_error_message`）。
3. 机器人能在私聊中响应。
4. 群内能创建/使用话题转发。

## 7. 本地 CLI 部署（可选）

如果你要本地发版，可直接使用：

```bash
wrangler deploy
```

如果想从 `.env` 自动同步 secrets，再部署：

```powershell
powershell -ExecutionPolicy Bypass -File scripts/deploy-with-env.ps1 -Deploy
```

## 8. 常见问题

1. 报错 `KV 'TOPIC_MAP' not bound.`  
   原因：`wrangler.toml` 未正确绑定 KV，或 namespace id 填错。

2. 机器人不回消息  
   先查：`BOT_TOKEN`、`SUPERGROUP_ID`、Webhook 是否正确。

3. `SUPERGROUP_ID must start with -100`  
   你的群 ID 不符合超级群格式，需重新确认。
