# OpenClaw 工作空间文件指南

> 本文档介绍 OpenClaw 的工作空间（Workspace）中各个核心文件的作用、读写方式，以及如何在它们之间建立跨通道的持久规则。
>
> 工作空间路径：`/home/salim/.openclaw/workspace/`

---

## 目录

1. [工作空间概览](#工作空间概览)
2. [核心文件详解](#核心文件详解)
   - [AGENTS.md — 行为规则与红线](#agentsmd--行为规则与红线)
   - [SOUL.md — 人格与语气](#soulmd--人格与语气)
   - [TOOLS.md — 本地工具笔记](#toolsmd--本地工具笔记)
   - [USER.md — 关于你的信息](#usermd--关于你的信息)
   - [IDENTITY.md — 助手身份](#identitymd--助手身份)
   - [BOOTSTRAP.md — 首次启动](#bootstrapmd--首次启动)
   - [HEARTBEAT.md — 心跳任务](#heartbeatmd--心跳任务)
   - [MEMORY.md — 长期记忆](#memorymd--长期记忆)
   - [memory/ — 每日笔记](#memory--每日笔记)
3. [跨通道建立规则](#跨通道建立规则)
4. [Skills vs Tools 区别](#skills-vs-tools-区别)
5. [实际操作示例](#实际操作示例)

---

## 工作空间概览

```
/home/salim/.openclaw/workspace/
├── AGENTS.md          # 🔴 核心：行为规则、红线、工作方式
├── SOUL.md            # 🟡 核心：人格、语气、边界
├── TOOLS.md           # 🟢 本地笔记（设备名、SSH、偏好等）
├── USER.md            # 🟢 关于你的个人信息
├── IDENTITY.md        # 🟢 我的身份（名字、emoji 等）
├── BOOTSTRAP.md       # ⚪ 首次启动脚本（用完即删）
├── HEARTBEAT.md       # ⚪ 心跳/定时任务清单
├── MEMORY.md          # 🔵 长期记忆（仅主会话加载）
└── memory/            # 🔵 每日笔记（YYYY-MM-DD.md）
```

---

## 核心文件详解

### AGENTS.md — 行为规则与红线

**作用：** 定义 OpenClaw 助手的行为规范、安全红线、工作方式。**这是最重要的规则文件。**

**包含内容：**
- 启动时加载哪些文件
- 记忆系统怎么用（MEMORY.md、每日笔记）
- 安全红线（不泄露隐私、危险操作需确认）
- 内外操作边界（读文件 ✅ / 发邮件 ❌ 要先问）
- 群聊行为规范
- 心跳/定时任务策略
- 平台格式化规则（Discord/WhatsApp）

**怎么改：**
```bash
# 直接编辑
nano ~/.openclaw/workspace/AGENTS.md

# 或者跟我说
"帮我把 AGENTS.md 里的 XX 规则改成 YY"
```

**典型添加规则：**
```markdown
## 自定义规则

- 任何时候不得删除 OpenClawChanges.md 中的记录
- 所有系统级修改必须记录到 OpenClawChanges.md
- 未经用户许可不得安装新软件
```

---

### SOUL.md — 人格与语气

**作用：** 定义我的个性、语气、工作风格。让我不只是"一个聊天机器人"。

**包含内容：**
- 核心信条（真帮忙、有主见、先尝试再问）
- 边界感（隐私不外泄、不确定就问、群聊里不是你的代言人）
- 风格（简洁、有温度、不浮夸）

**怎么改：**
说实话就行——比如「你以后回消息不要太啰嗦」，我会更新 SOUL.md。

---

### TOOLS.md — 本地工具笔记

**作用：** 记录你本地环境的特定信息，和通用 Skills 区分开。

**应该放什么：**
- 摄像头名称和位置
- SSH 主机别名
- 设备昵称
- TTS 声音偏好
- 目录快捷方式

**示例内容：**
```markdown
### SSH
- home-server → 192.168.1.100, user: admin

### 共享文件夹
- Sharing → /run/media/salim/FeatureDisk/Sharing（Samba 无密码共享）

### 磁盘
- FeatureDisk → /dev/sda1, EXT4, 共享盘
- 系统/软件 → Windows NTFS, 已隔离
```

---

### USER.md — 关于你的信息

**作用：** 记录你这个人（称呼、时区、偏好、项目等）。

**当前内容：**（尚未填写完整，初始状态）

**应该填：**
- 怎么称呼你
- 时区：Asia/Shanghai
- 你关注什么项目
- 什么会让你烦 / 什么会让你笑

---

### IDENTITY.md — 助手身份

**作用：** 我的身份定义（名字、物种、风格、emoji）。

**当前内容：** 尚未填写（待你和我一起定义）。

---

### BOOTSTRAP.md — 首次启动

**作用：** 首次启动时的指引。完成后会删除。**当前尚存，说明你还没完成首次对话的自我介绍环节。**

---

### HEARTBEAT.md — 心跳任务

**作用：** 定期检查的任务清单。我每 30 分钟会看一眼这个文件。

**怎么用：**
```markdown
# 每 2 小时检查邮箱
# 每天 9 点检查日历
# 每 4 小时看天气
```

---

### MEMORY.md — 长期记忆

**作用：** 我最重要的持久记忆。只在你和我的直接对话中加载，群聊/共享场景不可见。

**和每日笔记的关系：**
- `memory/YYYY-MM-DD.md` = 原始日志，什么都有
- `MEMORY.md` = 精选记忆，只保留重要的

---

### memory/ — 每日笔记

**作用：** 按天记录发生的事。原始日志，细节丰富。

**格式：** `memory/YYYY-MM-DD.md`

---

## 跨通道建立规则

### 原理

不管你在 WebChat、微信还是任何其他通道给我发消息，**我都是同一个 OpenClaw agent**。规则存在 workspace 文件中，所有通道共享。

### 方法 1：直接在对话中说

在任意通道告诉我规则，我会写进对应文件：

> 「以后任何系统修改都要记录到 OpenClawChanges.md」
> → 我写到 AGENTS.md

> 「记住我的 Github 用户名是 xxx」
> → 我写到 USER.md 或 MEMORY.md

### 方法 2：手动编辑文件

```bash
nano ~/.openclaw/workspace/AGENTS.md
# 加一行规则，保存
openclaw gateway restart  # 如果想立即生效
```

### 方法 3：让我读文件给你确认

> 「帮我看看现在 AGENTS.md 里有哪些规则，念给我听」

### 当前已建立的跨通道规则

| 规则 | 存储位置 | 状态 |
|------|----------|------|
| 所有系统修改记录到 OpenClawChanges.md | AGENTS.md（行为逻辑） | ✅ 生效中 |
| OpenClawChanges.md 中的记录不可删除，需用户许可 | AGENTS.md（行为逻辑） | ✅ 生效中 |
| 跨会话规则不变 | AGENTS.md（行为逻辑） | ✅ 生效中 |

---

## Skills vs Tools 区别

| | Tools（工具） | Skills（技能） |
|------|------|------|
| **是什么** | 基础能力（执行命令、读写文件、搜索等） | 包装了 Tools 的高级指令集 |
| **来源** | OpenClaw 内置 | 手动安装 / ClawHub / 自己写 |
| **能否修改** | ❌ 不能 | ✅ 能写能改能装新的 |
| **存在哪里** | OpenClaw 系统代码 | workspace 或 ClawHub |
| **例子** | `exec`、`read`、`web_search` | `weather`、`healthcheck`、`video-frames` |

### 现有 Skills

| 技能 | 作用 |
|------|------|
| `weather` | 查天气 |
| `healthcheck` | 主机安全审计 |
| `video-frames` | 从视频提取帧/片段 |
| `browser-automation` | 网页自动化控制 |
| `node-connect` | 诊断设备配对 |
| `skill-creator` | 创建/编辑新技能 |
| `taskflow` | 多步骤任务编排 |
| `session-logs` | 搜索分析历史会话 |

### 怎么装新 Skill

```bash
# 查看可用技能
openclaw skills list

# 搜索
openclaw skills search <关键词>

# 安装
openclaw skills install <技能名>

# 查看详情
openclaw skills info <技能名>

# 更新
openclaw skills update
```

---

## 实际操作示例

### 示例 1：建立一条新规则

在微信/WebChat 说：
> 「以后帮我写代码时，注释用中文」

我会在 AGENTS.md 或 SOUL.md 里记录这条偏好。

### 示例 2：记录个人信息

> 「记住我的全名是张三，别叫我 salim 了」

我会更新 USER.md。

### 示例 3：添加设备信息

> 「我的树莓派 IP 是 192.168.3.100，用户名 pi」

我会记到 TOOLS.md。

### 示例 4：创建每日记忆

> 「今天搞定了 Samba 共享，Windows 能访问了」

如果 MEMORY.md 还没建，我会先创建它，然后把重要事件写进去。

---

## 相关文档

- `knowledges/openclaw说明书.md` — OpenClaw CLI 命令大全
- `knowledges/2026-05-17-system-setup.md` — 今日系统配置完整记录

---

> 生成时间：2026-05-17
> 适用版本：OpenClaw 2026.5.12
