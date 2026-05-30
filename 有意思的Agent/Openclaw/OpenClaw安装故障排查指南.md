# OpenClaw安装故障排查指南

本文档记录了 OpenClaw（AI 小龙虾）从安装初始化到故障修复、正常使用的完整流程，涵盖了新版本兼容问题、环境变量问题、服务启动失败等常见故障的详细解决步骤，可用于后续同类问题的排查参考。

---

## 一、问题背景

本次部署过程中，遇到了一系列新版本兼容与环境配置类问题，完整问题链如下：

1. 旧版本配置格式不兼容，Gateway 绑定参数校验失败

2. 全局命令无法找到，系统环境变量 `PATH` 缺失

3. 旧的 Systemd 服务内置了过时的启动参数，导致 Gateway 服务一启动就崩溃退出
最终通过修复配置、更新服务参数完成了服务的正常部署。

---

## 二、故障排查与解决步骤

### 2\.1 初始配置格式不兼容问题

#### 故障现象

首次启动时提示配置校验失败：

```Plain Text
Invalid config at /home/salim/.openclaw/openclaw.json:
- gateway.bind: Invalid input (allowed: "auto", "lan", "loopback", "custom", "tailnet")
```

**原因**：新版本的 OpenClaw 废弃了旧版本中直接填写 IP 地址（如 `0\.0\.0\.0`/`localhost`）的 bind 配置方式，现在仅支持预设的绑定模式。

#### 解决方法

执行官方的自检修复命令，自动将旧配置转换为新版本兼容的格式：

```bash
openclaw doctor --fix
```

---

### 2\.2 `openclaw: 未找到命令` 环境变量问题

#### 故障现象

执行命令时系统提示找不到命令：

```Plain Text
openclaw: 未找到命令
```

**原因**：OpenClaw 安装后的命令目录没有加入系统的环境变量 `PATH`，系统无法定位到可执行文件。

#### 临时解决：直接调用本地可执行文件

在不修改环境变量的情况下，可以直接定位到本地的二进制文件运行，绕过 PATH 问题：

```bash
# 进入 OpenClaw 的安装根目录
cd ~/.openclaw
# 直接运行本地的 openclaw 程序，后面加需要执行的命令
./bin/openclaw doctor --fix
```

---

### 2\.3 永久修复环境变量 PATH

#### 故障提示

安装过程中给出了明确的 PATH 修复提示：

```Plain Text
! PATH missing npm global bin dir: /home/salim/.hermes/node/bin
  This can make openclaw show as "command not found" in new terminals.
  Fix (zsh: ~/.zshrc, bash: ~/.bashrc):
    export PATH="/home/salim/.hermes/node/bin:$PATH"
```

#### 解决步骤

根据你使用的终端类型，将路径永久加入环境变量：

##### ① Bash 终端（大部分 Linux 系统默认终端）

```bash
# 把路径追加到 bash 的配置文件末尾
echo 'export PATH="/home/salim/.hermes/node/bin:$PATH"' >> ~/.bashrc
# 立刻加载配置，不用重启终端
source ~/.bashrc
```

##### ② Zsh 终端（MacOS 默认终端、部分自定义终端）

```bash
# 把路径追加到 zsh 的配置文件末尾
echo 'export PATH="/home/salim/.hermes/node/bin:$PATH"' >> ~/.zshrc
# 立刻加载配置
source ~/.zshrc
```

---

### 2\.4 Gateway 服务启动失败问题

#### 故障现象

执行启动命令后，服务状态显示启动失败：

```Plain Text
Runtime: stopped (state failed, sub failed, last exit 1, reason 1)
Connectivity probe: failed
Probe target: ws://127.0.0.1:18789
  connect ECONNREFUSED 127.0.0.1:18789
```

服务启动后立刻崩溃退出，本地连接被拒绝。

排查后定位到根因：旧的 Systemd 服务文件里，内置了过时的启动参数：

```Plain Text
Command: /home/salim/.hermes/node/bin/node /home/salim/.hermes/node/lib/node_modules/openclaw/dist/index.js gateway --port 18789 --bind 0.0.0.0
```

这里的 `\-\-bind 0\.0\.0\.0` 是旧版本的参数，新版本已经不支持该格式，直接导致服务启动就崩溃。

#### 解决步骤

1. **卸载旧的错误服务**：先把内置了错误参数的旧服务删掉

```bash
openclaw gateway uninstall
```

2. **重新修复配置**：再次执行自检，确保所有配置都是新版本兼容的

```bash
openclaw doctor --fix
```

3. **重新安装正确的服务**：重新生成服务文件，这次会使用新版本兼容的启动参数

```bash
openclaw gateway install
```

4. **启动服务**：

```bash
openclaw gateway start
```

---

## 三、服务正常启动后的使用指南

### 3\.1 验证服务状态

确认服务是否正常运行：

```bash
openclaw gateway status
```

正常运行时会显示 `running` 状态，连接探测会显示成功。

### 3\.2 终端交互式聊天

不用打开浏览器，直接在终端里和 AI 交互聊天：

```bash
# 启动终端交互界面（TUI）
openclaw tui
```

TUI 常用操作：

- 输入消息后按 `Enter` 发送

- 按两次 `Ctrl\+C` 退出 TUI

- 快捷键：

    - `Ctrl\+L`：切换 AI 模型

    - `Ctrl\+G`：切换智能体

    - `Ctrl\+P`：切换历史会话

- 斜杠命令（输入框内直接输入）：

    - `/new`：新建空白会话

    - `/model`：查看 / 切换当前模型

    - `/skills`：查看已启用的工具技能

    - `/help`：查看所有命令帮助

### 3\.3 单次命令行查询

如果只是需要快速的单次查询，不用进入交互模式，可以直接用 agent 命令：

```bash
openclaw agent --message "帮我写一个批量重命名图片的shell脚本"
```

执行后会直接返回 AI 的回复，执行完就退出。

### 3\.4 Web 管理面板

如果需要用网页界面管理，可以执行命令打开管理面板：

```bash
openclaw dashboard
```

会自动弹出浏览器，默认访问地址：`http://127\.0\.0\.1:18789/`

---

## 四、常用命令速查

|功能|命令|
|---|---|
|后台启动 Gateway 服务|`openclaw gateway start`|
|停止 Gateway 服务|`openclaw gateway stop`|
|重启 Gateway 服务|`openclaw gateway restart`|
|检查服务运行状态|`openclaw gateway status`|
|自检并修复配置错误|`openclaw doctor \-\-fix`|
|启动终端聊天界面|`openclaw tui`|
|打开 Web 管理面板|`openclaw dashboard`|
|安装开机自启服务|`openclaw gateway install`|
|卸载系统服务|`openclaw gateway uninstall`|
|查看服务日志|`journalctl \-\-user \-u openclaw\-gateway\.service \-n 200 \-\-no\-pager`|

> （注：文档部分内容可能由 AI 生成）
