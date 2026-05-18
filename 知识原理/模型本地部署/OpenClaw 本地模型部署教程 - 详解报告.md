**来源**: [freedidi.com - OpenClaw 本地模型最佳方案](https://www.freedidi.com/23445.html)
**作者**: 零度博客
**分析时间**: 2026年3月20日

---

## 📋 目录

1. [背景与问题分析](#背景与问题分析)
2. [前期准备详解](#前期准备详解)
3. [安装 vLLM](#安装-vllm)
4. [模型选择与下载](#模型选择与下载)
5. [启动 vLLM 服务](#启动-vllm-服务)
6. [测试与配置](#测试与配置)
7. [性能优化](#性能优化)
8. [常见问题与解决方案](#常见问题与解决方案)

---

## 背景与问题分析

### 为什么需要本地模型？

**核心问题**：OpenClaw 在使用远程模型时会出现以下问题：
- ❌ **卡顿问题**：远程模型响应慢，影响自动化任务执行效率
- ❌ **上下文限制**：远程服务的上下文长度有限制
- ❌ **连续运行不稳定**：多任务执行后上下文容易被耗尽

### 推荐技术方案对比

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **SGLang** | 远程集群/多 Agent | 高并发、适合分布式 | 部署复杂，不适合单卡 |
| **vLLM** (推荐) | 单卡本地部署 | 部署简单、推理速度快、支持大模型 | 单机限制 |
| **Ollama** | 入门试用 | 安装简单 | 推理慢、性能不如 vLLM |

**核心结论**：单卡本地部署首选 vLLM，推理速度远超 Ollama。

---

## 前期准备详解

### 步骤 1：安装 Windows Terminal（可选但推荐）

**目的**：提供更好的终端体验，支持多标签、快捷键

**具体操作**：
1. 访问 Microsoft Store
2. 下载安装 Windows Terminal 应用
3. 安装后可以更好地管理 WSL 和 PowerShell 会话

**为什么推荐**：
- 比 CMD 更现代的界面
- 支持 UTF-8 编码，避免中文乱码
- 支持多标签页，方便同时操作多个终端

---

### 步骤 2：安装 WSL2（Windows Subsystem for Linux）

**目的**：在 Windows 上运行 Linux 环境，这是运行 vLLM 的基础

**具体操作详解**：

#### 2.1 初始化 WSL
```bash
wsl --install
```
**这个命令做了什么**：
- 启用 Windows 的 Linux 子系统功能
- 安装默认的 Linux 发行版（通常是 Ubuntu）
- 可能会提示重启电脑

#### 2.2 安装 Ubuntu（如果上述命令未自动安装）
```bash
wsl --install -d Ubuntu
```
**参数说明**：
- `-d Ubuntu`：指定安装 Ubuntu 发行版

#### 2.3 验证 WSL2 版本
```bash
wsl --version
```
**检查内容**：
- 确保版本号中包含 `2`，例如 `WSL version: 2.0.0.0`
- WSL2 支持 GPU 直通，这对运行本地模型至关重要

**为什么必须是 WSL2**：
- WSL2 使用真实的 Linux 内核
- 支持 GPU 直通（DirectX 到 Vulkan 的转换）
- 性能远高于 WSL1

---

### 步骤 3：配置 WSL 的 CUDA 驱动支持

**目的**：让 WSL 中的 Linux 能够使用 Windows 的 NVIDIA GPU

#### 3.1 前置条件
- Windows 主机必须先安装 NVIDIA 驱动
- 显卡必须是 NVIDIA（支持 CUDA）

#### 3.2 验证 GPU 直通
进入 WSL Ubuntu 终端，执行：
```bash
nvidia-smi
```

**这个命令检查什么**：
- 是否能识别到 NVIDIA 显卡
- 显示显存大小、驱动版本、CUDA 版本
- 如果显示显卡信息表，说明 GPU 直通成功

**成功输出示例**：
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.104.05   Driver Version: 535.104.05   CUDA Version: 12.2   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0  On |                  N/A |
|  0%   40C    P8    15W / 450W |     1000MiB / 24576MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

**如果失败**：
- 检查 Windows NVIDIA 驱动是否正确安装
- 更新到最新版本的 NVIDIA 驱动
- 确认 Windows 版本支持 WSL GPU（需要 Windows 10 2004+ 或 Windows 11）

---

### 步骤 4：安装 Python 环境

**目的**：安装运行 vLLM 所需的 Python 环境和包管理工具

#### 4.1 更新系统包
```bash
sudo apt update && sudo apt upgrade -y
```
**这个命令做了什么**：
- `apt update`：更新软件包列表（从 Ubuntu 软件源获取最新信息）
- `apt upgrade -y`：升级所有已安装的软件包
- `-y`：自动确认所有提示，无需手动输入 Y

#### 4.2 安装 Python 和 pip
```bash
sudo apt install python3-pip python3-venv -y
```
**安装了什么**：
- `python3-pip`：Python 包管理器（用于安装 vLLM 等库）
- `python3-venv`：虚拟环境工具（用于创建独立的 Python 环境）

**为什么需要虚拟环境**：
- 避免包冲突（不同项目需要不同版本的包）
- 保持系统环境干净
- 方便管理依赖关系

#### 4.3 创建并激活虚拟环境
```bash
cd ~
python3 -m venv vllm-env
source vllm-env/bin/activate
```
**每个命令的作用**：

1. `cd ~`
   - 切换到用户主目录（例如 `/home/ubuntu`）

2. `python3 -m venv vllm-env`
   - 创建名为 `vllm-env` 的虚拟环境
   - 在 `~/vllm-env/` 目录下创建独立的 Python 环境
   - 包含自己的 Python 解释器和 site-packages

3. `source vllm-env/bin/activate`
   - 激活虚拟环境
   - 修改环境变量 PATH
   - 之后执行的 python 和 pip 命令都会使用虚拟环境中的版本

**激活后的变化**：
- 命令提示符前会出现 `(vllm-env)` 前缀
- `(vllm-env) username@hostname:~$`

**重要提示**：每次打开新终端都需要执行激活命令

---

## 安装 vLLM

### 步骤 5：安装 vLLM 库

**目的**：安装高性能的 LLM 推理引擎

#### 5.1 升级 pip
```bash
pip install --upgrade pip
```
**为什么升级**：
- 旧版本 pip 可能有 bug
- 新版本支持更好的依赖解析
- 确保后续安装的包兼容

#### 5.2 安装 vLLM
```bash
pip install vllm
```
**这个命令做了什么**：
- 从 PyPI（Python 包索引）下载 vLLM 及其依赖
- 自动安装：
  - PyTorch（深度学习框架）
  - transformers（Hugging Face 模型库）
  - cuda-toolkit（CUDA 工具包，如需要）
  - 其他依赖库

**安装时间**：
- 根据网络速度，可能需要 5-20 分钟
- vLLM 约 1-2GB，加上依赖可能需要 5-10GB 下载量

#### 5.3 验证安装
```bash
python -c "import vllm; print('vLLM installed')"
```
**验证内容**：
- 检查 vLLM 库是否能正常导入
- 如果没有报错且输出 `vLLM installed`，说明安装成功

**常见错误**：
- `ImportError`: 可能是依赖包缺失或版本冲突
- `CUDA out of memory`: 显存不足，与安装无关

---

## 模型选择与下载

### 步骤 6：选择合适的模型

**推荐模型**：Qwen2.5-14B-Instruct-AWQ

#### 为什么选择 Qwen2.5-14B-Instruct-AWQ？

| 特性 | 说明 |
|------|------|
| **中文能力强** | 通义千问系列，针对中文优化，理解准确 |
| **推理能力强** | 14B 参数，在多个基准测试中表现优异 |
| **Agent 能力好** | 支持工具调用（Tool Calling），适合 OpenClaw 自动化 |
| **AWQ 量化** | 使用 4-bit 量化，大幅降低显存占用 |
| **显存友好** | 24GB 显存可运行，性价比高 |

#### 显存需求对照表

| 模型 | 显存需求 | 适用显卡 | 性能 |
|------|---------|---------|------|
| Qwen2.5-14B-Instruct-AWQ | 24GB | RTX 4090, A5000 | ⭐⭐⭐⭐⭐ |
| Qwen2.5-7B-Instruct-AWQ | 12-16GB | RTX 4070, 4060Ti | ⭐⭐⭐⭐ |
| Qwen2.5-4B-Instruct-AWQ | 8GB | RTX 3060, 4060 | ⭐⭐⭐ |
| Qwen2.5-32B-Instruct-AWQ | 48GB | A6000, 双 4090 | ⭐⭐⭐⭐⭐ |

**注**：vLLM 启动时会自动从 Hugging Face 下载模型，无需手动下载。

#### 模型下载原理

当执行 vLLM 启动命令时：
1. vLLM 检查本地缓存 `~/.cache/huggingface/hub/`
2. 如果不存在，从 Hugging Face Hub 下载模型文件
3. 模型文件包括：
   - `config.json`：模型配置文件
   - `model.safetensors` 或 `.bin`：模型权重（多个文件）
   - `tokenizer.json` / `tokenizer.model`：分词器
   - `special_tokens_map.json`：特殊 token 映射
4. 下载完成后缓存，下次启动无需重新下载

**下载时间**（取决于网络）：
- 国内直连 HF：可能很慢或失败
- 建议使用镜像站（如 hf-mirror.com）或代理

---

## 启动 vLLM 服务

### 步骤 7：启动 vLLM 服务器

**目的**：启动一个兼容 OpenAI API 的本地推理服务

#### 7.1 启动命令详解

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-14B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.9 \
  --max-model-len 32768 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

#### 每个参数详解

| 参数 | 值 | 作用说明 |
|------|---|---------|
| `--model` | `Qwen/Qwen2.5-14B-Instruct-AWQ` | 指定模型名称（Hugging Face Hub 标识符） |
| `--quantization` | `awq_marlin` | 使用 AWQ 量化格式，Marlin 优化推理速度 |
| `--gpu-memory-utilization` | `0.9` | 使用 90% 的 GPU 显存（预留 10% 给其他操作） |
| `--max-model-len` | `32768` | 最大上下文长度（32K tokens） |
| `--enable-auto-tool-choice` | （无值） | 启用自动工具选择（Agent 自动调用工具） |
| `--tool-call-parser` | `hermes` | 使用 Hermes 解析器处理工具调用 |

#### 关键参数深入说明

##### 1. `--quantization awq_marlin`
**AWQ（Activation-aware Weight Quantization）**：
- 4-bit 量化技术
- 保持模型精度的同时减少显存占用

**Marlin**：
- 专为 AWQ 优化的推理内核
- 比标准 AWQ 快 2-3 倍
- RTX 40 系列显卡加速显著

**为什么不选其他量化**：
- `gptq`：速度慢，已淘汰
- `awq`：标准 AWQ，但比 Marlin 慢
- `none`：不量化，显存占用高（14B 模型需 28GB+）

##### 2. `--gpu-memory-utilization 0.9`
**为什么留 10% 缓冲**：
- KV Cache 动态增长需要空间
- 避免 CUDA OOM（显存溢出）错误
- 系统其他进程可能需要 GPU 资源

**调整建议**：
- 显存紧张：设置 `0.85` 或 `0.8`
- 显存充足：可设 `0.95`
- 出现 OOM 错误：降低该值

##### 3. `--max-model-len 32768`
**32K tokens 的含义**：
- 约等于 24,000-30,000 中文字符
- 相当于 100-150 页 A4 纸
- 足够处理长文档、长对话

**显存与上下文长度的关系**：
```
显存占用 = 模型权重 + KV Cache

KV Cache ∝ batch_size × max_len × hidden_size × num_layers
```

**调整建议**：
- 24GB 显卡：`32768`（32K）
- 16GB 显卡：`16384`（16K）
- 12GB 显卡：`8192`（8K）

##### 4. `--enable-auto-tool-choice`
**工具调用（Tool Calling）**：
- OpenClaw 的核心功能
- 让 AI 自动决策何时调用外部工具（如文件操作、网络请求）

**auto-tool-choice**：
- 不需强制提示"使用工具"
- AI 自动判断何时调用工具
- 提升智能化程度

##### 5. `--tool-call-parser hermes`
**Hermes 解析器**：
- 从 Llama-2-Chat 模型演化而来
- 支持复杂工具调用场景
- 兼容 OpenAI 工具调用格式

#### 7.2 启动过程详解

**阶段 1：加载模型**
```
INFO:     Started server process
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

**阶段 2：初始化 GPU**
```
INFO:     Initializing an LLM engine with config: ...
INFO:     Loading model weights took 3.2 seconds
INFO:     CUDA Graphs: 0 blocks captured
```

**阶段 3：准备服务**
```
INFO:     Model Qwen2.5-14B-Instruct-AWQ is ready!
```

**启动成功标志**：
- 看到 `Model ... is ready!` 消息
- 终端没有报错（ERROR 或 CUDA OOM）
- 可以访问 `http://127.0.0.1:8000`

#### 7.3 常见启动问题

| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| `CUDA out of memory` | 显存不足 | 降低 `--gpu-memory-utilization` 或 `--max-model-len` |
| `ModuleNotFoundError` | 依赖缺失 | 重新安装 `pip install vllm --upgrade` |
| `Connection refused` | 端口被占用 | 添加 `--port 8001` 指定其他端口 |
| `Download timeout` | 模型下载失败 | 使用 HF 镜像站或代理 |

---

## 测试与配置

### 步骤 8：测试 vLLM 服务

**目的**：确认 vLLM 服务正常运行并可响应 API 请求

#### 8.1 测试 API 连接

在 Windows PowerShell 中执行：

```bash
curl http://127.0.0.1:8000/v1/models
```

**这个命令做了什么**：
- 发送 HTTP GET 请求到 vLLM 服务器
- 获取可用模型列表
- 使用 OpenAI 兼容的 `/v1/models` 端点

**成功响应示例**：
```json
{
  "object": "list",
  "data": [
    {
      "id": "Qwen/Qwen2.5-14B-Instruct-AWQ",
      "object": "model",
      "created": 1710995200,
      "owned_by": "Qwen"
    }
  ]
}
```

**响应字段说明**：
- `id`：模型唯一标识符（用于 API 调用时指定模型）
- `object`：资源类型
- `created`：模型创建时间戳
- `owned_by`：模型提供方

**如果失败**：
- `curl: (7) Failed to connect`：vLLM 服务未启动
- `curl: (52) Empty reply from server`：vLLM 正在初始化
- 检查 WSL 终端是否还有错误信息

---

### 步骤 9：安装 OpenClaw

**目的**：安装 OpenClaw 客户端，用于调用本地模型

#### 9.1 在 WSL 中安装 Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

**命令详解**：

1. `curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -`
   - `curl`：下载工具
   - `-fsSL`：静默模式，跟随重定向
   - `|`：管道，将下载的脚本传递给 bash
   - `sudo -E bash -`：以 root 权限执行脚本
   - `setup_22.x`：安装 Node.js 22.x 版本（LTS）

2. `sudo apt install -y nodejs`
   - 安装 Node.js 和 npm（包管理器）
   - `-y`：自动确认

**为什么需要 Node.js**：
- OpenClaw 是基于 Node.js 的命令行工具
- 需要 npm 来安装和管理依赖

#### 9.2 安装 OpenClaw

```bash
sudo npm install -g openclaw@latest
```

**参数说明**：
- `-g`：全局安装，可在任何目录使用 `openclaw` 命令
- `openclaw@latest`：安装最新版本的 OpenClaw
- `sudo`：需要 root 权限写入全局目录

**安装位置**：
- 命令：`/usr/local/bin/openclaw`
- 包：`/usr/local/lib/node_modules/openclaw/`

**验证安装**：
```bash
openclaw --version
```

---

### 步骤 10：配置 OpenClaw 使用本地模型

**目的**：让 OpenClaw 连接到本地 vLLM 服务

#### 10.1 进入配置向导

```bash
openclaw onboard
```

**这个命令做了什么**：
- 启动交互式配置向导
- 添加新的模型提供商
- 设置 API 端点和认证信息

#### 10.2 配置步骤详解

**步骤 1：选择模型提供商**
```
? Select model provider: (Use arrow keys)
❯ Custom
  OpenAI
  Azure OpenAI
  Anthropic
  ...
```
- 选择 `Custom`（自定义）

**步骤 2：输入 Base URL**
```
? Enter base URL: http://127.0.0.1:8000/v1
```
- 这是 vLLM 服务的地址
- `/v1` 后缀是 OpenAI API 兼容路径

**步骤 3：输入 API Key**
```
? Enter API key: 123456
```
- 本地服务不需要真实的 API Key
- 输入任意值即可（如 `123456`、`sk-xxxxx`）

**步骤 4：输入模型名称**
```
? Enter model name: Qwen2.5-14B-Instruct-AWQ
```
- 必须与 vLLM 启动时指定的 `--model` 参数一致
- 区分大小写

**步骤 5：保存配置**
- 按提示确认保存
- 配置文件保存在 `~/.openclaw/config.json`

#### 10.3 配置文件示例

```json
{
  "models": [
    {
      "provider": "custom",
      "baseUrl": "http://127.0.0.1:8000/v1",
      "apiKey": "123456",
      "name": "Qwen2.5-14B-Instruct-AWQ",
      "enabled": true
    }
  ]
}
```

---

## 性能优化

### 步骤 11：OpenClaw 参数调优

**目的**：在模型性能和资源占用之间找到平衡

#### 推荐参数配置

| 参数 | 推荐值 | 作用说明 |
|------|--------|---------|
| **Context length** | 6000-8000 | 单次对话上下文长度（tokens） |
| **Temperature** | 0.7 | 生成随机性（0-1，越高越随机） |
| **Max tokens** | 2048 | 单次回复最大长度 |

#### 参数详解

##### Context length（上下文长度）

**值域**：1000-32000

**设置原则**：
- **6000-8000**：适合大多数自动化任务
- **10000+**：长文档分析、代码审查
- **3000-5000**：简单对话、快速响应

**权衡**：
- 越大 → 支持长对话，但显存占用高
- 越小 → 响应快，但可能丢失上下文

##### Temperature（温度参数）

**值域**：0.0-2.0

**实际效果**：
```
Temperature = 0.0: 完全确定性，每次回复相同
Temperature = 0.7: 有一定创造性，但稳定（推荐）
Temperature = 1.5: 非常创造性，可能不稳定
```

**应用场景**：
- **0.2-0.4**：代码生成、数学计算（需要准确）
- **0.6-0.8**：一般对话、任务执行（推荐）
- **1.0+**：创意写作、头脑风暴

##### Max tokens（最大生成长度）

**值域**：512-4096

**设置原则**：
- **2048**：适合大多数任务（推荐）
- **1024**：快速响应、短回复
- **4096**：长文本生成

**注意**：实际生成可能少于该值，取决于内容完整性

---

### 步骤 12：vLLM 启动参数优化

**目的**：最大化 vLLM 性能和稳定性

#### 高性能启动参数（RTX 4090 推荐）

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-14B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.9 \
  --max-model-len 32768 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --tensor-parallel-size 1 \
  --dtype float16 \
  --enforce-eager
```

#### 新增参数说明

| 参数 | 值 | 作用 |
|------|---|------|
| `--tensor-parallel-size` | `1` | 张量并行度（单卡设为 1） |
| `--dtype` | `float16` | 使用 FP16 精度（平衡速度和精度） |
| `--enforce-eager` | （无值） | 强制使用 eager 模式（兼容性更好） |

#### 不同显卡配置建议

**RTX 4090 / A5000 (24GB)**
```bash
--gpu-memory-utilization 0.9
--max-model-len 32768
--quantization awq_marlin
```

**RTX 4070 / 3060 (12GB)**
```bash
--gpu-memory-utilization 0.85
--max-model-len 16384
--quantization awq_marlin
--dtype float16
```

**RTX 4060 / 3060Ti (8GB)**
```bash
--gpu-memory-utilization 0.8
--max-model-len 8192
--quantization awq_marlin
--dtype float16
```

---

### 步骤 13：解决长对话卡顿

**问题**：对话过长时，性能显著下降

#### 原因分析

长对话的 KV Cache 累积导致：
- 显存占用增加
- 推理速度变慢
- 可能超出 `max-model-len` 限制

#### 解决方案：System Prompt 添加摘要指令

在 OpenClaw 的 System Prompt 中添加：

> When the conversation becomes long, summarize previous messages into a short memory. Keep the memory under 200 tokens.

**这个指令的作用**：
- 监控对话长度
- 自动压缩历史消息
- 保持上下文在 200 tokens 内

#### 效果对比

| 场景 | 无摘要 | 有摘要 |
|------|--------|--------|
| 10 轮对话后 | 速度下降 40% | 速度基本不变 |
| 显存占用 | 累积 5000+ tokens | 稳定在 200 tokens |
| 响应延迟 | 增加 2-3 秒 | 保持稳定 |

---

## 性能参考数据（RTX 4090）

### 实测性能指标

| 指标 | 数值 | 说明 |
|------|------|------|
| **Token 生成速度** | 90-130 tokens/s | 每秒生成的 token 数量 |
| **首 Token 延迟** | 0.4-0.8 秒 | 从请求到第一个 token 的延迟 |
| **最大上下文** | 32K tokens | `max-model-len` 设置值 |
| **实际建议上下文** | 8K-16K tokens | 平衡性能和稳定性的推荐值 |
| **显存占用** | 10-12GB | 模型加载后 + 空闲状态 |

### 性能对比

| 推理引擎 | 速度（tokens/s） | 延迟（秒） |
|---------|-----------------|-----------|
| **vLLM (AWQ Marlin)** | 90-130 | 0.4-0.8 |
| Ollama (Qwen 14B) | 40-60 | 1.5-2.5 |
| Hugging Face Transformers | 20-30 | 3.0-5.0 |

### 资源占用监控

使用 `nvidia-smi` 实时监控：

```bash
watch -n 1 nvidia-smi
```

**正常状态示例**：
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.104.05   Driver Version: 535.104.05   CUDA Version: 12.2   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================+
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0  On |                  N/A |
| 40%   55C    P2   350W / 450W |    11500MiB / 24576MiB |     95%      Default |
+-------------------------------+----------------------+----------------------+
```

**关注指标**：
- **GPU-Util**: 应在 70-95% 之间（表示充分利用）
- **Memory-Usage**: 应在 10-12GB 之间（稳定状态）
- **Temp**: 应在 50-75°C 之间（正常范围）

---

## 常见问题与解决方案

### 问题 1：vLLM 启动时报 "CUDA out of memory"

**错误信息**：
```
torch.cuda.OutOfMemoryError: CUDA out of memory. Tried to allocate 512.00 MiB
```

**原因**：
- 显存不足
- `--gpu-memory-utilization` 或 `--max-model-len` 设置过高

**解决方案**：
1. 降低 GPU 内存利用率：
   ```bash
   --gpu-memory-utilization 0.8  # 从 0.9 降到 0.8
   ```

2. 减少最大上下文长度：
   ```bash
   --max-model-len 16384  # 从 32768 降到 16384
   ```

3. 换用更小的模型（如 7B 或 4B）

---

### 问题 2：模型下载很慢或失败

**错误信息**：
```
ConnectionError: HTTPSConnectionPool(host='huggingface.co', port=443): Max retries exceeded
```

**原因**：
- 网络问题
- Hugging Face 访问受限

**解决方案**：

**方法 1：使用 HF 镜像站**
```bash
export HF_ENDPOINT=https://hf-mirror.com
# 然后再启动 vLLM
```

**方法 2：手动下载模型**
1. 使用 Git LFS 克隆：
   ```bash
   git lfs install
   git clone https://hf-mirror.com/Qwen/Qwen2.5-14B-Instruct-AWQ
   ```

2. 启动时指定本地路径：
   ```bash
   --model /path/to/Qwen2.5-14B-Instruct-AWQ
   ```

**方法 3：使用代理**
```bash
export https_proxy=http://127.0.0.1:7890
export http_proxy=http://127.0.0.1:7890
```

---

### 问题 3：OpenClaw 无法连接到本地模型

**错误信息**：
```
Error: Failed to connect to http://127.0.0.1:8000/v1
```

**原因**：
- vLLM 服务未启动
- 端口被占用
- 防火墙阻止

**解决方案**：

1. **检查 vLLM 服务状态**：
   在 WSL 终端确认 vLLM 是否还在运行

2. **检查端口是否被占用**：
   ```bash
   netstat -tlnp | grep 8000
   ```

3. **重启 vLLM 服务**：
   - 停止当前服务（Ctrl+C）
   - 重新执行启动命令

4. **检查防火墙**：
   - Windows 防火墙：允许 WSL 端口
   - WSL 防火墙：`sudo ufw allow 8000`

---

### 问题 4：长对话后性能下降

**现象**：
- 初始速度：100 tokens/s
- 10 轮后：下降到 30 tokens/s

**原因**：
- KV Cache 累积过多
- 历史上下文未清理

**解决方案**：

1. **添加摘要指令**（参考步骤 13）
2. **定期重启对话**
3. **调整上下文长度**（降低 `Context length`）

---

### 问题 5：工具调用不生效

**现象**：
- AI 不调用工具，直接返回文本

**原因**：
- 工具调用解析器配置错误
- 模型不支持工具调用

**解决方案**：

1. **确认启动参数**：
   ```bash
   --enable-auto-tool-choice \
   --tool-call-parser hermes
   ```

2. **检查 System Prompt**：
   - 确保提示中包含工具调用说明

3. **更换模型**：
   - 确保模型支持工具调用（如 Qwen2.5-Instruct）

---

## 总结

### 完整部署流程图

```
┌─────────────────────────────────────────────────────┐
│ 1. 安装 Windows Terminal (可选)                      │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│ 2. 安装 WSL2 + Ubuntu                                │
│    - wsl --install                                   │
│    - wsl --install -d Ubuntu                         │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│ 3. 配置 GPU 直通                                      │
│    - 安装 Windows NVIDIA 驱动                        │
│    - 在 WSL 中运行 nvidia-smi 验证                   │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│ 4. 安装 Python 环境                                   │
│    - apt install python3-pip python3-venv            │
│    - python3 -m venv vllm-env                        │
│    - source vllm-env/bin/activate                    │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│ 5. 安装 vLLM                                         │
│    - pip install vllm                               │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│ 6. 启动 vLLM 服务                                     │
│    - python -m vllm.entrypoints.openai.api_server   │
│    - 配置量化、显存、上下文等参数                     │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│ 7. 安装 OpenClaw                                     │
│    - npm install nodejs                              │
│    - npm install -g openclaw@latest                  │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│ 8. 配置 OpenClaw 连接本地模型                         │
│    - openclaw onboard                               │
│    - Base URL: http://127.0.0.1:8000/v1              │
└──────────────────┬──────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────┐
│ 9. 性能优化                                          │
│    - 调整 Context length / Temperature               │
│    - 添加长对话摘要指令                               │
└─────────────────────────────────────────────────────┘
```

### 核心要点回顾

1. **vLLM > Ollama**：性能提升 2-3 倍
2. **AWQ Marlin 量化**：大幅降低显存占用，保持高速度
3. **GPU 直通是关键**：必须使用 WSL2
4. **参数调优很重要**：根据显存调整 `gpu-memory-utilization` 和 `max-model-len`
5. **工具调用是核心竞争力**：确保 `--enable-auto-tool-choice` 和 `--tool-call-parser hermes`

### 适用场景

✅ **适合**：
- 需要高性能推理的自动化任务
- 长对话、长文档分析
- 需要频繁调用工具的 Agent
- 对隐私有要求的本地部署

❌ **不适合**：
- 显存不足 8GB
- 不熟悉 Linux 命令行
- 需要多机分布式部署

### 后续优化建议

1. **监控日志**：定期检查 vLLM 日志，发现性能瓶颈
2. **A/B 测试**：对比不同参数配置的效果
3. **模型更新**：关注 Qwen 系列新版本
4. **自动化部署**：编写脚本一键启动 vLLM 服务

---

## 附录：快速启动脚本

### vLLM 启动脚本（start_vllm.sh）

```bash
#!/bin/bash

# 激活虚拟环境
source ~/vllm-env/bin/activate

# 设置环境变量（使用 HF 镜像站）
export HF_ENDPOINT=https://hf-mirror.com

# 启动 vLLM
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-14B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.9 \
  --max-model-len 32768 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --tensor-parallel-size 1 \
  --dtype float16
```

**使用方法**：
```bash
chmod +x start_vllm.sh
./start_vllm.sh
```

### 测试脚本（test_vllm.py）

```python
import requests
import json

url = "http://127.0.0.1:8000/v1/chat/completions"
headers = {"Content-Type": "application/json"}

payload = {
    "model": "Qwen/Qwen2.5-14B-Instruct-AWQ",
    "messages": [
        {"role": "user", "content": "你好，请用一句话介绍 vLLM。"}
    ],
    "temperature": 0.7,
    "max_tokens": 100
}

response = requests.post(url, headers=headers, json=payload)
print(json.dumps(response.json(), indent=2, ensure_ascii=False))
```

---

**报告完成时间**：2026年3月20日
**文档版本**：1.0
