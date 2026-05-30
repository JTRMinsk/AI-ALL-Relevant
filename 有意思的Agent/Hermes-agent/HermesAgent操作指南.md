# Hermes Agent 操作指南

> 版本：v0.14.0 (2026-05-16) · 安装日期：2026-05-17  
> 安装路径：`~/.hermes/hermes-agent`  
> 当前模型：DeepSeek v4-pro

---

## 一、快速开始

### 交互式对话

```bash
hermes
```

进入 TUI（终端界面），支持多行输入、命令补全、会话历史。

### 单次问答（oneshot，适合脚本）

```bash
hermes -z "你的问题"                           # 默认模型
hermes -z "你的问题" -m deepseek/deepseek-v4-pro  # 指定模型
```

`-z` 模式下只输出答案文本，无 banner、无 spinner、自动批准工具调用。

---

## 二、模型管理

### 选择/切换模型

```bash
hermes model          # 交互式选择 provider 和模型
hermes setup          # 完整配置向导（含模型、工具、平台）
```

### 手动指定

```bash
hermes -m deepseek/deepseek-v4-pro --provider deepseek
```

### 配置文件

模型配置在 `~/.hermes/config.yaml`：

```yaml
model:
  default: "deepseek/deepseek-v4-pro"
  provider: "deepseek"
```

API Key 在 `~/.hermes/.env`：

```
DEEPSEEK_API_KEY=sk-xxx
```

支持的 provider（`hermes model` 可选）：DeepSeek、OpenAI Codex、Anthropic Claude、OpenRouter、Nous Portal、Kimi、MiniMax、GLM/Z.AI、xAI、Gemini、NVIDIA NIM、HuggingFace、Xiaomi MiMo 等。

---

## 三、消息网关（多平台接入）

Hermes 可以接入多个聊天平台，让你的 agent 在微信、Telegram、Discord 等平台上持续在线。

```bash
hermes gateway setup     # 交互式配置平台
hermes gateway start      # 启动（systemd 服务模式）
hermes gateway status     # 查看状态
hermes gateway install    # 安装为 systemd 服务（开机自启）
hermes gateway run        # 前台运行（适合 Docker/WSL）
```

支持的平台：Telegram、Discord、Slack、WhatsApp、Signal、WeChat（微信）、飞书、企业微信、钉钉、QQ。

---

## 四、Skills（技能系统）

Hermes 内置 87 个技能，安装后自动同步到 `~/.hermes/skills/`。

```bash
hermes skills list        # 列出已安装
hermes skills search <关键词>  # 搜索技能市场
hermes skills install <技能名> # 安装新技能
hermes skills browse       # 浏览全部可用
```

技能涵盖：Notion、GitHub、博客监控 (blogwatcher)、邮件 (himalaya)、Obsidian、学术论文、PPT、Maps、Spotify、YouTube、设计、MLOps 等。

---

## 五、插件系统

```bash
hermes plugins list          # 已安装插件
hermes plugins install <url> # 从 Git 安装
hermes plugins enable <插件>  # 启用
hermes plugins disable <插件> # 禁用
```

内置插件 22 个（已启用 18 个），含：SearXNG、DDGS、Brave、Firecrawl、Tavily 等搜索后端，以及图像生成、视频生成等。

---

## 六、自动化与 Cron

```bash
hermes cron add "每天 9 点给我发今日天气"  # 自然语言创建定时任务
hermes cron list                            # 查看所有任务
hermes cron run <job-id>                    # 手动触发
```

支持向任意平台投递结果。

---

## 七、会话管理

```bash
hermes sessions list              # 列出会话
hermes sessions export <id>       # 导出会话
hermes sessions resume <id>       # 恢复之前的会话
hermes --resume <name>            # 启动时直接恢复
hermes --continue                 # 继续最近一次会话
```

---

## 八、Profiles（多套环境隔离）

```bash
hermes profile create work        # 创建工作 profile
hermes profile create personal    # 创建个人 profile
hermes profile list               # 列出所有
hermes profile use work           # 切换到工作 profile
```

每个 profile 有独立的配置、记忆、会话历史。

---

## 九、调试与维护

```bash
hermes doctor           # 检查配置和依赖是否正常
hermes status           # 所有组件状态一览
hermes logs             # 查看日志
hermes dump             # 输出完整诊断信息（用于反馈 bug）
hermes update           # 升级到最新版
hermes backup           # 备份整个 Hermes 目录为 zip
hermes import           # 恢复备份
hermes uninstall        # 卸载
```

---

## 十、从 OpenClaw 迁移

```bash
hermes claw migrate     # 一键迁移配置、数据、工作区
```

---

## 十一、关键路径速查

| 内容 | 路径 |
|------|------|
| 安装目录 | `~/.hermes/hermes-agent/` |
| CLI 入口 | `~/.local/bin/hermes` |
| 配置文件 | `~/.hermes/config.yaml` |
| API Key | `~/.hermes/.env` |
| 认证状态 | `~/.hermes/auth.json` |
| Skills | `~/.hermes/skills/` |
| 记忆 | `~/.hermes/memories/` |
| 会话 | `~/.hermes/sessions/` |
| 日志 | `~/.hermes/logs/` |
| Cron | `~/.hermes/cron/` |
| 插件 | `~/.hermes/hermes-agent/plugins/` |

---

## 十二、实用技巧

- **后台长期运行**：`hermes gateway install && hermes gateway start`（systemd 服务）
- **脚本集成**：用 `-z` 单次模式，输出纯文本可直接管道处理
- **免审批**：`--yolo` 跳过工具调用审批，自动执行
- **忽略本地规则**：`--ignore-rules`（测试时用）
- **Web Dashborad**：`hermes dashboard` 启动 Web 管理界面
- **Shell 补全**：`hermes completion bash > ~/.bashrc.d/hermes` 然后 source
