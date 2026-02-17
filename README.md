# Next Gen Forward (Modified)
<img width="535" height="185" alt="项目主图" src="https://github.com/user-attachments/assets/a632a655-0b6e-4f31-b6ea-1364028bf540" />

一个基于 Cloudflare Workers 的 Telegram 双向私聊机器人方案，支持快速部署与稳定运行。

本仓库基于 [mole404/NextGenForward](https://github.com/mole404/NextGenForward) 进行了小小修改。

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)

---

## 项目亮点

- 支持本地题库与 Cloudflare Turnstile（可选）两种验证方式
- 支持本地规则与 Grok（可选）组合识别
- 基于 Cloudflare Workers，部署成本低，维护简单
- 支持文本、图片、视频、文件等常见消息类型转发

---

## 目录

1. [部署前准备](#部署前准备)
2. [详细部署步骤](#详细部署步骤)
3. [详细配置步骤](#详细配置步骤)
4. [可选高级配置](#可选高级配置)
5. [验证部署是否成功](#验证部署是否成功)
6. [常见问题排查](#常见问题排查)

---

## 部署前准备

1. 准备 Cloudflare 账号。  
2. 使用 [@BotFather](https://t.me/BotFather) 创建机器人并拿到 `BOT_TOKEN`。  
3. 在 Telegram 创建一个超级群组（Supergroup），并开启 Topics。  
4. 将机器人加入该群并授予需要的权限。  
5. 获取 `SUPERGROUP_ID`，必须是 `-100` 开头。  

`SUPERGROUP_ID` 获取方法（桌面端最稳妥）：

1. 在群里任意消息上右键，复制消息链接。  
2. 链接里会有一段纯数字（例如 `1234567890`）。  
3. 在前面加 `-100`，得到 `SUPERGROUP_ID`（例如 `-1001234567890`）。  

---

## 详细部署步骤

### 方式 A：Cloudflare 连接 GitHub（推荐）

1. Fork 本仓库到你的 GitHub 账号。  
2. 打开 [Cloudflare Dashboard](https://dash.cloudflare.com/)。  
3. 进入 `Workers 和 Pages`。  
4. 点击创建应用，选择 `Connect to Git`。  
5. 授权并选择你 Fork 后的仓库。  
6. 构建设置保持默认，入口文件应为 `worker.js`。  
7. 点击部署，等待完成。  

### 方式 B：Cloudflare Dashboard 手动粘贴代码

1. 打开 Cloudflare Dashboard -> `Workers 和 Pages`。  
2. 创建 Worker。  
3. 打开编辑器，删除默认代码。  
4. 将仓库里的 `worker.js` 全量粘贴并保存部署。  

### 方式 C：Wrangler CLI 部署（适合本地开发）

1. 安装 Wrangler 并登录。  
2. 在仓库目录执行发布。  

```bash
npm i -g wrangler
wrangler login
wrangler deploy
```

3. 如果使用 CLI 管理 KV，可先创建命名空间，再在 `wrangler.toml` 中补充绑定。  

```bash
wrangler kv namespace create TOPIC_MAP
```

`wrangler.toml` 示例绑定：

```toml
kv_namespaces = [
  { binding = "TOPIC_MAP", id = "你的KV命名空间ID" }
]
```

---

## 详细配置步骤

### 步骤 1：绑定 KV（必做）

在 Cloudflare 中创建 KV 命名空间，并绑定到 Worker：

- 绑定名必须是 `TOPIC_MAP`
- 未绑定时 Worker 会直接返回错误：`KV 'TOPIC_MAP' not bound.`

### 步骤 2：配置环境变量

必填变量：

| 变量名 | 是否必填 | 示例 | 说明 |
| :--- | :---: | :--- | :--- |
| `BOT_TOKEN` | 是 | `123456:ABC...` | Telegram 机器人 Token，建议用密钥类型 |
| `SUPERGROUP_ID` | 是 | `-1001234567890` | 管理群 ID，必须 `-100` 开头 |

推荐变量：

| 变量名 | 是否必填 | 示例 | 说明 |
| :--- | :---: | :--- | :--- |
| `WORKER_URL` | 推荐 | `https://xxx.workers.dev` | 用于生成验证页面地址；建议始终设置 |
| `WEBHOOK_SECRET` | 推荐 | `Abc_1234-XYZ` | 启用 Telegram Secret Token 校验，提升安全性 |

常用可选变量：

| 变量名 | 示例 | 说明 |
| :--- | :--- | :--- |
| `API_BASE` | `https://api.telegram.org` | Telegram API 地址，默认可不填 |
| `ADMIN_IDS` | `123456789,987654321` | 指定可执行管理操作的管理员白名单（逗号分隔） |
| `VERIFIED_TTL_SECONDS` | `86400` | 验证通过状态 TTL；不填或 `<=0` 表示不过期 |

Turnstile 相关可选变量：

| 变量名 | 示例 | 说明 |
| :--- | :--- | :--- |
| `CF_TURNSTILE_SITE_KEY` | `0x4AAAA...` | Turnstile Site Key |
| `CF_TURNSTILE_SECRET_KEY` | `0x4AAAA...` | Turnstile Secret Key，建议密钥类型 |
| `TURNSTILE_ALLOWED_HOSTNAMES` | `a.workers.dev,b.example.com` | 可选，校验 Turnstile 返回 hostname |
| `TURNSTILE_ACTION` | `tg_verify` | 可选，校验 Turnstile action；默认 `tg_verify` |
| `VERIFY_SIGNING_SECRET` | `your_sign_secret` | 可选，验证完成回调签名密钥；未设置会回退到其他密钥 |

Grok 相关可选变量：

| 变量名 | 示例 | 说明 |
| :--- | :--- | :--- |
| `GROK_API_KEY` | `sk-...` | Grok 接口密钥 |
| `GROK_API_URL` | `https://.../v1/chat/completions` | Grok 接口地址 |
| `GROK_MODEL` | `grok-4.1-expert` | 使用的模型名 |
| `GROK_TIMEOUT_MS` | `12000` | 请求超时，默认 12000ms，最小按代码约束回退到 12000ms |
| `QUIZ_GROK_ENABLED` | `true` | 是否启用在线出题，默认 `true` |

### 步骤 3：激活 Webhook（必做）

本项目 webhook 路径默认为 `/`，激活 URL 如下。

未设置 `WEBHOOK_SECRET`：

```text
https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<你的Worker域名>
```

已设置 `WEBHOOK_SECRET`：

```text
https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<你的Worker域名>/&secret_token=<WEBHOOK_SECRET>
```

成功返回通常包含：`"Webhook was set"`。

查看当前 webhook：

```text
https://api.telegram.org/bot<BOT_TOKEN>/getWebhookInfo
```

删除 webhook（如需重置）：

```text
https://api.telegram.org/bot<BOT_TOKEN>/deleteWebhook?drop_pending_updates=true
```

---

## 可选高级配置

### 开启 Turnstile 网页验证

1. 在 Cloudflare Turnstile 创建站点。  
2. Hostname 中添加你的 Worker 域名。  
3. 将 `CF_TURNSTILE_SITE_KEY`、`CF_TURNSTILE_SECRET_KEY` 写入 Worker 环境变量。  
4. 建议同步设置 `WORKER_URL`。  
5. 如需更严格校验，可增加：  
`TURNSTILE_ALLOWED_HOSTNAMES` 和 `TURNSTILE_ACTION`。  

### 开启 Grok 垃圾识别

1. 配置 `GROK_API_KEY`、`GROK_API_URL`、`GROK_MODEL`。  
2. 可按需设置 `GROK_TIMEOUT_MS`。  
3. 不配置 Grok 也可运行，系统会以本地规则为主。  

### 开启在线题库出题

1. 配置好 Grok 相关变量。  
2. 设置 `QUIZ_GROK_ENABLED=true`。  
3. 当在线出题失败时，系统会自动回退到本地题库。  

### 限定管理员身份

1. 设置 `ADMIN_IDS`（逗号分隔）。  
2. 代码逻辑是“白名单 + 群管理员身份”双重校验。  

### 控制验证状态过期时间

1. 设置 `VERIFIED_TTL_SECONDS` 为正整数秒。  
2. 小于 60 秒会被提升到 60 秒。  
3. 不设置或 `<=0` 时，验证状态默认不过期。  

---

## 验证部署是否成功

按顺序自检：

1. 访问 Worker 域名返回 `OK`。  
2. `getWebhookInfo` 显示 webhook 已设置。  
3. 私聊机器人可收到验证提示。  
4. 管理群中可以看到转发话题。  

---

## 常见问题排查

### 1) 机器人无响应

检查以下项目：

- `BOT_TOKEN` 是否正确  
- `SUPERGROUP_ID` 是否 `-100` 开头  
- `TOPIC_MAP` 是否已正确绑定  
- webhook 是否已成功设置  

### 2) Turnstile 页面无法验证

重点检查：

- `WORKER_URL` 是否正确  
- `CF_TURNSTILE_SITE_KEY` / `CF_TURNSTILE_SECRET_KEY` 是否完整  
- Hostname 是否包含当前 Worker 域名  
- 若设置了 `TURNSTILE_ACTION`，是否与前端 action 一致  

### 3) 开了 Webhook Secret 后收不到消息

重点检查：

- `setWebhook` URL 是否携带 `secret_token`  
- Header `X-Telegram-Bot-Api-Secret-Token` 是否与 `WEBHOOK_SECRET` 一致  

---

## 致谢

感谢 Codex 的大力支持。

如果这个项目对你有帮助，欢迎 Star。
