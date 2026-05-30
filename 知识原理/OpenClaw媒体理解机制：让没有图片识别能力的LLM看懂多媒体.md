# OpenClaw 媒体理解机制：如何让不支持图片/音频的 LLM API「听懂、看懂」多媒体内容

## TL;DR

OpenClaw 在模型（如 DeepSeek-V4-Pro）不支持图片/音频输入时，会 **自动调用一个支持多模态的「中间模型」先描述图片或转录音频**，再把文本结果拼进上下文，让只能读文字的模型间接「看懂」「听懂」多媒体内容。

---

## 1. 它能做什么

- **图片理解**：你在任何支持的渠道（WebChat、Discord、Telegram、WeChat 等）发一张图片，OpenClaw 会返回图片内容的描述。
- **音频转写**：发一段音频，OpenClaw 会返回语音转文字的结果。
- **视频描述**：发视频片段，OpenClaw 会提取关键帧并描述。

这一切 **不需要你的聊天模型本身支持多模态输入**。

---

## 2. 它是怎么实现的（完整链路）

### 2.1 图片理解链路

```
你发图片
    ↓
OpenClaw Gateway 收到消息
    ↓
检查你的聊天模型（如 deepseek-v4-pro）是否在 inputs 里声明了 image
    ↓ NO ── 图片被存入磁盘（offload）
    ↓
OpenClaw 遍历你的 provider 列表，找第一个 inputs 包含 image 的模型
    ↓ 如 alibaba-dashscope/qwen3.6-plus（你在 openclaw.json 里配的）
    ↓
向 qwen3.6-plus 发 API 调用，带着图片 + prompt: "Describe the image."
    ↓
qwen3.6-plus 返回一段描述文本（如 "This is a screenshot of a RedNote post...")
    ↓
描述文本被格式化为：
    [Image]
    Description:
    <qwen3.6-plus 返回的描述>
    ↓
这段文本嵌入到发给 deepseek-v4-pro 的消息中
    ↓
deepseek-v4-pro 「看到」这段文字描述，据此回复你
```

### 2.2 上下文注入格式

最终嵌入到聊天消息中的格式长这样：

```
[Image]
Description:
This is a screenshot of a Xiaohongshu user profile page in dark mode...
```

如果你发了图还带了文字（如「这是谁」），格式是：

```
[Image]
User text:
这是谁
Description:
This is a screenshot of a Xiaohongshu user profile page in dark mode...
```

音频同理，`[Image]` 替换为 `[Audio]`，`Description` 替换为 `Transcript`。

### 2.3 默认 Prompt

图片描述的默认 prompt 只有 **四个英文单词**：

```
Describe the image.
```

没有中文指令，没有「请描述细节」，没有任何额外指引。这也是为什么 qwen3.6-plus 返回的描述偏向简洁、偶有省略的原因。

---

## 3. 前置条件

要让这个功能正常工作，你必须满足以下条件：

### 3.1 配置层面

| 条件 | 说明 |
|------|------|
| 至少配置了一个 **支持 image input** 的 provider | 如 `alibaba-dashscope/qwen3.6-plus`、`lm-studio/google/gemma-4-e4b`、`volces-ark/doubao-seed-2-0-lite` 等 |
| 该 provider 的 API key 有效、余额充足 | 每次触发图片理解都要消耗一次额外 API 调用 |
| Gateway 网络可达该 provider 的 API 端点 | 阿里云百炼、OpenRouter 等端点未被墙或代理阻断 |

### 3.2 配置示例（openclaw.json 片段）

```json
{
  "models": {
    "providers": {
      "alibaba-dashscope": {
        "apiKey": "sk-xxxxxxxxxxxxxxxx",
        "baseURL": "https://dashscope.aliyuncs.com/compatible-mode/v1",
        "models": [
          {
            "id": "qwen3.6-plus",
            "input": ["text", "image"],
            "capabilities": {
              "vision": true
            }
          }
        ]
      }
    }
  }
}
```

关键字段是 `"input": ["text", "image"]`——OpenClaw 依靠这个字段判断一个模型能否处理图片。

### 3.3 渠道层面

- 目前确认 **WebChat 控制台** 直接上传图片可以触发。
- 微信 / Discord / Telegram 等渠道取决于图片是否以二进制形式传入 Gateway，还是只传了个 URL。
- 如果消息图片只是一个外部链接（如豆瓣图片 URL），文件不在 Gateway 可访问的路径，可能不会走此链路。

---

## 4. 局限性（这很重要）

### 4.1 描述质量受限于中间模型

- 描述能力 = qwen3.6-plus 的水平，而非 deepseek-v4-pro 的水平。
- 默认 `"Describe the image."` 太简单，中间模型可能忽略细节、截断长文本、错误识别界面元素。
- **丢失的视觉信息不可恢复**：如果中间模型描述错误，deepseek-v4-pro 只能基于错误描述回答问题。

### 4.2 额外成本

| 成本项 | 说明 |
|--------|------|
| Token 消耗 | 描述文本本身消耗 context window（可能几百到几千 token） |
| API 费用 | qwen3.6-plus 的调用也要计费（图片按分辨率 token 化） |
| 延迟增加 | 多一次 API 往返（100ms - 3s 不等） |

尤其是高分辨率大图，qwen3.6-plus 的图片 token 消耗可能很可观。

### 4.3 只支持静态图片

- 不支持交互式 UI 理解（如「点击第三个按钮」）。
- 不支持多帧对比或时序理解。
- 不支持 OCR 内文本的精确提取（除非中间模型 OCR 能力较强且 prompt 要求了）。

### 4.4 语言一致性问题

- 默认 prompt 是英文，中间模型默认返回英文描述。
- 如果你用中文聊天，可能得到英文的图片描述，再在中文上下文中被理解——语义不会错，但多了一层翻译偏差。

### 4.5 中间模型选取规则

- OpenClaw **按 provider 列表顺序** 选取第一个声张 `input: image` 的模型。
- 如果你的 list 里第一个 vision 模型是本地 LM Studio 的 gemma-4-e4b 但没开机，**不会 fallback 到下一个**，只会报错。

### 4.6 无法修改默认 Prompt（在此版本中）

- `"Describe the image."` 写死在代码里，用户无法通过 openclaw.json 配置自定义 prompt。
- 这意味着你无法要求中间模型「用中文描述」「标注出所有文字」「提取 table 数据」等。

---

## 5. 改进建议

### 5.1 短期（不改代码）

- 确保你的多模态 provider 列表把 **最好的 vision 模型放最前面**。
- 可以考虑用 Gemini Flash 2.0、GPT-4o-mini 等替换 qwen3.6-plus（如果 API key 允许）。

### 5.2 长期（提 feature request 或等更新）

- 期望 OpenClaw 未来支持 **自定义 media understanding prompt**（如 `mediaUnderstandingPrompt: "用中文详细描述这张图片，提取所有可见文字"`）。
- 期望支持 **provider 级 fallback**：第一选择挂了能自动尝试下一个。

---

## 6. 源码溯源（2026.5.27）

| 模块 | 文件 | 关键逻辑 |
|------|------|----------|
| 默认 Prompt | `dist/defaults.constants-*.js` | `image: "Describe the image."` |
| 图片描述调用 | `dist/image-*.js` | `describeImageWithModel(params)` → `params.prompt ?? "Describe the image."` |
| 上下文注入格式 | `dist/apply-*.js` → `formatSection()` | `[Image]\nDescription:\n{text}` |
| Provider 选择 | `dist/apply-*.js` → `applyMediaUnderstanding()` | 遍历 providers，取第一个 `inputs` 含 `image` 的 |
| 音频转写 | 同上 | `[Audio]\nTranscript:\n{text}` |
| 视频描述 | 同上 | `[Video]\nDescription:\n{text}` |
| 注入点 | `dist/apply-*.js` → `formatMediaUnderstandingBody()` | 拼接所有 media description 到 context |

---

> **最后**：这个功能本身不是黑魔法，就是「我看不见 → 找个能看见的帮我看一眼 → 把它说的转述给你」。设计朴素但有效——前提是你得配好一个靠谱的 vision model 给它。
