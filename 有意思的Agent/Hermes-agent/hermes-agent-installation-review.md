# Hermes Agent 安装复盘

> 日期：2026-05-17  
> 指令：了解 hermes-agent 并安装

---

## 一、整体规划

1. 信息收集：搜索并阅读项目主页、文档，了解是什么、怎么装、有无冲突风险
2. 安全准备：备份 openclaw.json，检查 ~/.hermes 目录冲突
3. 执行安装：curl | bash 一键安装
4. 手动配置：安装脚本的交互式向导在非 TTY 环境失败，手动补齐 provider/model/api key 配置
5. 验证：hermes -z 命令行模式发送测试 prompt，确认 API 调用正常
6. 固化：写入 .env、config.yaml、auth.json，记录到 memory

---

## 二、使用的工具

| 工具 | 用途 | 是否使用 Skill |
|------|------|---------------|
| `web_search` | 搜索 hermes-agent 项目信息 | 否 |
| `web_fetch` ×3 | 读官网、中文社区、GitHub README | 否 |
| `memory_search` | 找回已有的 DeepSeek API key 配置 | 否 |
| `exec` (shell) ×15+ | 环境检查、curl 安装、grep 分析脚本、修改配置、测试 API、查源码定位 provider | 否 |
| `read` | 读 hermes 源码 auth.py、agent_init.py、auxiliary_client.py | 否 |

**没有使用任何 OpenClaw 内置 Skill**——这是纯系统安装操作，所有步骤靠基础工具链完成。

---

## 三、关键技术决策

### 3.1 安装前检查冲突

Hermes 默认安装到 `~/.hermes/hermes-agent`，但 `~/.hermes/` 已被 OpenClaw 的 `node` 子目录占用。提前 `grep` 分析了安装脚本的 `check_node()` 和 `install_node()` 函数逻辑，确认：
- `check_node()` 先检测系统中已有的 node → 不会触发 `install_node()`
- 不会删除已有的 `~/.hermes/node`
- 安全，直接安装

### 3.2 Provider 配置

Hermes 不支持通用的 `openai_compatible` provider 名。通过直接看源码 `hermes_cli/auth.py` 中的 `PROVIDER_REGISTRY` 发现 `deepseek` 是内置 provider：

```python
"deepseek": ProviderConfig(
    id="deepseek",
    name="DeepSeek",
    auth_type="api_key",
    inference_base_url="https://api.deepseek.com/v1",
    api_key_env_vars=("DEEPSEEK_API_KEY",),
    base_url_env_var="DEEPSEEK_BASE_URL",
),
```

### 3.3 Auth 分两层

Hermes 的配置不是单文件：
- `~/.hermes/config.yaml` → 声明 model 和 provider
- `~/.hermes/auth.json` → 持久化 provider 认证状态（`active_provider` + 每个 provider 的元数据）
- `~/.hermes/.env` → API key 实际值

三者缺一不可。`resolve_provider_client()` 先读 auth.json 找 active_provider，再从 PROVIDER_REGISTRY 拿配置，最后通过 `get_env_value()` 从 `.env` 或 `os.environ` 取 API key。

---

## 四、遇到的坑

| 问题 | 现象 | 根因 | 解决 |
|------|------|------|------|
| 配置向导失败 | `bash: 行 1490: /dev/tty: 没有那个设备或地址` | 非 TTY 环境（OpenClaw exec）下 `hermes setup` 需要 /dev/tty | 手动写 auth.json + config.yaml + .env |
| Unknown provider | `openai_compatible` 不被识别 | Hermes 没有这个 provider 名 | 查源码 → 用内置 `deepseek` provider |
| No LLM provider configured | auth.json 缺失 | 只改了 config.yaml 没写 auth store | 手动写 auth.json，设 active_provider |
| hermes -z 超时无输出 | 命令等待 30s 后 kill 无返回 | DeepSeek v4-pro 是推理模型，`max_tokens` 太小（10）时 reasoning token 吃光输出空间 | `PYTHONUNBUFFERED=1` + 确认 API 正常（max_tokens=50 时返回 OK） |
| SearXNG baseUrl 丢失 | web_search 报错 SearXNG base URL not configured | 之前的配置热加载被冲掉（仅剩 enabled:true 无 config） | 恢复 plugins.entries.searxng.config.webSearch.baseUrl |

---

## 五、安装结果

- **版本**: Hermes Agent v0.14.0 (2026.5.16)
- **路径**: `~/.hermes/hermes-agent`，CLI: `~/.local/bin/hermes`
- **模型**: DeepSeek v4-pro，通过 `~/.hermes/.env` 配 `DEEPSEEK_API_KEY`
- **Skills**: 87 个内置（含 notion、blogwatcher、himalaya、多搜索引擎等）
- **验证通过**: `hermes -z "reply with only the word OK"` → 输出 `OK`

---

## 六、教训

1. **先跑后查**：安装类是确定性操作，一条 curl | bash 先干，出问题再修，不必先读三份文档
2. **调研的颗粒度要恰当**：重点关注「能不能装」「会不会冲突」，而不是「怎么打开 PowerShell」
3. **看源码比猜配置更快**：provider 名不确定时，直接在源码里 grep `PROVIDER_REGISTRY`，2 秒解决问题
4. **非 TTY 环境要注意**：很多 CLI 配置向导依赖 /dev/tty，需要预留手动配置时间
5. **推理模型的 token 预算**：DeepSeek v4-pro 的 reasoning token 会挤占输出 token，默认 max_tokens 设置要足够大
