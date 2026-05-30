# OpenClaw 命令行操作指南

> 本文档介绍 OpenClaw（版本 2026.5.12）的日常命令行操作，包含常用命令、参数拆解和实用示例。
>
> 当前配置路径：`~/.openclaw/openclaw.json`
> CLI 路径：`/home/salim/.hermes/node/bin/openclaw`
>
> 提示：所有命令均可添加 `--help` 查看完整参数。

---

## 目录

1. [基础概念](#基础概念)
2. [状态查看](#状态查看)
3. [Gateway 网关管理](#gateway-网关管理)
4. [通道管理（微信等）](#通道管理微信等)
5. [配置管理](#配置管理)
6. [插件管理](#插件管理)
7. [模型管理](#模型管理)
8. [定时任务](#定时任务)
9. [会话管理](#会话管理)
10. [诊断与修复](#诊断与修复)
11. [更新与升级](#更新与升级)
12. [技能管理](#技能管理)
13. [常用速查表](#常用速查表)

---

## 基础概念

| 概念 | 说明 |
|------|------|
| **Gateway** | OpenClaw 的核心服务进程，WebSocket 网关 |
| **Channel** | 消息通道（WebChat / 微信 / Telegram / WhatsApp 等） |
| **Plugin** | 功能扩展插件 |
| **Agent** | AI 助手实例，可以有多个（如 main、work 等） |
| **Session** | 一次对话会话 |
| **Skill** | AI 技能包，提供专业指令 |
| **Cron Job** | 定时任务 |

---

## 状态查看

### 综合状态

```bash
openclaw status
```

显示网关状态、通道健康、当前模型、会话摘要。**最常用的命令**。

```bash
openclaw status --deep       # 深度探测，检查通道连接状态
openclaw status --all        # 完整诊断报告（只读，适合粘贴）
openclaw status --json       # JSON 格式输出（机器可读）
openclaw status --usage      # 显示 API 用量/配额
```

**输出包含：**
- Gateway 服务状态（运行中/停止）
- 通道列表及连接状态
- 当前会话 token 用量
- 安全审计建议

---

## Gateway 网关管理

Gateway 是 OpenClaw 的心脏，所有通道和 agent 都连到它。

```bash
openclaw gateway status       # 查看网关状态
openclaw gateway restart      # 重启网关（改配置后常用）
openclaw gateway start        # 启动网关服务
openclaw gateway stop         # 停止网关服务
openclaw gateway run          # 前台运行（调试用）
openclaw gateway health       # 获取详细健康信息
openclaw gateway discover     # 局域网/广域网发现网关
```

**常见场景：**
- 改了 `openclaw.json` 配置 → `openclaw gateway restart`
- 新装了插件/通道 → `openclaw gateway restart`
- 网关挂了 → `openclaw gateway status` 看看，然后 `openclaw gateway restart`

---

## 通道管理（微信等）

通道就是你和 OpenClaw 通信的入口。

```bash
openclaw channels list                    # 列出已配置的通道
openclaw channels list --all              # 列出所有可用通道（含未安装）
openclaw channels status                  # 通道连接状态
openclaw channels login --channel <名称>   # 登录通道（扫码等）
openclaw channels logout --channel <名称>  # 登出
openclaw channels add                     # 交互式添加通道
openclaw channels remove --channel <名称>  # 移除通道
openclaw channels logs                    # 查看通道日志
```

**本机当前通道：**
| 通道 | 状态 |
|------|------|
| `webchat` | ✅ 运行中 |
| `openclaw-weixin` | ✅ 运行中 |

**微信登录示例：**
```bash
openclaw channels login --channel openclaw-weixin
# 终端显示二维码 → 手机微信扫码确认
openclaw gateway restart
```

---

## 配置管理

OpenClaw 的配置存储在 `~/.openclaw/openclaw.json`。

```bash
openclaw config file                        # 查看配置文件路径
openclaw config get <路径>                   # 读取某个配置项
openclaw config set <路径> <值>              # 设置配置项
openclaw config unset <路径>                 # 删除配置项
openclaw config validate                    # 验证配置文件合法性
openclaw config schema                      # 查看配置 JSON Schema
```

**常用示例：**

```bash
# 查看当前模型
openclaw config get agents.defaults.model.primary

# 修改网关端口
openclaw config set gateway.port 19001

# 启用插件
openclaw config set plugins.entries.openclaw-weixin.enabled true

# 验证配置是否有误
openclaw config validate

# 配置交互式向导
openclaw configure
```

---

## 插件管理

```bash
openclaw plugins list              # 列出已安装插件
openclaw plugins install <包名>     # 安装插件
openclaw plugins uninstall <包名>   # 卸载插件
openclaw plugins enable <包名>      # 启用插件
openclaw plugins disable <包名>     # 禁用插件
openclaw plugins inspect <包名>     # 查看插件详情
openclaw plugins search <关键词>    # 搜索 ClawHub 插件
openclaw plugins update            # 更新所有插件
```

**示例：**
```bash
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw plugins enable openclaw-weixin
openclaw plugins doctor            # 检查插件加载问题
```

---

## 模型管理

```bash
openclaw models list              # 列出可用模型
openclaw models status            # 当前模型状态
openclaw models set <模型ID>      # 设置默认模型
openclaw models auth              # 管理 API 认证
openclaw models scan              # 扫描 OpenRouter 免费模型
```

**示例：**
```bash
# 查看当前使用什么模型
openclaw models status

# 切换到另一个模型（如有）
openclaw models set openai/gpt-4o
```

---

## 定时任务

```bash
openclaw cron status              # 定时任务调度器状态
openclaw cron list                # 列出所有定时任务
openclaw cron add                 # 添加定时任务
openclaw cron edit <任务ID>       # 编辑定时任务
openclaw cron rm <任务ID>         # 删除定时任务
openclaw cron enable <任务ID>     # 启用任务
openclaw cron disable <任务ID>    # 禁用任务
openclaw cron run <任务ID>        # 立即运行一次
openclaw cron runs <任务ID>       # 查看运行历史
```

---

## 会话管理

```bash
openclaw sessions                    # 列出所有会话
openclaw sessions --active 120       # 最近 120 分钟内的会话
openclaw sessions --agent main       # 查看指定 agent 的会话
openclaw sessions --json             # JSON 格式
openclaw sessions --limit 25         # 显示最近 25 个会话
openclaw sessions cleanup            # 清理过期会话
```

---

## 诊断与修复

```bash
openclaw doctor                     # 运行健康检查
openclaw doctor --fix               # 自动修复常见问题
openclaw doctor --force             # 强制修复（会覆盖自定义配置）
openclaw doctor --deep              # 深度扫描（含系统服务检查）
openclaw doctor --non-interactive   # 无交互模式
```

**常见场景：**
```bash
# 通道连不上 → 诊断
openclaw doctor

# 配置改了不生效 → 修复 + 重启
openclaw doctor --fix
openclaw gateway restart
```

---

## 更新与升级

```bash
openclaw update status                       # 查看版本和更新通道
openclaw update                              # 更新到最新稳定版
openclaw update --channel beta               # 切换到 beta 通道
openclaw update --dry-run                    # 预览更新（不实际执行）
openclaw update --no-restart                 # 更新但不重启网关
openclaw update --yes                        # 非交互式更新
openclaw update --json                       # JSON 格式输出
```

**注意：**
- 更新后网关会自动重启（除非加 `--no-restart`）
- 降级需要确认
- 工作目录有未提交修改时可能会跳过更新

---

## 技能管理

```bash
openclaw skills list               # 列出可用技能
openclaw skills check              # 检查哪些技能就绪
openclaw skills info <技能名>       # 查看技能详情
openclaw skills install <技能名>    # 安装技能
openclaw skills search <关键词>     # 搜索技能
openclaw skills update             # 更新已安装技能
```

---

## 常用速查表

| 场景 | 命令 |
|------|------|
| 看看跑得咋样 | `openclaw status` |
| 全面检查 | `openclaw status --all` |
| 改配置后重启 | `openclaw gateway restart` |
| 查看通道状态 | `openclaw channels status` |
| 修复出问题了 | `openclaw doctor --fix` |
| 微信扫码登录 | `openclaw channels login --channel openclaw-weixin` |
| 查看当前模型 | `openclaw models status` |
| 更新 OpenClaw | `openclaw update` |
| 查看会话列表 | `openclaw sessions` |
| 查看日志 | `openclaw gateway logs` 或 `openclaw channels logs` |
| 配置文件路径 | `openclaw config file` |
| 交互式配置 | `openclaw configure` |
| 安装插件 | `openclaw plugins install <包名>` |

---

## 相关文档

- 完整 CLI 文档：https://docs.openclaw.ai/cli
- 配置参考：https://docs.openclaw.ai/gateway/configuration
- FAQ：https://docs.openclaw.ai/faq

---

> 生成时间：2026-05-17 03:00 CST
> 适用版本：OpenClaw 2026.5.12
