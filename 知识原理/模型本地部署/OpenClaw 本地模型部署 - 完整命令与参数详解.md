**文档版本**: 2.0
**创建时间**: 2026年3月20日
**适用场景**: 本地 OpenClaw + vLLM 部署

---

## 📋 目录

1. [硬件配置检测](#硬件配置检测)
2. [模型选择策略](#模型选择策略)
3. [环境准备 - 每条命令详解](#环境准备---每条命令详解)
4. [vLLM 安装 - 每条命令详解](#vllm-安装---每条命令详解)
5. [模型启动 - 每条参数详解](#模型启动---每条参数详解)
6. [OpenClaw 配置 - 每条命令详解](#openclaw-配置---每条命令详解)
7. [参数调优 - 详细配置表](#参数调优---详细配置表)
8. [性能监控命令](#性能监控命令)
9. [故障排查 - 诊断命令](#故障排查---诊断命令)

---

## 硬件配置检测

### 步骤 1：检查系统信息

#### 1.1 检查 Windows 版本（PowerShell）

```powershell
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `systeminfo` | 显示系统详细信息 |
| `|` | 管道，将前面命令的输出传递给下一个命令 |
| `findstr` | 文本搜索工具（类似 grep） |
| `/B` | 从行首开始匹配 |
| `/C:"OS Name"` | 搜索包含 "OS Name" 的行 |
| `/C:"OS Version"` | 搜索包含 "OS Version" 的行 |

**为什么检查**：
- Windows 10 版本 2004+ 或 Windows 11 才支持 WSL GPU 直通
- 确认系统兼容性，避免后续部署失败

**期望输出**：
```
OS Name:                   Microsoft Windows 11 Pro
OS Version:                10.0.22621 N/A Build 22621
```

---

#### 1.2 检查 NVIDIA 显卡（PowerShell）

```powershell
wmic path win32_VideoController get name,AdapterRAM
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `wmic` | Windows 管理工具命令行 |
| `path win32_VideoController` | 指定查询显卡控制器信息 |
| `get` | 获取指定属性 |
| `name` | 显卡名称 |
| `AdapterRAM` | 显存大小（字节） |

**为什么检查**：
- 确认显卡型号（必须支持 CUDA）
- 确认显存大小（决定能运行多大模型）

**期望输出**：
```
Name                           AdapterRAM
NVIDIA GeForce RTX 4090        25769803776
```

**显存换算**：
```
25769803776 bytes = 25769803776 / 1024 / 1024 / 1024 = 24 GB
```

---

#### 1.3 检查 NVIDIA 驱动版本（PowerShell）

```powershell
nvidia-smi
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `nvidia-smi` | NVIDIA 系统管理界面（System Management Interface） |
| 无参数 | 显示默认视图（GPU 信息表） |

**为什么检查**：
- 确认驱动已安装
- 检查 CUDA 版本（至少 11.8+）
- 确认 GPU 可用状态

**期望输出**：
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.104.05   Driver Version: 535.104.05   CUDA Version: 12.2   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================+
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0  On |                  N/A |
|  0%   38C    P0     8W / 450W |     500MiB / 24576MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

**关键字段**：
- `Driver Version`: 535.104.05（驱动版本）
- `CUDA Version`: 12.2（支持的 CUDA 版本）
- `Memory-Usage`: 500MiB / 24576MiB（显存使用量 / 总显存）
- `GPU-Util`: 0%（当前 GPU 利用率）

---

## 模型选择策略

### 选择矩阵 - 基于硬件配置

#### 显存 vs 模型选择对照表

| 显存       | 推荐模型                     | 量化方式      | 参数量 | 预估性能        | 使用场景       |
| -------- | ------------------------ | --------- | --- | ----------- | ---------- |
| **8GB**  | Qwen2.5-4B-Instruct-AWQ  | AWQ 4-bit | 4B  | 150-200 t/s | 轻量对话、快速响应  |
| **12GB** | Qwen2.5-7B-Instruct-AWQ  | AWQ 4-bit | 7B  | 120-170 t/s | 日常任务、文档分析  |
| **16GB** | Qwen2.5-7B-Instruct-AWQ  | AWQ 4-bit | 7B  | 120-170 t/s | 复杂任务、长上下文  |
| **24GB** | Qwen2.5-14B-Instruct-AWQ | AWQ 4-bit | 14B | 90-130 t/s  | 专业级任务、推理密集 |
| **48GB** | Qwen2.5-32B-Instruct-AWQ | AWQ 4-bit | 32B | 60-90 t/s   | 企业级、研究任务   |

**注**：
- `t/s` = tokens per second（每秒生成速度）
- 性能数据基于 RTX 4090 实测，其他显卡会有所差异

---

#### 显卡型号对照表

| 显卡型号 | 显存 | 推荐模型 | 备注 |
|---------|------|---------|------|
| **RTX 4090** | 24GB | Qwen2.5-14B-Instruct-AWQ | 最佳性能选择 |
| **RTX 4080** | 16GB | Qwen2.5-14B-Instruct-AWQ | 需降低 max-model-len 到 16K |
| **RTX 4070 Ti** | 12GB | Qwen2.5-7B-Instruct-AWQ | 性价比高 |
| **RTX 4070** | 12GB | Qwen2.5-7B-Instruct-AWQ | 标准配置 |
| **RTX 4060 Ti** | 16GB | Qwen2.5-7B-Instruct-AWQ | 平衡选择 |
| **RTX 4060** | 8GB | Qwen2.5-4B-Instruct-AWQ | 入门级 |
| **RTX 3060** | 12GB | Qwen2.5-7B-Instruct-AWQ | 上一代性价比 |
| **A5000** | 24GB | Qwen2.5-14B-Instruct-AWQ | 专业显卡 |
| **A6000** | 48GB | Qwen2.5-32B-Instruct-AWQ | 企业级 |

---

### 模型选择决策树

```
开始
  ↓
显存大小？
  ├─ 8GB  → 选择 Qwen2.5-4B-Instruct-AWQ
  ├─ 12GB → 选择 Qwen2.5-7B-Instruct-AWQ（标准配置）
  ├─ 16GB → 选择 Qwen2.5-7B-Instruct-AWQ（提升上下文到 16K）
  ├─ 24GB → 选择 Qwen2.5-14B-Instruct-AWQ（推荐）
  └─ 48GB+ → 选择 Qwen2.5-32B-Instruct-AWQ（专业级）
           ↓
任务类型？
  ├─ 快速对话 → 降低 temperature，提高 speed
  ├─ 推理密集 → 提高上下文长度
  └─ 创意写作 → 提高 temperature
```

---

### 不同场景的参数推荐

#### 场景 1：日常对话（RTX 4070, 12GB）

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-7B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.85 \
  --max-model-len 16384 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

**参数说明**：
- `--gpu-memory-utilization 0.85`：留 15% 缓冲，避免 OOM
- `--max-model-len 16384`：16K tokens，足够日常对话

**OpenClaw 配置**：
- Context length: 6000
- Temperature: 0.7
- Max tokens: 1024

---

#### 场景 2：长文档分析（RTX 4080, 16GB）

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-14B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.85 \
  --max-model-len 32768 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

**参数说明**：
- `--max-model-len 32768`：32K tokens，支持长文档
- 降低 `gpu-memory-utilization` 以适应 16GB 显存

**OpenClaw 配置**：
- Context length: 10000
- Temperature: 0.5（更精确）
- Max tokens: 2048

---

#### 场景 3：快速响应（RTX 4060, 8GB）

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-4B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.9 \
  --max-model-len 8192 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

**参数说明**：
- 使用 4B 小模型，速度最快
- `--max-model-len 8192`：8K tokens，小显存友好

**OpenClaw 配置**：
- Context length: 3000
- Temperature: 0.7
- Max tokens: 512

---

## 环境准备 - 每条命令详解

### 步骤 2：安装 WSL2

#### 2.1 启用 WSL 功能（PowerShell 管理员）

```powershell
wsl --install
```

**命令详解**：
| 参数 | 作用 |
|------|------|
| `wsl` | WSL 命令行工具 |
| `--install` | 安装 WSL 及默认 Linux 发行版 |

**执行过程**：
1. 启用 WSL 功能
2. 启用虚拟机平台
3. 下载并安装 Linux 内核更新包
4. 设置 WSL 2 为默认版本
5. 下载并安装 Ubuntu

**为什么需要管理员权限**：
- 启用 Windows 功能需要系统权限
- 修改系统配置需要提升权限

**执行后**：
- 系统提示重启电脑
- 重启后自动安装 Ubuntu

---

#### 2.2 手动安装 Ubuntu（如自动安装失败）

```powershell
wsl --install -d Ubuntu
```

**命令详解**：
| 参数 | 作用 |
|------|------|
| `-d` | 指定发行版（distribution） |
| `Ubuntu` | 发行版名称（最新版 Ubuntu LTS） |

**可用发行版**：
- `Ubuntu-22.04`（LTS，推荐）
- `Ubuntu-20.04`（旧版 LTS）
- `Ubuntu-24.04`（最新 LTS）

**查看所有可用发行版**：
```powershell
wsl --list --online
```

---

#### 2.3 验证 WSL 版本

```powershell
wsl --version
```

**命令详解**：
| 参数 | 作用 |
|------|------|
| `--version` | 显示 WSL 版本信息 |

**期望输出**：
```
WSL 版本: 2.0.9.0
内核版本: 5.15.90.1
WSLg 版本: 1.0.51
...
```

**关键检查**：
- 确认版本号包含 `2.x.x.x`（WSL2）
- WSL1 不支持 GPU 直通

---

### 步骤 3：配置 WSL GPU 直通

#### 3.1 进入 WSL Ubuntu 终端

**方法 1**：在开始菜单搜索 "Ubuntu"
**方法 2**：在 PowerShell 执行 `wsl`

#### 3.2 验证 NVIDIA 驱动（WSL 终端）

```bash
nvidia-smi
```

**命令详解**：
- 与 Windows 中相同，但在 WSL 中执行

**期望输出**：
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.104.05   Driver Version: 535.104.05   CUDA Version: 12.2   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================+
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0  On |                  N/A |
|  0%   38C    P0     8W / 450W |     500MiB / 24576MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

**关键检查**：
- `Bus-Id`: 不为空（如 `00000000:01:00.0`）
- `Memory-Usage`: 显示总显存（如 `24576MiB`）

**如果失败（命令不存在）**：
- Windows NVIDIA 驱动未安装或版本过低
- 下载最新驱动：https://www.nvidia.com/Download/index.aspx

---

### 步骤 4：安装 Python 环境

#### 4.1 更新系统包

```bash
sudo apt update && sudo apt upgrade -y
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `sudo` | 以 root 权限执行 |
| `apt` | 高级包管理工具（Advanced Package Tool） |
| `update` | 更新软件包列表（从 Ubuntu 软件源） |
| `&&` | 逻辑与，前一个命令成功后执行下一个 |
| `upgrade` | 升级已安装的软件包 |
| `-y` | 自动确认所有提示（无需手动输入 Y） |

**为什么需要 update**：
- 确保 apt 使用最新的软件包索引
- 避免下载旧版本或不兼容的包

**为什么需要 upgrade**：
- 修复安全漏洞
- 提供新功能
- 确保依赖包的兼容性

**执行时间**：5-10 分钟（取决于网络速度）

---

#### 4.2 安装 Python 和 pip

```bash
sudo apt install python3-pip python3-venv -y
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `install` | 安装软件包 |
| `python3-pip` | Python 包管理器（用于安装第三方库） |
| `python3-venv` | 虚拟环境工具（创建独立 Python 环境） |
| `-y` | 自动确认 |

**安装内容**：
- Python 3.x 解释器（通常已预装）
- pip（包管理器）
- venv（虚拟环境模块）

**验证安装**：
```bash
python3 --version
pip3 --version
```

**期望输出**：
```
Python 3.10.12
pip 23.0.1 from ...
```

---

#### 4.3 创建虚拟环境

```bash
cd ~
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `cd` | 切换目录（Change Directory） |
| `~` | 用户主目录（如 `/home/username`） |

**为什么切换到主目录**：
- 保持环境集中管理
- 避免在系统目录创建虚拟环境

---

```bash
python3 -m venv vllm-env
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `python3` | Python 解释器 |
| `-m` | 作为模块运行 |
| `venv` | 虚拟环境模块 |
| `vllm-env` | 虚拟环境名称（可自定义） |

**创建的目录结构**：
```
~/vllm-env/
├── bin/           # 可执行文件（python, pip 等）
├── include/       # C 头文件
├── lib/           # Python 库和已安装的包
├── lib64 -> lib/  # 符号链接
└── pyvenv.cfg     # 虚拟环境配置文件
```

**创建时间**：10-30 秒

---

```bash
source vllm-env/bin/activate
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `source` | 读取并执行文件（在当前 shell 中） |
| `vllm-env/bin/activate` | 激活脚本路径 |

**激活后的变化**：
1. 修改环境变量 `PATH`
2. 将 `~/vllm-env/bin` 添加到 PATH 前端
3. 命令提示符前缀变为 `(vllm-env)`

**激活前**：
```
username@hostname:~$
```

**激活后**：
```
(vllm-env) username@hostname:~$
```

**验证激活**：
```bash
which python
```

**期望输出**：
```
/home/username/vllm-env/bin/python
```

**退出虚拟环境**：
```bash
deactivate
```

---

## vLLM 安装 - 每条命令详解

### 步骤 5：升级 pip

```bash
pip install --upgrade pip
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `pip` | Python 包管理器 |
| `install` | 安装软件包 |
| `--upgrade` | 升级到最新版本 |
| `pip` | 目标包名（升级 pip 自身） |

**为什么升级**：
- 新版本支持更好的依赖解析
- 修复已知 bug
- 提高下载速度和稳定性

**执行时间**：10-30 秒

---

### 步骤 6：安装 vLLM

#### 6.1 标准安装

```bash
pip install vllm
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `pip` | 包管理器 |
| `install` | 安装软件包 |
| `vllm` | 目标包名 |

**安装内容**：
- vLLM 核心库（约 1-2MB）
- 依赖包：
  - PyTorch（约 2GB）
  - transformers（约 100MB）
  - tokenizers（约 50MB）
  - 其他依赖（约 500MB）

**总下载量**：约 3-5GB
**执行时间**：5-20 分钟（取决于网络速度）

---

#### 6.2 使用镜像加速（国内推荐）

```bash
pip install vllm -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `-i` | 指定索引 URL（index） |
| `https://pypi.tuna.tsinghua.edu.cn/simple` | 清华大学 PyPI 镜像源 |

**其他可用镜像**：
- 阿里云：`https://mirrors.aliyun.com/pypi/simple/`
- 中科大：`https://pypi.mirrors.ustc.edu.cn/simple/`
- 华为云：`https://mirrors.huaweicloud.com/repository/pypi/simple/`

---

#### 6.3 验证安装

```bash
python -c "import vllm; print('vLLM installed successfully')"
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `python` | Python 解释器 |
| `-c` | 执行字符串中的 Python 代码 |
| `"import vllm; print('vLLM installed successfully')"` | 要执行的代码 |

**代码执行过程**：
1. `import vllm`：导入 vLLM 模块
2. `;`：语句分隔符
3. `print(...)`：打印成功消息

**期望输出**：
```
vLLM installed successfully
```

**如果失败**：
- `ModuleNotFoundError: No module named 'vllm'`：安装失败
- `ImportError`：依赖缺失或版本冲突

**完整测试**：
```bash
python -c "import vllm; from vllm import LLM, SamplingParams; print('All imports OK')"
```

---

## 模型启动 - 每条参数详解

### 步骤 7：启动 vLLM 服务

#### 完整命令（RTX 4090 + 14B 模型）

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
  --port 8000 \
  --host 0.0.0.0
```

---

#### 参数详解表

| 参数 | 值 | 必填 | 作用 | 调整建议 |
|------|---|------|------|---------|
| `--model` | `Qwen/Qwen2.5-14B-Instruct-AWQ` | ✅ | 模型标识符（Hugging Face） | 根据硬件选择 4B/7B/14B/32B |
| `--quantization` | `awq_marlin` | ❌ | 量化方法（4-bit） | 无显存限制时不填 |
| `--gpu-memory-utilization` | `0.9` | ❌ | GPU 显存利用率（0-1） | 显存紧张时降低到 0.8 |
| `--max-model-len` | `32768` | ❌ | 最大上下文长度（tokens） | 根据显存调整 8K/16K/32K |
| `--enable-auto-tool-choice` | （无值） | ❌ | 启用自动工具选择 | OpenClaw 必填 |
| `--tool-call-parser` | `hermes` | ❌ | 工具调用解析器 | OpenClaw 必填 |
| `--tensor-parallel-size` | `1` | ❌ | 张量并行度（单卡=1） | 多卡时增加 |
| `--dtype` | `float16` | ❌ | 数据类型 | bfloat16（A100/H100） |
| `--port` | `8000` | ❌ | 服务端口 | 冲突时改为 8001 |
| `--host` | `0.0.0.0` | ❌ | 监听地址 | 仅本机使用 127.0.0.1 |

---

#### 每个参数的深度解析

##### 参数 1：`--model`

**语法**：
```bash
--model <model_name>
```

**模型名称格式**：
- `<组织>/<模型名>`
- 示例：`Qwen/Qwen2.5-14B-Instruct-AWQ`

**可选模型**：

| 模型名 | 参数量 | 量化 | 显存需求 | 速度 | 精度 |
|--------|--------|------|---------|------|------|
| `Qwen/Qwen2.5-4B-Instruct-AWQ` | 4B | AWQ 4-bit | 8GB | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| `Qwen/Qwen2.5-7B-Instruct-AWQ` | 7B | AWQ 4-bit | 12GB | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| `Qwen/Qwen2.5-14B-Instruct-AWQ` | 14B | AWQ 4-bit | 24GB | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| `Qwen/Qwen2.5-32B-Instruct-AWQ` | 32B | AWQ 4-bit | 48GB | ⭐⭐ | ⭐⭐⭐⭐⭐ |

**非量化模型**（更高精度，更大显存）：
- `Qwen/Qwen2.5-7B-Instruct`（需 14GB+）
- `Qwen/Qwen2.5-14B-Instruct`（需 28GB+）

**选择原则**：
- 显存 8GB → 4B-AWQ
- 显存 12GB → 7B-AWQ
- 显存 24GB → 14B-AWQ（推荐）
- 显存 48GB+ → 32B-AWQ

---

##### 参数 2：`--quantization`

**语法**：
```bash
--quantization <method>
```

**可选方法**：

| 方法 | 精度 | 速度 | 显存节省 | 兼容性 |
|------|------|------|---------|--------|
| `awq_marlin` | 4-bit | ⭐⭐⭐⭐⭐ | 75% | RTX 30/40 系列 |
| `awq` | 4-bit | ⭐⭐⭐ | 75% | 所有 GPU |
| `gptq` | 4-bit | ⭐⭐ | 75% | 所有 GPU（不推荐） |
| `none` | 16-bit | ⭐⭐⭐⭐ | 0% | 所有 GPU |

**AWQ Marlin 详解**：
- **AWQ（Activation-aware Weight Quantization）**：
  - 激活感知权重量化
  - 保持激活值的精度
  - 比 GPTQ 更稳定

- **Marlin 优化**：
  - 针对 RTX 40 系列优化的 CUDA 内核
  - 利用 Tensor Cores 加速
  - 比标准 AWQ 快 2-3 倍

**什么时候不使用量化**：
- 显存充足（48GB+）
- 需要最高精度
- 使用 A100/H100 等专业显卡

---

##### 参数 3：`--gpu-memory-utilization`

**语法**：
```bash
--gpu-memory-utilization <ratio>
```

**值域**：`0.0` - `1.0`（推荐 `0.7` - `0.95`）

**不同显卡的建议值**：

| 显卡 | 显存 | 推荐值 | 理由 |
|------|------|--------|------|
| RTX 4060 | 8GB | 0.80 | 留 20% 给 KV Cache |
| RTX 4070 | 12GB | 0.85 | 留 15% 缓冲 |
| RTX 4080 | 16GB | 0.85 | 平衡性能和稳定 |
| RTX 4090 | 24GB | 0.90 | 充分利用显存 |
| A6000 | 48GB | 0.90 | 留 10% 缓冲 |

**KV Cache 计算公式**：
```
KV Cache = batch_size × max_len × hidden_size × num_layers × 2 × 2 bytes

示例（Qwen 14B）：
= 1 × 32768 × 5120 × 40 × 2 × 2
= 26.8 GB

这超过了 24GB 显存，所以必须降低 max-model-len 或 gpu-memory-utilization
```

**调整策略**：
- 出现 OOM 错误 → 降低 0.05（如 0.9 → 0.85）
- 显存大量闲置 → 提高 0.05
- 长上下文场景 → 降低 0.1

---

##### 参数 4：`--max-model-len`

**语法**：
```bash
--max-model-len <tokens>
```

**常用值**：`4096`, `8192`, `16384`, `32768`

**显存与上下文关系**：

| 显存 | 模型 | 推荐 max-model-len | 理由 |
|------|------|-------------------|------|
| 8GB | 4B-AWQ | 8192 | 基础模型权重约 5GB，剩余 3GB 给 KV Cache |
| 12GB | 7B-AWQ | 16384 | 模型权重 8GB，剩余 4GB 支持 16K 上下文 |
| 16GB | 7B-AWQ | 32768 | 充足显存支持 32K 上下文 |
| 24GB | 14B-AWQ | 32768 | 模型权重 10GB，剩余 14GB 支持 32K |

**Token 与字数换算**：
```
中文：1 token ≈ 1.5-2 个汉字
英文：1 token ≈ 4 个字符

示例：
8192 tokens ≈ 12000-16000 中文字符
32768 tokens ≈ 49000-65000 中文字符
```

**选择原则**：
- 日常对话：`4096` - `8192`
- 文档分析：`16384` - `32768`
- 代码审查：`8192` - `16384`

---

##### 参数 5：`--enable-auto-tool-choice`

**语法**：
```bash
--enable-auto-tool-choice
```

**无参数值**：仅开关作用

**作用**：
- 允许模型自动决定何时调用工具
- 无需在提示中明确要求"使用工具"
- 提升 Agent 的智能化程度

**工作原理**：
```
用户输入 → 模型分析 → 自动判断是否需要工具
                          ↓
                    是 → 调用工具
                    否 → 直接生成文本
```

**为什么 OpenClaw 需要**：
- OpenClaw 是自动化框架
- 工具调用是核心功能
- 必须启用才能正常工作

---

##### 参数 6：`--tool-call-parser`

**语法**：
```bash
--tool-call-parser <parser_name>
```

**可选解析器**：

| 解析器 | 兼容性 | 特点 |
|--------|--------|------|
| `hermes` | ⭐⭐⭐⭐⭐ | OpenAI 格式，推荐 |
| `openai` | ⭐⭐⭐⭐ | 标准 OpenAI 格式 |
| `mistral` | ⭐⭐⭐ | Mistral 模型专用 |
| `qwen` | ⭐⭐⭐⭐ | Qwen 模型原生 |

**Hermes 解析器特点**：
- 从 Llama-2-Chat 演化而来
- 支持复杂工具调用场景
- 完全兼容 OpenAI API

**为什么推荐 Hermes**：
- 兼容性最好
- 支持多轮工具调用
- 解析准确率高

---

##### 参数 7：`--tensor-parallel-size`

**语法**：
```bash
--tensor-parallel-size <num_gpus>
```

**值域**：`1`（单卡），`2`, `4`, `8`（多卡）

**单卡部署**：
```bash
--tensor-parallel-size 1
```

**双卡部署（如 RTX 4090 x2）**：
```bash
--tensor-parallel-size 2
```

**多卡分配示例**（双 4090）：
```
GPU 0: 模型前半部分（50%）
GPU 1: 模型后半部分（50%）
```

**注意**：
- 多卡需要相同型号的显卡
- 多卡通过 NVLink 或 PCIe 通信
- NVLink 性能更好（推荐）

---

##### 参数 8：`--dtype`

**语法**：
```bash
--dtype <data_type>
```

**可选类型**：

| 类型 | 精度 | 显存占用 | 速度 | 适用显卡 |
|------|------|---------|------|---------|
| `float16` (FP16) | 16-bit | 标准 | 快 | 所有 GPU |
| `bfloat16` (BF16) | 16-bit | 标准 | 更快 | RTX 30/40, A100, H100 |
| `float32` (FP32) | 32-bit | 2x | 慢 | 不推荐 |
| `auto` | 自动选择 | - | - | 推荐 |

**FP16 vs BF16**：
- **FP16**：传统的半精度浮点
  - 范围：65504 到 -65504
  - 优点：兼容所有 GPU
  - 缺点：精度较低

- **BF16**：Brain Float 16
  - 范围：3.38×10^38 到 -3.38×10^38
  - 优点：与 FP32 精度接近，动态范围大
  - 缺点：仅支持较新的 GPU

**选择建议**：
- RTX 30/40 系列 → `bfloat16`
- A100/H100 → `bfloat16`
- 旧显卡 → `float16`
- 不确定 → `auto`

---

##### 参数 9：`--port`

**语法**：
```bash
--port <port_number>
```

**默认值**：`8000`

**常用端口**：
- `8000`：vLLM 默认
- `8001`, `8002`：多实例
- `11434`：Ollama 端口

**端口冲突检测**：
```bash
# 检查端口是否被占用
lsof -i :8000
# 或
netstat -tlnp | grep 8000
```

**如果端口被占用**：
```bash
--port 8001
```

---

##### 参数 10：`--host`

**语法**：
```bash
--host <host_address>
```

**常用地址**：

| 地址 | 作用 |
|------|------|
| `0.0.0.0` | 监听所有网卡（可远程访问） |
| `127.0.0.1` | 仅监听本机（更安全） |

**安全建议**：
- 仅本机使用 → `--host 127.0.0.1`
- 局域网访问 → `--host 0.0.0.0` + 配置防火墙
- 公网访问 → `--host 0.0.0.0` + 反向代理（Nginx） + 认证

---

### 不同硬件配置的完整启动命令

#### 配置 1：RTX 4060 (8GB) + 4B 模型

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-4B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.80 \
  --max-model-len 8192 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --dtype float16 \
  --host 127.0.0.1 \
  --port 8000
```

**参数说明**：
- 4B 小模型，适配 8GB 显存
- `gpu-memory-utilization 0.80`：留足 KV Cache 空间
- `max-model-len 8192`：8K tokens，小显存友好
- `float16`：兼容 RTX 4060

**预期性能**：
- Token 速度：150-200 tokens/s
- 首延迟：0.3-0.5 秒
- 显存占用：6-7GB

---

#### 配置 2：RTX 4070 (12GB) + 7B 模型

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-7B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.85 \
  --max-model-len 16384 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --dtype bfloat16 \
  --host 127.0.0.1 \
  --port 8000
```

**参数说明**：
- 7B 模型，平衡性能和显存
- `gpu-memory-utilization 0.85`：适中配置
- `max-model-len 16384`：16K tokens，足够日常使用
- `bfloat16`：RTX 4070 支持，性能更好

**预期性能**：
- Token 速度：120-170 tokens/s
- 首延迟：0.4-0.6 秒
- 显存占用：9-10GB

---

#### 配置 3：RTX 4080 (16GB) + 14B 模型

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-14B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.85 \
  --max-model-len 32768 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --dtype bfloat16 \
  --host 127.0.0.1 \
  --port 8000
```

**参数说明**：
- 14B 模型，更高精度
- `gpu-memory-utilization 0.85`：16GB 显存需留缓冲
- `max-model-len 32768`：32K tokens，支持长文档

**预期性能**：
- Token 速度：90-130 tokens/s
- 首延迟：0.5-0.8 秒
- 显存占用：13-14GB

---

#### 配置 4：RTX 4090 (24GB) + 14B 模型（推荐）

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-14B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.90 \
  --max-model-len 32768 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --dtype bfloat16 \
  --host 127.0.0.1 \
  --port 8000
```

**参数说明**：
- 14B 模型，最佳性能选择
- `gpu-memory-utilization 0.90`：充分利用 24GB 显存
- `max-model-len 32768`：32K tokens，长上下文

**预期性能**：
- Token 速度：90-130 tokens/s
- 首延迟：0.4-0.8 秒
- 显存占用：10-12GB

---

#### 配置 5：A6000 (48GB) + 32B 模型

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-32B-Instruct-AWQ \
  --quantization awq_marlin \
  --gpu-memory-utilization 0.90 \
  --max-model-len 65536 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --dtype bfloat16 \
  --host 127.0.0.1 \
  --port 8000
```

**参数说明**：
- 32B 模型，企业级性能
- `max-model-len 65536`：64K tokens，超长上下文
- 充足显存支持大模型和长上下文

**预期性能**：
- Token 速度：60-90 tokens/s
- 首延迟：0.6-1.0 秒
- 显存占用：35-38GB

---

## OpenClaw 配置 - 每条命令详解

### 步骤 8：安装 Node.js

#### 8.1 添加 NodeSource 仓库

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `curl` | 下载工具（Client URL） |
| `-f` | 静默模式，不显示进度条 |
| `-s` | 静默模式，不显示错误 |
| `-S` | 显示错误（与 -s 配合使用） |
| `-L` | 跟随重定向 |
| `https://deb.nodesource.com/setup_22.x` | NodeSource 仓库设置脚本 |
| `|` | 管道，传递给下一个命令 |
| `sudo -E bash -` | 以 root 权限执行脚本 |

**为什么 NodeSource**：
- Ubuntu 默认源的 Node.js 版本较旧
- NodeSource 提供最新 LTS 版本
- `22.x` 是 Node.js 22.x LTS

**脚本做了什么**：
1. 下载 GPG 密钥（验证包签名）
2. 添加 NodeSource 仓库到 apt
3. 更新软件包索引

---

#### 8.2 安装 Node.js

```bash
sudo apt install -y nodejs
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `sudo` | root 权限 |
| `apt` | 包管理器 |
| `install` | 安装软件包 |
| `-y` | 自动确认 |
| `nodejs` | Node.js（包含 npm） |

**安装内容**：
- Node.js 22.x 解释器
- npm（包管理器）
- 相关依赖

**验证安装**：
```bash
node --version
npm --version
```

**期望输出**：
```
v22.11.0
10.9.0
```

---

### 步骤 9：安装 OpenClaw

```bash
sudo npm install -g openclaw@latest
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `sudo` | root 权限（全局安装需要） |
| `npm` | Node.js 包管理器 |
| `install` | 安装软件包 |
| `-g` | 全局安装（global） |
| `openclaw@latest` | 包名 @ 版本 |

**全局 vs 本地安装**：
- **全局（-g）**：安装到 `/usr/local/lib/node_modules/`，任何目录可使用
- **本地**：安装到 `./node_modules/`，仅当前项目可用

**@latest 含义**：
- 安装最新稳定版
- 也可指定版本：`@1.2.3`、`@beta`、`@rc`

**安装位置**：
```
命令：/usr/local/bin/openclaw
包：/usr/local/lib/node_modules/openclaw/
```

**安装时间**：1-3 分钟（取决于网络）

---

#### 验证安装

```bash
openclaw --version
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `openclaw` | OpenClaw 命令 |
| `--version` | 显示版本号 |

**期望输出**：
```
OpenClaw v2.x.x
```

---

### 步骤 10：配置 OpenClaw

#### 10.1 启动配置向导

```bash
openclaw onboard
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `openclaw` | OpenClaw 命令 |
| `onboard` | 配置向导子命令 |

**向导流程**：

**步骤 1：选择模型提供商**
```
? Select model provider: (Use arrow keys)
❯ Custom
  OpenAI
  Azure OpenAI
  Anthropic
  Google
```

**选择原因**：
- `Custom`：自定义 API 端点（本地 vLLM）
- 其他：云端服务

**步骤 2：输入 Base URL**
```
? Enter base URL: http://127.0.0.1:8000/v1
```

**参数详解**：
| 部分 | 说明 |
|------|------|
| `http://` | 协议（非加密） |
| `127.0.0.1` | 本机 IP（仅本机访问） |
| `8000` | vLLM 端口（与启动命令一致） |
| `/v1` | OpenAI API 路径前缀 |

**为什么 `/v1`**：
- vLLM 兼容 OpenAI API
- 所有请求都在 `/v1` 路径下
- 确保与其他服务区分

---

**步骤 3：输入 API Key**
```
? Enter API key: 123456
```

**参数说明**：
- 本地服务不需要真实的 API Key
- 输入任意值即可（如 `123456`、`sk-xxxxx`、`dummy`）
- 用于格式兼容，不验证有效性

**为什么需要 API Key**：
- OpenAI API 规范要求
- 模拟真实 API 调用
- 方便切换到云端服务

---

**步骤 4：输入模型名称**
```
? Enter model name: Qwen2.5-14B-Instruct-AWQ
```

**关键点**：
- **必须与 vLLM 启动命令的 `--model` 参数完全一致**
- **区分大小写**
- **不能省略前缀**（如 `Qwen/`）

**错误示例**：
- ❌ `qwen2.5`（大小写错误）
- ❌ `Qwen2.5-14B`（省略前缀）
- ✅ `Qwen/Qwen2.5-14B-Instruct-AWQ`

---

**步骤 5：确认并保存**
```
? Save this configuration? (Y/n) Y
```

**保存位置**：
```
~/.openclaw/config.json
```

**配置文件示例**：
```json
{
  "models": [
    {
      "id": "custom-001",
      "provider": "custom",
      "baseUrl": "http://127.0.0.1:8000/v1",
      "apiKey": "123456",
      "name": "Qwen/Qwen2.5-14B-Instruct-AWQ",
      "enabled": true
    }
  ],
  "defaultModel": "custom-001"
}
```

**字段说明**：
| 字段 | 说明 |
|------|------|
| `id` | 唯一标识符（自动生成） |
| `provider` | 提供商类型（`custom`） |
| `baseUrl` | API 端点 |
| `apiKey` | 认证密钥 |
| `name` | 模型名称 |
| `enabled` | 是否启用 |
| `defaultModel` | 默认使用的模型 |

---

#### 10.2 手动编辑配置文件

如果向导有问题，可直接编辑：

```bash
nano ~/.openclaw/config.json
```

**编辑后保存**：`Ctrl + O` → `Enter` → `Ctrl + X`

**验证配置**：
```bash
openclaw config list
```

**期望输出**：
```
ID           Provider   Model
custom-001   custom     Qwen/Qwen2.5-14B-Instruct-AWQ
```

---

## 参数调优 - 详细配置表

### OpenClaw 参数配置

#### 配置界面访问

```bash
openclaw config
```

或

```bash
openclaw configure
```

---

#### 核心参数详解

| 参数 | 说明 | 推荐范围 | 影响因素 |
|------|------|---------|---------|
| **Context length** | 单次对话上下文长度（tokens） | 4000-12000 | 显存、任务类型 |
| **Temperature** | 生成随机性（0-2.0） | 0.5-0.9 | 任务类型 |
| **Max tokens** | 单次回复最大长度（tokens） | 512-4096 | 任务复杂度 |
| **Top p** | 核采样概率（0-1） | 0.8-0.95 | 输出多样性 |
| **Frequency penalty** | 频率惩罚（0-2） | 0-0.5 | 重复性控制 |
| **Presence penalty** | 存在惩罚（0-2） | 0-0.5 | 主题多样性 |

---

#### Context length（上下文长度）

**值域**：`1024` - `32768`

**选择指南**：

| 场景 | 推荐值 | 理由 |
|------|--------|------|
| 快速对话 | 3000-5000 | 响应快，占用少 |
| 日常任务 | 6000-8000 | 平衡性能和记忆 |
| 代码生成 | 8000-10000 | 需要更多上下文 |
| 文档分析 | 10000-15000 | 长文档理解 |
| 长对话 | 12000-20000 | 多轮对话记忆 |

**显存与 Context length 的关系**：
```
KV Cache 显存 ≈ Context length × 隐藏层大小 × 层数 × 2 × 2 bytes

Qwen 14B（AWQ 4-bit）：
- 模型权重：~10GB
- KV Cache（8K tokens）：~2GB
- KV Cache（16K tokens）：~4GB
- KV Cache（32K tokens）：~8GB

总显存 = 模型权重 + KV Cache + 缓冲

RTX 4090（24GB）：
- 8K 上下文：10 + 2 + 2 = 14GB ✓
- 16K 上下文：10 + 4 + 2 = 16GB ✓
- 32K 上下文：10 + 8 + 6 = 24GB ⚠️
```

**调整建议**：
- 显存充足 → 提高 Context length
- 性能下降 → 降低 Context length
- 多轮对话卡顿 → 添加摘要指令（见后文）

---

#### Temperature（温度参数）

**值域**：`0.0` - `2.0`

**效果对比**：

| 值 | 效果 | 适用场景 |
|----|------|---------|
| **0.0-0.2** | 完全确定性，每次回复相同 | 代码生成、数学计算、需要准确答案 |
| **0.3-0.5** | 低随机性，稳定输出 | 技术文档、翻译、结构化输出 |
| **0.6-0.8** | 适度随机，平衡创造性和稳定性 | 日常对话、任务执行（推荐） |
| **0.9-1.1** | 高随机性，多样输出 | 创意写作、头脑风暴 |
| **1.2-2.0** | 极高随机性，可能不稳定 | 实验性、极端创意 |

**实际测试示例**：

**Prompt**："请用一句话描述春天。"

- **Temperature = 0.2**：
  > 春天是万物复苏的季节，草木萌发，鸟语花香。

- **Temperature = 0.7**（推荐）：
  > 春天是生命的赞歌，绿意盎然，充满希望。

- **Temperature = 1.5**：
  > 春天像一首流动的诗，在每一片新叶上写下生命的奇迹。

**调优建议**：
- 代码生成 → `0.2-0.4`
- 文档写作 → `0.5-0.7`
- 日常对话 → `0.7-0.8`
- 创意写作 → `0.9-1.2`

---

#### Max tokens（最大生成长度）

**值域**：`512` - `4096`

**选择指南**：

| 任务类型 | 推荐值 | 理由 |
|---------|--------|------|
| 快速问答 | 256-512 | 简短回答 |
| 代码片段 | 512-1024 | 完整函数 |
| 文章段落 | 1024-2048 | 详细段落 |
| 完整文章 | 2048-4096 | 长文本 |

**注意**：
- 实际生成可能少于该值
- 如果内容完整，模型会提前停止
- 设置过大浪费资源

---

#### Top p（核采样）

**值域**：`0.0` - `1.0`

**说明**：
- 从概率最高的 tokens 中累积概率达到 p 时停止
- 控制输出多样性

**示例**：
```
假设 p = 0.9
Token 概率分布：
A: 0.40
B: 0.30
C: 0.15
D: 0.10
E: 0.05

累积：
A: 0.40
B: 0.70
C: 0.85
D: 0.95 ✓（达到 0.9）

从 A, B, C, D 中采样，排除 E
```

**推荐值**：
- **0.8-0.9**：多样性与稳定性平衡
- **0.95+**：更多样，但不稳定
- **0.6-0.7**：更稳定，但可能重复

---

#### Frequency & Presence Penalty

**值域**：`0.0` - `2.0`

**区别**：

| 惩罚类型 | 作用 | 适用场景 |
|---------|------|---------|
| **Frequency penalty** | 惩罚频繁出现的 tokens | 减少重复词汇 |
| **Presence penalty** | 惩罚已经出现过的 tokens | 增加主题多样性 |

**示例**：
- 频率惩罚 `0.5`：减少"然后"、"因此"等词的重复
- 存在惩罚 `0.5`：鼓励模型讨论新话题

**推荐配置**：
- 日常对话：频率 `0.0`，存在 `0.0`（不惩罚）
- 长文本生成：频率 `0.3`，存在 `0.3`
- 创意写作：频率 `0.0`，存在 `0.6`（更多样化）

---

### 不同场景的完整配置

#### 场景 1：快速对话（RTX 4070 + 7B 模型）

**vLLM 启动参数**：
```bash
--model Qwen/Qwen2.5-7B-Instruct-AWQ \
--gpu-memory-utilization 0.85 \
--max-model-len 16384 \
--quantization awq_marlin
```

**OpenClaw 配置**：
```
Context length: 6000
Temperature: 0.7
Max tokens: 1024
Top p: 0.9
Frequency penalty: 0.0
Presence penalty: 0.0
```

**预期效果**：
- 响应速度：120-170 tokens/s
- 首延迟：0.4-0.6 秒
- 对话质量：流畅、自然

---

#### 场景 2：代码生成（RTX 4090 + 14B 模型）

**vLLM 启动参数**：
```bash
--model Qwen/Qwen2.5-14B-Instruct-AWQ \
--gpu-memory-utilization 0.9 \
--max-model-len 32768 \
--quantization awq_marlin \
--dtype bfloat16
```

**OpenClaw 配置**：
```
Context length: 10000
Temperature: 0.3
Max tokens: 2048
Top p: 0.8
Frequency penalty: 0.0
Presence penalty: 0.0
```

**预期效果**：
- 代码准确性：高
- 格式规范性：好
- 响应速度：90-130 tokens/s

---

#### 场景 3：创意写作（RTX 4080 + 14B 模型）

**vLLM 启动参数**：
```bash
--model Qwen/Qwen2.5-14B-Instruct-AWQ \
--gpu-memory-utilization 0.85 \
--max-model-len 32768 \
--quantization awq_marlin
```

**OpenClaw 配置**：
```
Context length: 12000
Temperature: 1.0
Max tokens: 4096
Top p: 0.95
Frequency penalty: 0.3
Presence penalty: 0.6
```

**预期效果**：
- 创意丰富度：高
- 多样性：好
- 重复性：低

---

#### 场景 4：长文档分析（RTX 4090 + 14B 模型）

**vLLM 启动参数**：
```bash
--model Qwen/Qwen2.5-14B-Instruct-AWQ \
--gpu-memory-utilization 0.9 \
--max-model-len 32768 \
--quantization awq_marlin \
--enable-auto-tool-choice \
--tool-call-parser hermes
```

**OpenClaw 配置**：
```
Context length: 15000
Temperature: 0.5
Max tokens: 2048
Top p: 0.9
Frequency penalty: 0.0
Presence penalty: 0.0
```

**System Prompt 添加**：
> When the conversation becomes long, summarize previous messages into a short memory. Keep the memory under 200 tokens.

**预期效果**：
- 文档理解：深入
- 上下文保持：稳定
- 性能下降：避免

---

## 性能监控命令

### 实时监控 GPU 状态

#### 1. 持续监控（每秒刷新）

```bash
watch -n 1 nvidia-smi
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `watch` | 周期性执行命令 |
| `-n 1` | 每 1 秒刷新一次 |
| `nvidia-smi` | NVIDIA 系统管理界面 |

**退出监控**：`Ctrl + C`

**关注的指标**：
- `GPU-Util`：GPU 利用率（70-95% 为正常）
- `Memory-Usage`：显存使用情况
- `Temp`：GPU 温度（50-75°C 为正常）
- `Fan`：风扇转速

---

#### 2. 查看 GPU 详细信息

```bash
nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,utilization.memory,memory.total,memory.used,memory.free --format=csv
```

**命令详解**：
| 参数 | 说明 |
|------|------|
| `--query-gpu=...` | 查询 GPU 属性 |
| `index` | GPU 编号 |
| `name` | GPU 名称 |
| `temperature.gpu` | GPU 温度 |
| `utilization.gpu` | GPU 利用率（百分比） |
| `utilization.memory` | 显存利用率（百分比） |
| `memory.total` | 总显存（MB） |
| `memory.used` | 已用显存（MB） |
| `memory.free` | 空闲显存（MB） |
| `--format=csv` | CSV 格式输出 |

**期望输出**：
```
index, name, temperature.gpu, utilization.gpu, utilization.memory, memory.total, memory.used, memory.free
0, NVIDIA GeForce RTX 4090, 55, 95, 50, 24576, 12288, 12288
```

---

#### 3. 监控 vLLM 进程

```bash
ps aux | grep vllm
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `ps` | 进程状态 |
| `aux` | 显示所有用户进程（详细） |
| `|` | 管道 |
| `grep vllm` | 过滤包含 "vllm" 的行 |

**期望输出**：
```
username   12345  0.5 15.0  24576128 10240000 pts/0  Sl+  22:30   0:05 python -m vllm.entrypoints.openai.api_server --model Qwen/Qwen2.5-14B-Instruct-AWQ ...
```

**字段说明**：
- `PID 12345`：进程 ID
- `%CPU 0.5`：CPU 使用率
- `%MEM 15.0`：内存使用率
- `VSZ 24576128`：虚拟内存大小（KB）
- `RSS 10240000`：常驻内存大小（KB，实际物理内存）

---

#### 4. 监控端口状态

```bash
netstat -tlnp | grep 8000
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `netstat` | 网络状态 |
| `-t` | TCP 连接 |
| `-l` | 监听端口 |
| `-n` | 数字格式（不解析主机名） |
| `-p` | 显示进程信息 |

**期望输出**：
```
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      12345/python
```

**说明**：
- `0.0.0.0:8000`：监听所有网卡的 8000 端口
- `LISTEN`：监听状态
- `12345/python`：进程 ID 和进程名

---

#### 5. 查看 vLLM 日志

如果 vLLM 后台运行，查看日志：

```bash
journalctl -u vllm -f
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `journalctl` | 系统日志 |
| `-u vllm` | 指定服务（需要配置 systemd） |
| `-f` | 持续刷新 |

**如果没有配置 systemd**，查看启动时的输出（未后台运行）。

---

## 故障排查 - 诊断命令

### 问题 1：CUDA Out of Memory

#### 诊断命令

```bash
nvidia-smi
```

**检查内容**：
1. 显存使用率是否接近 100%
2. Memory-Usage 是否接近总显存
3. 是否有其他进程占用显存

**解决方案**：

**方案 1：降低 GPU 内存利用率**
```bash
# 修改启动命令
--gpu-memory-utilization 0.8  # 从 0.9 降到 0.8
```

**方案 2：降低上下文长度**
```bash
--max-model-len 16384  # 从 32768 降到 16384
```

**方案 3：换用小模型**
```bash
--model Qwen/Qwen2.5-7B-Instruct-AWQ  # 从 14B 换到 7B
```

**方案 4：释放其他进程显存**
```bash
# 查看占用显存的进程
nvidia-smi

# 如果有其他 GPU 进程，停止它们
pkill -f <进程名>
```

---

### 问题 2：模型下载失败

#### 诊断命令

```bash
curl -I https://huggingface.co
```

**命令详解**：
| 命令部分 | 作用 |
|---------|------|
| `curl` | HTTP 请求工具 |
| `-I` | 仅显示响应头（不下载内容） |
| `https://huggingface.co` | Hugging Face 主页 |

**期望输出**：
```
HTTP/2 200
date: Thu, 20 Mar 2026 15:30:00 GMT
...
```

**如果连接失败**：
```
curl: (7) Failed to connect to huggingface.co
```

**解决方案**：

**方案 1：使用 HF 镜像站**
```bash
export HF_ENDPOINT=https://hf-mirror.com
# 然后启动 vLLM
```

**方案 2：使用代理**
```bash
export https_proxy=http://127.0.0.1:7890
export http_proxy=http://127.0.0.1:7890
```

**方案 3：手动下载模型**
```bash
# 安装 git-lfs
sudo apt install git-lfs

# 克隆模型（使用镜像）
git lfs clone https://hf-mirror.com/Qwen/Qwen2.5-14B-Instruct-AWQ

# 启动时指定本地路径
--model /path/to/Qwen2.5-14B-Instruct-AWQ
```

---

### 问题 3：OpenClaw 无法连接

#### 诊断命令

**步骤 1：检查 vLLM 是否运行**
```bash
ps aux | grep vllm
```

**如果无输出**：vLLM 未启动，需重新启动

---

**步骤 2：检查端口是否监听**
```bash
netstat -tlnp | grep 8000
```

**期望输出**：
```
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      12345/python
```

**如果无输出**：端口未监听，需检查 vLLM 启动日志

---

**步骤 3：测试 API 连接**
```bash
curl http://127.0.0.1:8000/v1/models
```

**期望输出**：
```json
{
  "object": "list",
  "data": [
    {
      "id": "Qwen/Qwen2.5-14B-Instruct-AWQ",
      ...
    }
  ]
}
```

**如果失败**：
- `Connection refused`：端口未监听
- `Connection timeout`：vLLM 未响应
- `curl: command not found`：需安装 curl

---

**步骤 4：检查防火墙**
```bash
sudo ufw status
```

**如果防火墙启用**：
```bash
sudo ufw allow 8000
```

---

### 问题 4：性能下降

#### 诊断命令

**步骤 1：检查 GPU 利用率**
```bash
watch -n 1 nvidia-smi
```

**正常情况**：
- GPU-Util：70-95%
- 温度：50-75°C

**如果 GPU 利用率低（<50%）**：
- 可能是 CPU 瓶颈
- 可能是数据传输延迟

**如果 GPU 温度过高（>80°C）**：
- 可能是散热问题
- 可能会触发降频

---

**步骤 2：检查 CPU 利用率**
```bash
top
```

**按 `1` 查看各 CPU 核心**

**如果 CPU 利用率高（>80%）**：
- 可能是预处理/后处理瓶颈
- 可能是 batch size 太大

---

**步骤 3：检查内存使用**
```bash
free -h
```

**如果内存不足（<1GB）**：
- 可能导致频繁 swap
- 影响 vLLM 性能

---

**解决方案**：

**方案 1：增加 batch size**
```bash
python -m vllm.entrypoints.openai.api_server \
  ... \
  --max-num-seqs 8  # 默认 256，可根据显存调整
```

**方案 2：减少上下文长度**
```bash
--max-model-len 16384  # 从 32768 降到 16384
```

**方案 3：添加摘要指令**（避免长对话性能下降）
> When the conversation becomes long, summarize previous messages into a short memory. Keep the memory under 200 tokens.

---

### 问题 5：工具调用不生效

#### 诊断步骤

**步骤 1：检查启动参数**
```bash
ps aux | grep vllm
```

**确认包含**：
- `--enable-auto-tool-choice`
- `--tool-call-parser hermes`

---

**步骤 2：测试工具调用**
```bash
curl -X POST http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-14B-Instruct-AWQ",
    "messages": [
      {"role": "user", "content": "What is the current time?"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_current_time",
          "description": "Get the current time"
        }
      }
    ]
  }'
```

**期望输出**：包含 `tool_calls` 字段

---

**解决方案**：

**方案 1：确认模型支持工具调用**
- 使用 `Instruct` 版本（如 `Qwen2.5-14B-Instruct-AWQ`）
- 避免使用 Base 模型

**方案 2：检查 OpenClaw System Prompt**
- 确保提示中包含工具使用说明

**方案 3：调整 tool-call-parser**
```bash
--tool-call-parser hermes  # 或 qwen、openai
```

---

## 附录：实用脚本

### 启动脚本（start_vllm.sh）

```bash
#!/bin/bash

# ========================================
# vLLM 启动脚本
# ========================================

# 配置区域（根据你的硬件修改）
MODEL="Qwen/Qwen2.5-14B-Instruct-AWQ"
QUANTIZATION="awq_marlin"
GPU_MEMORY_UTIL="0.9"
MAX_MODEL_LEN="32768"
DTYPE="bfloat16"
PORT="8000"
HOST="127.0.0.1"

# ========================================
# 激活虚拟环境
# ========================================
source ~/vllm-env/bin/activate

# ========================================
# 设置环境变量
# ========================================
export HF_ENDPOINT=https://hf-mirror.com
export CUDA_VISIBLE_DEVICES=0

# ========================================
# 启动 vLLM
# ========================================
echo "========================================"
echo "启动 vLLM 服务"
echo "========================================"
echo "模型: $MODEL"
echo "量化: $QUANTIZATION"
echo "GPU 利用率: $GPU_MEMORY_UTIL"
echo "最大上下文: $MAX_MODEL_LEN tokens"
echo "数据类型: $DTYPE"
echo "端口: $PORT"
echo "========================================"
echo ""

python -m vllm.entrypoints.openai.api_server \
  --model $MODEL \
  --quantization $QUANTIZATION \
  --gpu-memory-utilization $GPU_MEMORY_UTIL \
  --max-model-len $MAX_MODEL_LEN \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --dtype $DTYPE \
  --port $PORT \
  --host $HOST
```

**使用方法**：
```bash
chmod +x start_vllm.sh
./start_vllm.sh
```

---

### 测试脚本（test_vllm.py）

```python
#!/usr/bin/env python3
"""
vLLM API 测试脚本
"""

import requests
import json
import time

# ========================================
# 配置
# ========================================
API_URL = "http://127.0.0.1:8000/v1/chat/completions"
MODEL_NAME = "Qwen/Qwen2.5-14B-Instruct-AWQ"

# ========================================
# 测试 1：简单对话
# ========================================
def test_simple_chat():
    print("=" * 50)
    print("测试 1：简单对话")
    print("=" * 50)
    
    payload = {
        "model": MODEL_NAME,
        "messages": [
            {"role": "user", "content": "你好，请用一句话介绍 vLLM。"}
        ],
        "temperature": 0.7,
        "max_tokens": 100
    }
    
    start_time = time.time()
    response = requests.post(API_URL, json=payload)
    end_time = time.time()
    
    print(f"响应时间: {end_time - start_time:.2f} 秒")
    print(f"状态码: {response.status_code}")
    
    if response.status_code == 200:
        result = response.json()
        print(f"回复: {result['choices'][0]['message']['content']}")
    else:
        print(f"错误: {response.text}")
    
    print()

# ========================================
# 测试 2：工具调用
# ========================================
def test_tool_calling():
    print("=" * 50)
    print("测试 2：工具调用")
    print("=" * 50)
    
    payload = {
        "model": MODEL_NAME,
        "messages": [
            {"role": "user", "content": "北京现在几点了？"}
        ],
        "tools": [
            {
                "type": "function",
                "function": {
                    "name": "get_current_time",
                    "description": "获取当前时间",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "timezone": {
                                "type": "string",
                                "description": "时区"
                            }
                        },
                        "required": ["timezone"]
                    }
                }
            }
        ],
        "temperature": 0.7
    }
    
    start_time = time.time()
    response = requests.post(API_URL, json=payload)
    end_time = time.time()
    
    print(f"响应时间: {end_time - start_time:.2f} 秒")
    print(f"状态码: {response.status_code}")
    
    if response.status_code == 200:
        result = response.json()
        message = result['choices'][0]['message']
        print(f"是否调用工具: {'tool_calls' in message}")
        
        if 'tool_calls' in message:
            print(f"工具调用: {json.dumps(message['tool_calls'], indent=2, ensure_ascii=False)}")
        else:
            print(f"回复: {message['content']}")
    else:
        print(f"错误: {response.text}")
    
    print()

# ========================================
# 测试 3：流式输出
# ========================================
def test_streaming():
    print("=" * 50)
    print("测试 3：流式输出")
    print("=" * 50)
    
    payload = {
        "model": MODEL_NAME,
        "messages": [
            {"role": "user", "content": "请写一首关于春天的短诗。"}
        ],
        "temperature": 0.8,
        "stream": True
    }
    
    response = requests.post(API_URL, json=payload, stream=True)
    
    print("生成内容：")
    print("-" * 50)
    
    for line in response.iter_lines():
        if line:
            line = line.decode('utf-8')
            if line.startswith('data: '):
                data = line[6:]  # 去掉 'data: ' 前缀
                if data != '[DONE]':
                    try:
                        chunk = json.loads(data)
                        content = chunk['choices'][0]['delta'].get('content', '')
                        print(content, end='', flush=True)
                    except json.JSONDecodeError:
                        pass
    
    print()
    print()

# ========================================
# 主函数
# ========================================
if __name__ == "__main__":
    print("\n开始 vLLM API 测试\n")
    
    try:
        test_simple_chat()
        test_tool_calling()
        test_streaming()
        
        print("=" * 50)
        print("所有测试完成")
        print("=" * 50)
        
    except requests.exceptions.ConnectionError:
        print("错误: 无法连接到 vLLM 服务")
        print("请确认 vLLM 已启动并运行在 http://127.0.0.1:8000")
    except Exception as e:
        print(f"错误: {e}")
```

**使用方法**：
```bash
python3 test_vllm.py
```

---

## 总结

### 快速部署流程

```
1. 检查硬件（8GB/12GB/16GB/24GB/48GB 显存）
   ↓
2. 选择模型（4B/7B/14B/32B）
   ↓
3. 安装 WSL2 + Ubuntu
   ↓
4. 配置 GPU 直通（nvidia-smi 验证）
   ↓
5. 创建 Python 虚拟环境
   ↓
6. 安装 vLLM
   ↓
7. 启动服务（根据硬件配置参数）
   ↓
8. 安装 Node.js + OpenClaw
   ↓
9. 配置 API 连接
   ↓
10. 调优参数（Context length, Temperature）
```

### 硬件配置速查表

| 显存 | 推荐模型 | gpu-memory-util | max-model-len | 预期速度 |
|------|---------|-----------------|---------------|---------|
| 8GB | Qwen2.5-4B-AWQ | 0.80 | 8192 | 150-200 t/s |
| 12GB | Qwen2.5-7B-AWQ | 0.85 | 16384 | 120-170 t/s |
| 16GB | Qwen2.5-7B-AWQ | 0.85 | 32768 | 120-170 t/s |
| 24GB | Qwen2.5-14B-AWQ | 0.90 | 32768 | 90-130 t/s |
| 48GB | Qwen2.5-32B-AWQ | 0.90 | 65536 | 60-90 t/s |

### 关键参数速查

| 参数 | 快速对话 | 代码生成 | 创意写作 | 文档分析 |
|------|---------|---------|---------|---------|
| Context length | 6000 | 10000 | 12000 | 15000 |
| Temperature | 0.7 | 0.3 | 1.0 | 0.5 |
| Max tokens | 1024 | 2048 | 4096 | 2048 |
| Top p | 0.9 | 0.8 | 0.95 | 0.9 |

### 常见问题速查

| 问题 | 快速解决 |
|------|---------|
| CUDA OOM | 降低 gpu-memory-utilization 或 max-model-len |
| 下载失败 | 使用 HF 镜像站或代理 |
| 连接失败 | 检查 vLLM 是否运行、端口是否监听 |
| 性能下降 | 降低 context length、添加摘要指令 |
| 工具调用失效 | 确认 --enable-auto-tool-choice 和 --tool-call-parser |

---

**文档版本**: 2.0
**最后更新**: 2026年3月20日
**适用版本**: vLLM 0.6.x, OpenClaw 2.x
