# OpenClaw 自定义模型添加指南

本文档介绍如何在 OpenClaw Gateway 中添加自定义大语言模型（包括 OpenAI 兼容的私有部署模型、第三方云模型等）。

---

## 概述

OpenClaw 的模型配置分为**两个独立的位置**，缺一不可：
1.  **Provider 定义**：声明模型所在的 API 端点、认证方式、模型能力规格
2.  **Agent 可用列表**：把模型注册到 Agent 的运行时允许列表，让它在 Control UI 里可见、可切换

---

## 第一步：准备信息

在开始之前，你需要准备这些信息：

| 字段 | 说明 |
|------|------|
| `baseUrl` | API 端点的根地址，例如 `https://api.example.com/v1` |
| `apiKey` | 你的 API 密钥 |
| `model id` | 后端模型的 ID 标识 |
| `contextWindow` | 上下文窗口大小（token 数） |
| `maxTokens` | 最大输出 token 数 |
| `能力` | 是否支持推理（reasoning）、多模态输入（image/audio）等 |

绝大多数新模型都兼容 OpenAI `/v1/chat/completions` 协议，使用 `api: "openai-completions"` 即可。

---

## 第二步：配置 Provider

打开 OpenClaw 主配置文件：`~/.openclaw/openclaw.json`

在 `models.providers` 对象里新增你的 provider 条目：

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "my-custom-provider": {
        "baseUrl": "https://api.your-endpoint.com/v1",
        "api": "openai-completions",
        "apiKey": "sk-xxxxxxxxxxxxxxxx",
        "models": [
          {
            "id": "your-model-id-here",
            "name": "你的模型名称（显示用）",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 128000,
            "maxTokens": 8192,
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            }
          }
        ]
      }
    }
  }
}
```

字段说明：
- `reasoning`: `true` 表示模型返回思维链（DeepSeek R1、Doubao Seed 系列等）
- `input`: 能力列表，支持 `["text"]` 纯文本 或 `["text", "image"]` 多模态
- `cost`: 本地估算用，0 表示免费/不记账

---

## 第三步：注册到 Agent 可用列表

只配置 Provider 还不够，模型不会出现在 Control UI 里。你需要在 `agents.defaults.models` 补充一条：

```json
{
  "agents": {
    "defaults": {
      "workspace": "/home/you/.openclaw/workspace",
      "model": {
        "primary": "custom-api-deepseek-com/deepseek-v4-pro"
      },
      "models": {
        "custom-api-deepseek-com/deepseek-v4-pro": {},
        "alibaba-dashscope/qwen3.7-max": {},
        "my-custom-provider/your-model-id-here": {}
      }
    }
  }
}
```

这一步是让 Agent 知道这个模型**允许在运行时使用**，没有这一步 Control UI 的模型选择器不会显示它。

---

## 第四步：验证 JSON 格式并重启

检查 JSON 是否合法：

```bash
python3 -m json.tool ~/.openclaw/openclaw.json > /dev/null && echo "OK"
```

重启 Gateway 加载新配置：

```bash
systemctl --user restart openclaw-gateway
```

几秒钟后 Gateway 恢复运行。

---

## 第五步：确认生效

用 CLI 命令查看已注册的模型列表：

```bash
openclaw models list
```

输出里应当能看到你刚添加的模型条目，类似：

```
Model                                      Input      Ctx         Local Auth
my-custom-provider/your-model-id-here      text+image 128k        no    yes
```

刷新 Control UI 页面，新模型就会出现在「模型选择」下拉列表中了。

---

## 常见问题

**Q：改完配置模型还是不显示？**
-  检查是不是忘了加 `agents.defaults.models` 那一条
-  确认 Gateway 已经重启完成
-  Control UI 页面强制刷新（Ctrl+Shift+R）清除前端缓存

**Q：中文模型名会有编码问题吗？**
-  不会。JSON 是 UTF-8 标准，OpenClaw 全链路支持 Unicode 中文显示。

**Q：配置前需要备份吗？**
-  强烈建议每次改 `openclaw.json` 前先备份一份，防止格式错误导致 Gateway 启动失败。

**Q：支持多少种 provider/模型？**
-  没有硬上限。多个 provider 可以共存，网关运行时自动轮询。
