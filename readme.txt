# 🎮 LemeHost Auto Renewal — GitHub Action 版

> 将 [LemeHost](https://lemehost.com/) 免费服务器自动续期脚本从 HuggingFace 改写为 **GitHub Action** 版本。

## ✨ 功能

| 功能 | 说明 |
|------|------|
| ✅ **自动续期** | 倒计时 < 阈值（默认 15 分钟）自动续期 |
| ✅ **自动开机** | 检测到服务器离线自动通过 WebSocket 拉起 |
| ✅ **验证码识别** | 基于 `ddddocr` 自动识别登录/续期验证码 |
| ✅ **Cookie 缓存** | 登录后缓存 session，后续轮次直接复用，减少风控触发 |
| ✅ **TG 通知** | 续期/开机/失败结果推送到 Telegram |
| ✅ **多账号** | 每行一个，独立运行 |

## 🔄 与原版（HuggingFace）差异

| 对比项 | HuggingFace 版 | GitHub Action 版 |
|------|----------------|------------------|
| 运行方式 | 7×24 长驻进程 | **自触发循环**：每轮结束 sleep 后触发下一轮 |
| 监控面板 | ✅ Gradio UI | ❌ 不需要（在 Actions 日志查看） |
| 保活机制 | ✅ 自带 | ❌ 不需要（自触发永不停） |
| 部署难度 | 需创建 Space + 上传文件 | 直接 fork 仓库 + 配置 Secrets |
| 成本 | HuggingFace 免费额度 | **GitHub Actions public 仓库完全免费，不限时长** |
| 通知方式 | 每次续期单独通知 | 每轮汇总 + 续期通知 |

> 💡 **为什么不用 cron schedule？**
> 1. GitHub Actions 的 `schedule` 高峰期延迟 5–30 分钟甚至更久，**可能错过 15 分钟续期窗口**
> 2. 仓库 60 天无 push 会自动禁用 scheduled workflows
> 3. 自触发模式（`gh workflow run`）更可靠、间隔更精确

## 🚀 部署

### 第一步：Fork 本仓库

点击右上角 **Fork** 按钮，将本仓库复制到你的 GitHub 账号下。

### 第二步：创建 Personal Access Token（PAT）

> ⚠️ 必须用 PAT，默认 `GITHUB_TOKEN` 无法触发自身的 `workflow_dispatch`

1. 打开 [GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens](https://github.com/settings/personal-access-tokens/new)
2. 点击 **Generate new token**，配置：
   - **Token name**：`lemehost-renew`
   - **Expiration**：建议 90 天（到期需更新）
   - **Repository access**：选择你 fork 的仓库
   - **Permissions**：
     - `Actions` → **Read and write**
     - `Contents` → **Read-only**
   - 或 classic token 勾选 `repo` + `workflow`
3. 复制 token 保存

### 第三步：配置 Secrets

进入你 fork 的仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**：

| Secret 名称 | 值 | 必填 | 说明 |
|-------------|-----|------|------|
| `LEME` | `邮箱-----密码` | ✅ **必填** | 账号密码，多账号换行 |
| `GH_PAT` | 上一步的 token | ✅ **必填** | 用于自触发下一轮 |
| `TG_BOT_TOKEN` | Bot Token | 推荐 | Telegram Bot Token |
| `TG_CHAT_ID` | Chat ID | 推荐 | Telegram 聊天 ID |
| `TG_API` | 反代地址 | 可选 | TG API 反代（GA 通常可直连，无需配置） |
| `SLEEP_SECONDS` | `660` | 可选 | 每轮间隔秒数，默认 660（11分钟） |
| `RENEW_THRESHOLD` | `900` | 可选 | 续期阈值秒数，默认 900（15分钟） |

#### LEME 格式

```
admin@example.com-----123456
user2@example.com-----abcdef
```

> ⚠️ 邮箱和密码之间用 **5个短横线** `-----` 分隔，多账号换行。

### 第四步：首次启动

1. 进入 **Actions** 标签页
2. 点击 **I understand my workflows, go ahead and enable them**
3. 点击 **LemeHost Auto Renewal** → **Run workflow** → **Run workflow**
4. 首次运行会自动执行一轮检查，结束 sleep 后自动触发下一轮，**后续无需手动操作**

## 📱 Telegram 通知设置

### 1. 创建 Bot
1. Telegram 搜索 [@BotFather](https://t.me/BotFather)，发送 `/newbot`
2. 获得 **Bot Token**（格式：`7594103635:AAEoQKB_xxxxx`）

### 2. 获取 Chat ID
1. 搜索 [@userinfobot](https://t.me/userinfobot)，发送任意消息
2. 获得 Chat ID（格式：`123456789`）

### 3. 添加到 Secrets
- `TG_BOT_TOKEN` = Bot Token
- `TG_CHAT_ID` = Chat ID

> GA 的 ubuntu runner 通常可直连 `api.telegram.org`。如果遇到网络问题，可用目录下的 `_worker.js` 部署到 Cloudflare Workers 做反代。_worker.js 文件

## 📊 查看运行结果

每次 Action 运行后：
1. 进入 **Actions** 标签页
2. 点击最新一次运行（可以看到自动循环的链）
3. 展开 **Run renewal check** 步骤查看日志
4. 如果配置了 TG，也会收到推送通知

## ⚙️ 运行流程

```
自触发循环（SSR - Self-Scheduled Run）：

每轮循环：
  1. 解析 LEME 环境变量中的账号
  2. 对每个账号：
     a. 登录（含验证码识别，最多 30 次重试）
     b. 获取服务器列表
     c. 对每台服务器：
        - 检查倒计时 & 停机状态
        - 停机 → WebSocket 连接 → 开机
        - 倒计时 < 阈值 → 续期（含验证码）
        - TG 通知结果
  3. 输出统计 → TG 汇总通知
  4. 如果失败 > 0，标记 Action 失败
  5. sleep SLEEP_SECONDS 秒
  6. gh workflow run renew.yml 触发下一轮
```

## ❓ FAQ

**Q: 自触发模式如果当前运行被取消，链会断吗？**
> 不会。`concurrency: cancel-in-progress: true` 会取消旧运行，但新运行（手动触发）结束后依然会触发下一轮，链自动续上。

**Q: 无需手动管理 cron 或保活吗？**
> 完全不需要。自触发模式永不停机，也没有 60 天不活动 disable 的问题。

**Q: 如果 Action 运行失败会怎样？**
> 本轮失败（`renew.py` 退出码非 0），但 `Schedule next run` 步骤 `if: always()` 保证**即使失败也继续触发下一轮**。失败原因会通过 TG 通知。

**Q: PAT 过期了怎么办？**
> GitHub 会发邮件提醒。到期前更新 Secret 里的 `GH_PAT` 即可，无需重启。

**Q: GitHub Actions 免费额度够用吗？**
> **public 仓库完全免费，不限时长**。private 仓库有 2000 分钟/月限制，请确保仓库为 public。

**Q: 验证码识别成功率如何？**
> ddddocr 对 LemeHost 验证码识别率约 10-20%，脚本会自动重试最多 30 次登录 + 30 次续期，通常都能成功。

**Q: 如何调试？**
> 手动触发 Action（**Run workflow**），查看实时日志输出。

## ⚠️ 免责声明

本项目仅供学习研究使用。使用本脚本产生的任何后果由使用者自行承担。
