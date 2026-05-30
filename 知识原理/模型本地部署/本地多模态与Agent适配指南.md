# 本地多模态与 Agent 适配指南

> 整理自与 Cursor 的讨论。本文聚焦 **本地可部署的原生多模态模型现状**、**Hermes / OpenClaw 等 Agent 框架与本地 LLM（特别是 Qwen3.5-9B）的真实兼容性**，以及在 9B 这种"中等智商"模型上跑 Agent 的实操技巧。

---

## 一、本地原生多模态（Omni）模型现状

### 1.1 重要背景：开源 Omni 的真实能力边界

> **目前开源的"原生多模态"基本都是"多模态输入 + 文本/语音输出"，能"任意输入 + 任意输出"（包括生图生视频）的 Omni 在开源界几乎还没有。**

### 1.2 本地可部署的 Omni 模型（2026 年）

#### 🥇 第一梯队：值得直接上手

| 模型 | 厂商 | 参数 | 输入 | 输出 | 显存 |
|------|------|------|------|------|------|
| **Qwen2.5-Omni-7B** | 阿里 | 7B | 文本/图像/音频/视频 | 文本 + 流式语音 | 16-24 GB |
| **Qwen2.5-Omni-3B** | 阿里 | 3B | 同上 | 同上 | 8-12 GB |
| **MiniCPM-o 2.6** | 面壁 | 8B | 文本/图像/视频/音频 | 文本 + 语音 | 16-20 GB |
| **Phi-4-multimodal** | 微软 | 5.6B | 文本/图像/音频 | 文本 | 12-16 GB |

#### 🥈 第二梯队：特定场景强

| 模型 | 厂商 | 参数 | 特长 | 显存 |
|------|------|------|------|------|
| **GLM-4-Voice** | 智谱 | 9B | 端到端语音对话 | 16-20 GB |
| **Step-Audio 2** | StepFun | 13B | 音频理解+生成 | 24 GB |
| **Baichuan-Omni-1.5** | 百川 | 7B | 全模态理解 | 16 GB |
| **InternLM-XComposer-2.5-OL** | 上海 AI Lab | 7B | 实时流式多模态 | 16 GB |
| **Ola** | 清华 | 7B | 全模态对齐 | 16 GB |

#### 🥉 第三梯队：偏视觉，多模态较弱

| 模型 | 输入 | 输出 |
|------|------|------|
| Llama 3.2 Vision (11B/90B) | 文+图 | 文 |
| Pixtral (12B) | 文+图 | 文 |
| Qwen2.5-VL (7B/72B) | 文+图+视频 | 文 |
| InternVL 3 | 文+图+视频 | 文 |

### 1.3 能力矩阵对比

| 模型 | 文本 | 看图 | 看视频 | 听音频 | 输出文本 | 输出语音 | 生成图像 | 生成视频 |
|------|------|------|--------|--------|----------|---------|----------|----------|
| **Qwen2.5-Omni** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **MiniCPM-o 2.6** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Phi-4-mm** | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **GLM-4-Voice** | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **GPT-4o（参考）** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Gemini 2.5（参考）** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |

**关键观察：开源 Omni 的"输出端"还没破"图像/视频生成"这个堡垒。**

### 1.4 为什么开源 Omni 不能生成图像？

技术上的难点：

1. **离散 token vs 连续表示** —— LLM 输出离散 token，图像需要连续像素
2. **训练数据稀缺** —— 高质量"图文音视频"四模态配对数据极少
3. **训练成本爆炸** —— 要让一个模型同时学好生成图像 + 文本 + 语音，需要 GPT-4o 级别的算力
4. **架构融合难** —— 自回归（文本）vs 扩散（图像）vs 流式（音频）三种范式很难统一

**目前唯一接近"任意输入任意输出"的开源尝试**：
- **AnyGPT**（学术）
- **Janus-Pro**（DeepSeek，能生图但效果一般）
- **Show-o**（学术）
- 这些都还没到"能用"的程度

---

## 二、MoE 多模态：DeepSeek-VL2 与 Aria

### 2.1 时间线

```
2024 年 10 月   Aria 由 Rhymes AI 开源       ← 开源 MoE 多模态先行者
2024 年 12 月   DeepSeek-VL2 系列开源         ← 同 12 月，DeepSeek 三件套齐发
                ├─ DeepSeek-VL2-Tiny  (3B/1B)
                ├─ DeepSeek-VL2-Small (16B/2.8B)
                └─ DeepSeek-VL2       (27B/4.5B)
```

> 注：DeepSeek-VL2 是 DeepSeek-VL（v1，2024 年 3 月）的 MoE 升级版。

### 2.2 各自特点

#### Aria (Rhymes AI) —— 2024 年 10 月

```
参数: 25.3B 总 / 3.9B 激活
专家: 64 个，每次激活 6 个
模态: 文本 + 图像 + 视频 + 长文档
上下文: 64K
```

**特点：**
- 🌟 **第一个开源原生多模态 MoE**（同期还没对手）
- 🌟 **长上下文 64K** 在多模态里很罕见
- 🌟 **视频理解强**（不是把视频拆成几帧那种偷懒）
- ✅ 知识/推理在多模态 MoE 里是顶级
- ⚠️ OCR 不是它强项
- ⚠️ Rhymes AI 是个不太知名的小团队，社区生态相对弱

**适合**：长文档分析、视频理解、需要"看得久"的任务

#### DeepSeek-VL2-Tiny —— 2024 年 12 月

```
参数: 3.4B 总 / 1.0B 激活
模态: 文本 + 图像
```

**特点：**
- ✅ 极小，4070 跑得飞快
- ✅ OCR、文档、图表理解强（继承 DeepSeek-VL 家族传统）
- ⚠️ 推理能力弱，毕竟激活才 1B

**适合**：边缘设备、移动端、批量 OCR

#### DeepSeek-VL2-Small —— 2024 年 12 月（性价比之王）

```
参数: 16.1B 总 / 2.8B 激活
模态: 文本 + 图像
```

**特点：**
- 🌟 **甜点尺寸**：12GB 显卡 4-bit 量化轻松装下
- 🌟 OCR 和文档理解**业界顶级**（小模型里几乎最强）
- 🌟 视觉定位（Visual Grounding）能力强
- ✅ DeepSeek 调校得很好，对话也丝滑
- ⚠️ 不支持视频
- ⚠️ 上下文 4K（短）

**适合**：日常视觉助手、PDF 分析、图表/表格读取

#### DeepSeek-VL2 —— 2024 年 12 月

```
参数: 27.5B 总 / 4.5B 激活
模态: 文本 + 图像
```

**特点：**
- 🌟 DeepSeek-VL2 系列**质量最高**版本
- 🌟 OCR/文档能力进一步提升
- ✅ 综合能力对标 GPT-4V Mini 量级
- ⚠️ 12GB 显卡需要 4-bit + offload
- ⚠️ 速度变慢（4.5B 激活 vs 2.8B）
- ⚠️ 不支持视频，上下文 4K

**适合**：质量优先、不在乎速度的视觉任务

### 2.3 三个模型的两个排序维度

```
按"硬件能否轻松跑动"：
  1. DeepSeek-VL2-Small（最稳，12GB Q4 装下）
  2. Aria 或 DeepSeek-VL2（势均力敌，需要 offload）
  3. DeepSeek-VL2-Tiny（备胎）

按"综合能力 + 多模态完整度"：
  1. Aria（视频 + 长文 + 综合）
  2. DeepSeek-VL2（图像理解最强）
  3. DeepSeek-VL2-Small（性价比之王）
  4. Tiny（轻量）
```

### 2.4 ⚠️ 重要：这些已经被 2026 新一代统一多模态全面碾压

DeepSeek-VL2 / Aria 在 2024 底是 SOTA，但 **2026 年 5 月已经是"上古"** ——
**Qwen3.5-9B 等新一代统一多模态在同样硬件上效果好得多**。

---

## 三、Omni MoE 现状（2026 年 5 月）

**坏消息：原生多模态 MoE Omni（同时听说看说）开源里几乎没有。**

| 模型 | 是否 MoE? |
|------|----------|
| Qwen2.5-Omni | ❌ 稠密 |
| MiniCPM-o 2.6 | ❌ 稠密 |
| Phi-4-multimodal | ❌ 稠密 |
| GLM-4-Voice | ❌ 稠密 |

**Omni 模型还在追"能跑通"阶段，没空做 MoE 优化。** 预计 2026 下半年到 2027 年会出现真正的 MoE Omni。

> **唯一的遗憾**：原生 Omni（语音对话级别）目前还没有 MoE 版本能本地跑，要等 2026 下半年。

---

## 四、为什么 2026 年的 Qwen3.5-9B 比这些"上古多模态"更值得选？

### 4.1 Qwen3.5 系列把"多模态"和"LLM"统一了

**2026 年 2-3 月发布**，**8 个尺寸全是统一多模态**（同一权重既是 LLM 又是 VLM）：

```
0.8B / 2B / 4B / 9B / 27B / 35B-A3B / 122B-A10B / 397B-A17B
```

#### Qwen3.5-9B 的关键 benchmark（2026.05）

| 测试 | Qwen3.5-9B | 对比对象 |
|------|-----------|---------|
| GPQA Diamond（推理） | **81.7** | GPT-OSS-120B 80.1（输给 9B！） |
| MMLU-Pro | **82.5** | — |
| MMMU-Pro（多模态） | **70.1** | GPT-5-Nano 57.2（差 13 分） |
| MathVision | **78.9** | GPT-5-Nano 62.2 |
| OmniDocBench | **87.7** | GPT-5-Nano 55.9 |
| VideoMME | **84.5** | — |

### 4.2 Qwen3.5-9B 能干 / 不能干的事

**能干：**
- 看图说话（VLM）
- 看视频总结
- 文档分析（OCR、图表理解）
- 数学推理、代码生成
- 长上下文（262K，可扩展到 1M via YaRN）

**不能干：**
- ❌ 生成图像（要外挂 Flux）
- ❌ 生成视频（要外挂 Wan）
- ❌ 生成音乐（要外挂 Suno API 或本地 YuE）
- ❌ 端到端语音对话（要外挂 TTS）

---

## 五、Agent 框架与本地 LLM 适配

### 5.1 Hermes 与 OpenClaw 简介

#### Hermes（Nous Research）

| 名字 | 是什么 |
|------|-------|
| **Hermes Agent** | Nous 的**自主 Agent 框架**，68 个内置工具，支持 OpenAI / Anthropic / Codex 三种 API 协议 |
| **Hermes 4** | Nous 的 **LLM 模型**（基于 Llama 微调，强项是函数调用） |

> 这两个名字很容易混。说"Hermes 这个 agent"通常指**前者（Hermes Agent 框架）**。

#### OpenClaw

2026 年初崛起的**开源 Agent 框架**，三大组件：

```
1. CLI Agent     ← 读代码库、终端里跑
2. Gateway       ← 多模型路由层
3. Skills + ClawHub  ← 40 万+ 社区技能市场
```

特色：
- 有持久记忆（`SOUL.md` / `MEMORY.md` / `USER.md`）
- 支持 cron 定时自主执行
- 能接 Telegram / Slack / Discord / GitHub
- **明确支持本地模型（通过 Ollama）**

### 5.2 Agent 对模型的真实要求

**Agent 框架对模型的要求比"聊天"高得多**，主要靠这 4 个能力：

| 能力 | Qwen3.5-9B 表现 |
|------|----------------|
| **函数调用（Tool Calling）** | ✅ 原生支持，质量好 |
| **指令遵循** | ✅ IFEval 91.5（同尺寸顶级） |
| **长上下文** | ✅ 262K（Agent 需要装大量历史） |
| **规划/推理** | ⚠️ 9B 限制下"够用但不顶" |
| **JSON 严格输出** | ✅ Qwen3.5 训练时强化过 |

#### 量化的影响

```
Q6_K 对函数调用的影响：可忽略（<1% 错误率上升）
Q4_K_M 的话会更明显：约 3-5% 错误率上升
```

所以 Agent 场景**别用低于 Q5 的量化**。

---

## 六、Hermes Agent + Qwen3.5-9B Q6_K

### 6.1 兼容性

```
通信协议:   chat_completions（OpenAI 兼容）
本地接入:   通过 LM Studio / Ollama 暴露的 /v1/chat/completions 端点
工具调用:   ✅ Qwen3.5 原生支持 OpenAI 格式 tool_calls
```

**理论上完全兼容。**

### 6.2 注意事项

Hermes Agent 有个 issue **#7628**："fallback tool call parsing for Qwen3-Coder and other models" —— 说明 Hermes 对部分 Qwen 变体的工具调用解析会出错，需要 fallback 解析器。

**实际情况：**
- Qwen3.5（标准版）：大概率没问题
- Qwen3.5-Coder（如果有）：可能需要 patch
- Qwen3.5-9B Thinking 模式：thinking 标签可能干扰 tool_call 解析

### 6.3 实际配置示例

```yaml
# Hermes 配置（伪代码）
provider: openai_compatible
base_url: http://localhost:11434/v1   # Ollama
# 或 http://localhost:1234/v1         # LM Studio
model: qwen3.5:9b-q6_K
temperature: 0.3                       # 低温度提升 tool 调用稳定性
thinking_mode: false                   # Hermes 期间关掉 thinking
```

### 6.4 预期表现

- ✅ 简单工具调用（搜索、文件读写）：稳定
- ✅ 单轮 1-3 个工具：稳定
- ⚠️ 复杂多步规划（10+ 工具链）：可能卡顿/出错
- ❌ 让它当"自主长跑 Agent"（几小时无人值守）：不靠谱

### 6.5 评分

```
Hermes Agent + Qwen3.5-9B Q6_K：⭐⭐⭐⭐ (4/5)
  适合：开发测试、个人小工具、原型验证
  不适合：生产级长链 Agent
```

---

## 七、OpenClaw + Qwen3.5-9B Q6_K

### 7.1 兼容性

OpenClaw 文档**明确支持 Ollama 本地模型**：

```yaml
# OpenClaw 配置（伪代码）
gateway:
  providers:
    ollama:
      base_url: http://localhost:11434
      models:
        - qwen3.5:9b-q6_K
```

### 7.2 但要诚实的事

OpenClaw 是**面向 Claude / GPT / Gemini 这类前沿模型设计的**。文档支持本地模型主要是为了"隐私场景"和"成本敏感场景"的兜底方案。

**用 9B 跑 OpenClaw 会遇到的问题：**

| 痛点 | 原因 |
|------|------|
| **复杂规划失败率高** | OpenClaw 的"Goal → Plan → Act → Observe → Evaluate → Repeat"循环对推理深度要求高 |
| **Skills 调用偶尔出错** | Skills 系统的 schema 复杂，9B 偶尔生成不合规 JSON |
| **持久记忆理解不深** | SOUL.md / MEMORY.md 上下文很长，9B 处理时容易"忘细节" |
| **多轮自主任务掉链** | 几十轮迭代后，错误累积明显 |

### 7.3 适合 vs 不适合

```
✅ 适合的任务（"窄而浅"）：
   - 单一 Skill 调用：摘要 / 翻译 / 格式化
   - 简单文件操作：读写本地文档
   - 短链流程：3-5 步以内
   - 原型测试

❌ 不适合的任务（"宽而深"）：
   - 自主代码重构
   - 多 Agent 协作
   - 长时无人值守任务
   - 跨多个 Skill 的复杂工作流
```

### 7.4 进阶玩法：混合调度（强推）

```yaml
gateway:
  routing_strategy: hybrid
  rules:
    - simple_tasks → ollama/qwen3.5-9b      # 本地、免费、快
    - complex_planning → claude_sonnet      # 云端、贵、可靠
    - code_generation → claude_opus         # 关键时刻派大模型
```

OpenClaw 的 Gateway 天生支持多 Provider 路由 —— **用 Qwen3.5-9B 干 80% 简单活，用 Claude / GPT 干 20% 关键活**，性价比最高。

### 7.5 评分

```
OpenClaw + Qwen3.5-9B Q6_K（纯本地）: ⭐⭐⭐ (3/5)
  适合：测试、学习、隐私场景
  不适合：生产 Agent

OpenClaw + 混合调度（9B 兜底 + 云端关键）：⭐⭐⭐⭐⭐ (5/5)
  适合：性价比最高的真实使用方式
```

---

## 八、Hermes 和 OpenClaw 横向对比

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|---------|
| **本地模型友好度** | ✅ 协议标准化好 | ✅ 但更偏向云端 |
| **对模型能力的要求** | 中等（工具优先） | **较高**（自主规划） |
| **生态丰富度** | 68 个内置工具 | 40 万+ Skills 市场 |
| **学习曲线** | 低 | 中等 |
| **Qwen3.5-9B 跑得动?** | ✅ 良好 | ⚠️ 简单任务 OK |
| **社区** | Nous 社区 | ClawHub 大社区 |
| **隐私** | ✅ 完全本地可行 | ✅ 完全本地可行 |
| **混合云本地** | 一般 | **优秀**（Gateway 设计） |

### 推荐场景

```
🎯 想要"开箱即用、本地稳定"
   → Hermes Agent + Qwen3.5-9B

🎯 想要"未来可拓展、混合调度"
   → OpenClaw + Qwen3.5-9B（兜底）+ Claude / GPT API（关键时）

🎯 想要"快速验证 Agent 概念"
   → 都行，Hermes 上手稍快
```

---

## 九、其他主流 Agent 对 Qwen3.5 的适配

| Agent 框架 | Qwen3.5-9B 适配 |
|-----------|----------------|
| **Cline**（VSCode 插件）| ✅ Ollama 直连 |
| **Continue** | ✅ |
| **Aider** | ✅ |
| **OpenHands**（前 OpenDevin）| ⚠️ 9B 偏弱 |
| **AutoGen** | ✅ |
| **CrewAI** | ✅ |
| **LangGraph** | ✅ |
| **MetaGPT** | ⚠️ 9B 偏弱 |

**编程类 Agent**（Cline、Continue、Aider）对 9B 友好；**复杂多 Agent 协作**（OpenHands、MetaGPT）建议用更大模型。

---

## 十、提升 9B 模型 Agent 表现的 5 个实操技巧

不管选 Hermes 还是 OpenClaw，这几招能让 9B 表现更好：

### 1️⃣ 关闭 Thinking 模式做工具调用

```
Thinking 模式会输出 <think>...</think> 标签
某些 Agent 框架的 tool_call 解析器会被这些标签干扰
解决：在 system prompt 中加 "/no_think" 或 API 中关闭
```

### 2️⃣ 降低 temperature

```
温度 0.7 → 工具调用错误率高
温度 0.2-0.3 → 显著稳定
但温度太低也会"死板"，建议 0.3 起步
```

### 3️⃣ 严格的 Tool Schema

```
JSON Schema 写得越严格越好
required 字段都标清楚
description 写明确
9B 模型靠这些"约束"减少幻觉
```

### 4️⃣ 限制工具数量

```
9B 同时面对 50 个工具会犯傻
简化到 5-10 个核心工具
或者用"分层 Agent"：先选工具组再用工具
```

### 5️⃣ 给"足够上下文"

```
不要省 token，把任务背景写清楚
9B 比 70B 更需要明确指令
"你是一个 X，目标是 Y，约束是 Z" 这种结构化 prompt 帮助巨大
```

---

## 十一、实战部署建议（4070 12GB + 32GB RAM）

### 推荐组合：单模型方案

如果需求是 **"日常 AI 助手 + 看图听音 + 语音对话"**：

```
单模型方案（最省）：
└─ Qwen3.5-9B Q6_K  ← 一张 4070 搞定 90% 需求

如果还想要图像生成：
└─ Qwen3.5-9B + Flux.1 dev (按需调用)
   总显存约 24-32 GB，需要切换加载或上 4090

如果想要视频生成：
└─ 加 Wan 2.2 14B（按需切换加载）
```

### 部署工具

| 工具 | 适合 |
|------|------|
| **LM Studio** | 图形化，对多模态支持好（首推） |
| **llama-server** | 性能最强（极客党） |
| **vLLM** | 高性能推理，主流 Omni 都支持（多并发场景） |
| **Ollama** | 部分支持 Omni（Qwen2.5-VL 等已上） |
| **官方 Demo（HF Transformers）** | 最稳，跟着 Hugging Face 仓库 README 走 |

---

## 十二、一个值得期待的方向（2026-2027）

2026 年大家盯着这几个**真·Omni**进展：

- **Qwen3-Omni / Qwen3.5-Omni**（阿里）
- **DeepSeek 多模态后续**（Janus 系列）
- **MiMo-V2.5**（小米的原生 Omni MoE，但 310B 太大本地跑不动）
- **开源版 GPT-4o** —— 至今没出现，但很多团队在追

**如果开源界出了一个"7B 能生图能生视频还能对话"的模型，那就是 ImageNet 时刻。** 目前还没到。

---

## 十三、一句话总结

> **现阶段最务实的本地"全能助手"方案就是 Qwen3.5-9B-Instruct Q6_K —— 一张 4070，听说看一体，少装四五个模型。但要图像 / 视频生成，还得外挂 Flux + Wan，没办法绕开。**
>
> **Hermes Agent + Qwen3.5-9B Q6_K：理论完全兼容，实际跑简单 Agent 没问题，复杂多步任务力有未逮。**
>
> **OpenClaw + Qwen3.5-9B：能跑但不是最佳搭配，建议用 Gateway 做"本地兜底 + 云端关键"的混合调度。**
>
> **核心真相：9B 是"称职助手"，不是"自主员工"。Agent 框架是放大器——好模型放大它的好，差模型放大它的差。Qwen3.5-9B 在 9B 这个尺寸已经是顶级，但还是 9B。**
