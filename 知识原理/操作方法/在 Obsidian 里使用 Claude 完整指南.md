# 在 Obsidian 里使用 Claude 完整指南

> 把 AI 嵌入到你的笔记库,让 Claude 直接读写笔记、跨文档搜索、批量改写。
> 本文涵盖:Claudian 插件安装、常见踩坑、以及高手们的实战玩法。

---

## 一、Claudian 是什么?能做什么?

**Claudian** 是一个 Obsidian 第三方插件,作者 [YishenTu](https://github.com/YishenTu/claudian),把 **Claude Code / Codex / OpenCode** 这类命令行 AI 工具嵌入到 Obsidian 侧栏里,让 AI 把你的整个笔记库当作工作目录。

### 它能做什么

```
✅ 读取并理解你的所有笔记
✅ 直接修改、新建、删除笔记文件
✅ 跨笔记搜索 + 信息整合
✅ 选中文字 + 快捷键 → 行内改写(带 diff 预览)
✅ 用斜杠命令 / @mention 调用预设模板和 MCP 服务器
✅ 在你的 vault 里执行 bash 命令、多步任务
```

### 跟其他方案的区别

| 方案 | 适用场景 |
|---|---|
| **Claudian** | 在笔记本里做事 —— 边记边问 AI、整理录音、改写笔记 |
| **MCP 桥接 Claude Code** | 边写代码边查笔记 —— 让 Claude Code 能读你的 Obsidian 知识库 |
| **复制粘贴到 Claude 网页** | 临时性、单次需求 —— 不需要 AI 长期了解你的笔记结构 |

如果你的工作流是**"在 Obsidian 里写笔记 + 想让 AI 帮忙读写"** → Claudian 最对路。
如果是**"在 IDE 里写代码 + 偶尔查笔记"** → 用 MCP 方案更合适。

---

## 二、安装前提

### 你需要先装好这些

```
1. Obsidian       https://obsidian.md/                  (免费笔记软件)
2. Node.js LTS    https://nodejs.org/                   (Claude Code 的依赖)
3. Claude Code    npm 命令安装                          (命令行 AI 工具)
4. Claudian       Obsidian 插件市场 / BRAT              (本文主角)
```

### 你还需要一个 Claude 账号

Claudian **不是免费的 AI** —— 它只是个嫁接层。底层调的是 Claude Code,所以你需要:
- **Claude Pro / Max 订阅**(用于 Claude Code 登录),或
- **Anthropic API Key**(按调用计费)

免费 Claude 账号没法用 Claude Code。

---

## 三、安装步骤

### Step 1:装 Node.js

去 https://nodejs.org/ 下载 **LTS 版本**(左边那个稳定版),一路下一步。

装完打开终端验证:
```bash
node -v
```
看到 `v18` 或更高就行。

### Step 2:装 Claude Code

终端运行:
```bash
npm install -g @anthropic-ai/claude-code
```

⚠️ Mac/Linux 报权限错时前面加 `sudo`,Windows **不要加**。

装完验证:
```bash
claude --version
```
能看到版本号就成功。

### Step 3:首次登录 Claude Code

终端输入:
```bash
claude
```

第一次会跳浏览器登录,授权完就能用。可以发个 "hello" 测一下,然后输 `/exit` 退出。

### Step 4:装 Obsidian + 创建 Vault

如果还没装,从 https://obsidian.md/ 下载。

打开 Obsidian → 创建一个新的 Vault(本质就是一个文件夹)。

### Step 5:装 Claudian 插件

⚠️ **注意:Claudian 目前(2026 年 5 月)不在 Obsidian 官方插件市场**(Community plugins → Browse 里搜不到)。
官方仓库只支持以下两种安装方式:

#### 方法 A:BRAT(推荐 —— 能自动更新)
1. 在 Obsidian Community plugins → Browse 里装 **BRAT** 这个插件
2. BRAT 设置 → Add Beta plugin
3. 填 `https://github.com/YishenTu/claudian`
4. BRAT 自动装好 Claudian
5. 之后 BRAT 会自动检查新版本,有更新会通知你

#### 方法 B:手动安装(适合不想用 BRAT 的人)
1. 去 https://github.com/YishenTu/claudian/releases 下载最新 release
2. 下载这三个文件:`main.js`、`manifest.json`、`styles.css`
3. 在你的 vault 里找到 `.obsidian/plugins/` 文件夹(`.obsidian` 是隐藏文件夹,Mac 按 `Cmd+Shift+.`、Windows 在文件夹选项里勾"显示隐藏文件")
4. 在 plugins 里新建一个文件夹 `claudian`,把上面三个文件放进去
5. 重启 Obsidian → 设置 → Community plugins → 列表里找到 Claudian → 启用

### Step 6:配置 Claude CLI 路径

⚠️ **这一步是大坑,所有 Windows 用户都会遇到。** 单独开一节讲。

---

## 四、关键踩坑:Claude CLI 路径怎么填

这是 Windows 用户**最容易卡死**的地方。错填会一直报 `spawn EINVAL` 错误。

### 先理解 Claude Code 在 Windows 上的本质

当你用 `npm install -g` 装 Claude Code 之后,Windows 上会生成多个文件,跑 `where claude` 会看到:

```
C:\Users\你\AppData\Roaming\npm\claude
C:\Users\你\AppData\Roaming\npm\claude.cmd
```

它们看起来都是 `claude`,但**实际作用不同**,而且**不是哪个都能给 Claudian 用**的。

### 三种填法对应三种安装方式

Claudian 设置里 **Setup → Claude CLI path** 这个输入框,填什么取决于你怎么装的 Claude Code:

| 你的安装方式 | 应该填什么 | 例子 |
|---|---|---|
| **原生安装器(.exe / .pkg)** | `claude.exe` | `C:\Program Files\Claude\claude.exe` |
| **npm / pnpm / yarn 安装** | `cli-wrapper.cjs` | `C:\Users\你\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\cli-wrapper.cjs` |
| **老版本 npm 包**(legacy) | `cli.js` | 同上目录里的 `cli.js`(新版没有这个文件) |

### ❌ 千万不要填的

```
❌ claude(没有后缀)         → Node.js 不知道怎么启动
❌ claude.cmd                → Node.js 18.20+ 安全策略默认不让 spawn .cmd
❌ claude.ps1                → 同上,PowerShell 脚本也被禁
```

如果你已经填错并撞上 `spawn EINVAL`,这就是原因。

### ✅ Windows 用户最稳的做法(npm 安装)

直接填 `cli-wrapper.cjs` 的完整路径:

```
C:\Users\你的用户名\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\cli-wrapper.cjs
```

把 `你的用户名` 换成你自己的。

### Mac / Linux 不用纠结

直接填 `claude` 命令的路径(`which claude` 出来那个)就行。常见的:
```
/usr/local/bin/claude
/opt/homebrew/bin/claude
~/.npm-global/bin/claude
```

也可以**留空**让 Claudian 自动检测(Mac 上一般能成)。

### 改完路径后必须做的事

1. **完全退出 Obsidian** —— 系统托盘里也要退出,不只是关窗口
2. 重新打开
3. 进 Claudian 侧栏发个 "hi" 测试

---

## 五、其他常见问题

### 问题 1:`spawn EINVAL`

**症状**:每发一条消息就跳红 X 报这个错。

**原因**:CLI 路径填错(见上一节)。

**解法**:按 npm 安装方式填 `cli-wrapper.cjs`。

### 问题 2:Claudian 找不到 Claude Code

**症状**:错误提示 "Claude CLI not found" 或 "spawn claude ENOENT"。

**原因**:Claude Code 没装,或装了但没在 PATH 里。

**解法**:
1. 终端跑 `claude --version`,确认能输出版本号
2. 跑 `where claude`(Windows)或 `which claude`(Mac/Linux)拿到完整路径
3. 把这个路径填到 Claudian 设置里

### 问题 3:GUI 应用找不到 Node.js

**症状**:终端里 Claude Code 能跑,但在 Obsidian 里启动不了。

**原因**:GUI 应用的 PATH 跟终端不一样。

**解法**:在 Claudian 设置 → **Environment → Custom variables** 里,加上 Node.js 所在目录到 PATH。或者用 `cli-wrapper.cjs` 这种**不依赖 PATH** 的完整路径填法。

### 问题 4:权限错误 / 一直跳确认弹窗

**症状**:每个文件操作都要点确认。

**解法**:
- **YOLO 模式**(右下角开关):打开后跳过所有确认。⚠️ **不熟时不要开**,会被 AI 改飞。
- **Safe mode 配置**:Claudian 设置里调 Safe mode 为 `acceptEdits`(自动允许编辑)或 `bypassPermissions`(全开)。

### 问题 5:更新插件后行为异常

**解法**:
1. 退出 Obsidian
2. 删除 vault 里的 `.obsidian/plugins/claudian/data.json`(这是配置缓存)
3. 重新进 Obsidian,重新填路径

---

## 六、Claudian 的几个核心用法

进了 Claudian 侧栏后,你可以这样用:

### 直接聊天 + 让它读笔记

```
"读一下 [[今天的会议笔记]],帮我提炼出 3 个 Action Items"
"我的 vault 里有几篇关于 RAG 的笔记?都讲了什么?"
"把 /项目/AI训练师 文件夹里所有 md 文件汇总成一份索引"
```

### 行内编辑(选中文字 + 快捷键)

1. 在笔记里选中一段文字
2. 按快捷键(默认 `Ctrl+Shift+E`)
3. 输入指令,比如"把这段改得更书面"
4. Claude 会给一个**带 diff 的预览**,可以接受或拒绝

### Slash Commands(预设模板)

输入 `/` 触发命令面板,可以定义自己的常用指令模板,比如:
- `/summarize` —— 总结当前笔记
- `/translate` —— 翻译选中段落
- `/expand` —— 把列表展开成正文

### @mention(精确指定上下文)

输入 `@` 来 @ 文件、子代理或外部目录:
- `@today.md` —— 把今天的笔记加进上下文
- `@/项目/AI训练师/` —— 把整个文件夹喂给它
- `@subagent-name` —— 调用子代理

---

## 七、高手们怎么用 Obsidian + Claude(进阶玩法)

> 这部分是真正能改变工作方式的地方。Claudian 只是入口,**真正的玩法在于把 Obsidian 变成一个 AI 第二大脑**。

### 玩法 1:产品经理的 Living Second Brain

> 来自 Medium 上一篇产品经理的工作流分享。

每天早上把所有零散信息扔到 Obsidian 的 `Inbox` 文件夹:
- 手机录的语音备忘
- 转发到笔记的邮件
- 白板拍照
- 临时文字记录

然后只对 Claude 说一句:**"Process my inbox"**(处理我的收件箱)。

它会:
- 把语音转写归档到对应项目
- 邮件提炼出 Action Items
- 白板照片识别成结构化笔记
- 自动建立跨文件的双链

→ 详细案例:[How I Use Claude Code + Obsidian as a Product Manager](https://medium.com/all-about-claude/how-i-use-claude-code-obsidian-as-a-product-manager-4-workflows-that-actually-changed-my-work-bc04360b905d)

### 玩法 2:让笔记库"会学习"(Session Logs 模式)

每次跟 Claude 聊完,让它在 vault 里写一个**结构化总结**:
- 我们讨论了什么
- 你做了什么决定
- 偏好的风格(比如"用简洁要点而非长段落")
- 哪些项目暂停 / 进行中

下次开新会话,Claude 先读最近几次的 session log,**等于有了记忆**。

这种方式不依赖模型自己的上下文,而是**通过结构化文档建立连续性**。

→ 详细教程:[Build an AI Second Brain with Claude Code and Obsidian](https://www.mindstudio.ai/blog/build-ai-second-brain-claude-code-obsidian)

### 玩法 3:Karpathy 的 LLM Wiki 模式

OpenAI 创始团队成员 **Andrej Karpathy** 在 2026 年初发了一份 GitHub gist 叫 *LLM Knowledge Base*,据 codewithseb 等技术博主整理,这份 gist 在社区引发了相当大的反响。

核心思想极简:
```
1. 把所有知识存为 Markdown 文件
2. 让 LLM 看这堆 Markdown
3. AI 自己搜索、连接、扩充内容
```

每加入一个新源(论文 / 视频 / 文章),Claude 自动:
- 提取概念和实体
- 更新交叉引用
- 维护索引页
- **不需要你手动归档**

→ 现成模板:[claude-obsidian](https://github.com/AgriciDaniel/claude-obsidian)(基于 Karpathy 模式的开箱即用 vault)

### 玩法 4:CLAUDE.md —— 给 AI 的"使用说明书"

在 vault 根目录放一个 `CLAUDE.md` 文件,告诉 Claude:
- 我的笔记库是怎么组织的(folder 命名约定)
- 我喜欢什么样的写作风格
- 哪些文件夹是工作区,哪些是归档区
- 写新笔记时的模板

每次 Claude 启动会先读这个文件,**等于一个永久的 system prompt**。

→ 完整指南:[CLAUDE.md & Memory Guide](https://www.codewithseb.com/blog/claude-code-obsidian-second-brain-guide)

### 玩法 5:开发者:笔记 + 代码闭环

把 Obsidian 笔记跟代码项目用 **MCP 服务器**或**符号链接**打通:

```
"搜我 vault 里关于认证设计的笔记,然后基于那些决策重构 token rotation 代码"
```

Claude 能读你**当时的设计思路**,做出符合你原意的重构 —— 而不是凭空猜代码。

→ 实战案例:[Claude Code + Obsidian Workflow](https://self.md/guides/claude-code-obsidian/)

### 玩法 6:任务驱动型工作流(对工程师)

把 Linear / Jira 工单 + Obsidian 项目笔记 + 代码仓库**串成一条流水线**:

```
你只说一句:"Implement TRA-142"

Claude 自动:
1. 拉 Linear 工单详情
2. 读 Obsidian 项目笔记里的相关设计
3. 查 Sentry 看相关报错
4. 写实现计划 → 等你确认 → 写代码 → 跑测试
```

→ 完整 Skill 配置:[How I Use Claude Code](https://www.damiangalarza.com/posts/2025-11-25-how-i-use-claude-code/)

---

## 八、心法:别让笔记变成 AI 的"前置作业"

最后说一个反直觉的提醒,来自 self.md 那篇文章:

> The vault should be a bonus, not a requirement.
> 笔记库应该是加分项,不是前置条件。

意思是:**Claude 在没有 Obsidian 的时候就能用得很好**,有 vault 是锦上添花,但如果"必须先把笔记整理好才能让 AI 干活"——你就把"做事"变成了"整理仪式"。

```
✅ 正确节奏:笔记自然积累 → AI 顺手用一下 vault → 工作更顺
❌ 错误节奏:为了让 AI 用得爽 → 强迫自己结构化所有笔记 → 笔记变负担
```

工具是为你服务的,别反过来。

---

## 九、参考资源

### 官方
- [Claudian GitHub](https://github.com/YishenTu/claudian) —— 主项目仓库,issue 区有最新踩坑解法
- [Claude Code 官方文档](https://docs.claude.com/en/docs/claude-code)
- [Obsidian 官网](https://obsidian.md/)

### 社区高质量教程
- [Claude Code + Obsidian: Build an AI Second Brain That Actually Works](https://www.codewithseb.com/blog/claude-code-obsidian-second-brain-guide) —— 最完整的中级教程
- [Build an AI Second Brain with Claude Code and Obsidian (MindStudio)](https://www.mindstudio.ai/blog/build-ai-second-brain-claude-code-obsidian) —— 系统化方法论
- [Karpathy LLM Wiki Pattern 实现](https://github.com/AgriciDaniel/claude-obsidian) —— 拿来即用的 vault 模板

### 工作流案例
- [产品经理的 4 个 Claude + Obsidian 工作流](https://medium.com/all-about-claude/how-i-use-claude-code-obsidian-as-a-product-manager-4-workflows-that-actually-changed-my-work-bc04360b905d)
- [软件工程师的完整 Claude Code 工作流](https://www.damiangalarza.com/posts/2025-11-25-how-i-use-claude-code/)
- [Claude Code + Obsidian 闭环开发](https://self.md/guides/claude-code-obsidian/)

---

## 十、常用命令速查

### 终端命令

| 命令 | 作用 |
|---|---|
| `node -v` | 看 Node.js 版本 |
| `claude --version` | 看 Claude Code 版本 |
| `where claude`(Win) / `which claude`(Mac) | 找 Claude 路径 |
| `npm root -g` | 看 npm 全局包目录 |
| `npm install -g @anthropic-ai/claude-code` | 装 Claude Code |
| `npm update -g @anthropic-ai/claude-code` | 升级 Claude Code |

### Claudian 内常用操作

| 操作 | 说明 |
|---|---|
| `Ctrl+P` → `Claudian: Open chat` | 打开聊天面板 |
| `@文件名` | 把指定文件加进上下文 |
| `/命令名` | 执行预设命令模板 |
| 选中文字 + 快捷键 | 行内改写 |
| 右下角 YOLO 开关 | 关闭确认提示(慎用) |

---

## 附:正确路径填法的最终速查

```
                                     Claude CLI path 应填
┌────────────────────────────────────────────────────────────────────┐
│ Windows + 原生安装器     →  C:\Program Files\...\claude.exe         │
│ Windows + npm           →  ...\node_modules\...\cli-wrapper.cjs    │
│ Mac/Linux + 任何方式    →  留空(自动检测)或 which claude 输出     │
└────────────────────────────────────────────────────────────────────┘

❌ 永远不要填:claude(无后缀)、claude.cmd、claude.ps1
```

记住这一条,90% 的安装问题都能避免。
