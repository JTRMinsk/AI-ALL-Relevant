# OpenClaw 配置复盘：让 AI Agent 触达外网

## 背景

OpenClaw 是一个可自部署的 AI Agent 平台，内置 `web_search` 和 `web_fetch` 两个工具，让 Agent 能够搜索互联网和抓取网页内容。

默认情况下，`web_search` 通过本地 SearXNG 代理转发到 Bing 搜索引擎。在国内网络环境中，Bing 国际版通常不可达，搜索结果局限在中文国内源，Agent 无法获取国际资讯。`web_fetch` 同样受限于网络可达性，国外网站返回超时而非内容。

本文目标是让这两个工具能稳定获取国际互联网内容，同时不影响国内 API 的直连性能。

---

## 整体架构

```
┌─────────────────┐
│  web_search 请求  │
└────────┬────────┘
         ↓
┌─────────────────┐
│ OpenClaw Gateway │
└────────┬────────┘
         ↓
┌─────────────────┐
│ SearXNG Proxy    │  ← 本地 8888 端口，Python HTTP 服务
│ (Bing 搜索代理)  │
└────────┬────────┘
         ↓  HTTP_PROXY 环境变量
┌─────────────────┐
│ Mihomo           │  ← 本地 7890 端口，Clash Meta 内核
│ (代理客户端)     │
└────────┬────────┘
         ↓  加密隧道
┌─────────────────┐
│ 海外节点         │
└────────┬────────┘
         ↓
   Bing 国际版 / Google / 外网站点
```

核心思路：
- **Mihomo**（Clash Meta 内核）作为本地代理客户端，连接海外节点，所有需要翻墙的流量走 7890 端口的 SOCKS/HTTP 代理
- **SearXNG Proxy** 是一个 Python 脚本，将 web_search 请求转发到 Bing 并解析 HTML 搜索结果，通过 `HTTP_PROXY` 环境变量走代理访问 Bing 国际版
- **NO_PROXY** 环境变量确保国内 API（DeepSeek、阿里云通义千问等）直连，不绕代理，保证低延迟

---

## 实施过程

### 第一步：选择代理软件

**尝试 Clash Verge Rev（GUI 版）→ 放弃**

Clash Verge Rev 提供了一个图形化界面管理代理，安装后订阅了机场节点。但问题在于：它的配置文件不是纯文本，而是存储在嵌入式 SQLite 数据库中。手动编辑 `config.yaml` 后重启，改动会被 GUI 从数据库中重新生成的文件覆盖。这意味着无法程序化地管理配置、添加自定义规则或自动切换节点。

**转向 Mihomo CLI**

Mihomo 是 Clash Meta 内核的命令行版本，直接读取 YAML 配置文件，不会有 GUI 覆盖的问题。安装方式：

```bash
# 安装 mihomo（Clash Meta 内核）
# 不同发行版路径可能不同，常见位置如 /usr/bin/verge-mihomo
# 将 GUI 版已下载的内核二进制复用即可
cp /path/to/verge-mihomo /usr/bin/verge-mihomo
```

创建配置目录：

```
~/.config/mihomo/
├── config.yaml       # 主配置：端口、节点列表、代理规则
├── Country.mmdb      # GeoIP 数据库，用于按国家分流
├── geoip.dat         # IP 规则数据
└── geosite.dat       # 域名分类数据
```

配置文件核心内容：

```yaml
mixed-port: 7890          # HTTP/SOCKS 混合代理端口
allow-lan: true           # 允许局域网设备使用
external-controller: 127.0.0.1:9090  # RESTful API 控制端口
secret: "自定义密码"       # API 鉴权密码
mode: rule                # 规则模式

proxies:                  # 节点列表（从机场订阅获取）
  - name: "节点名称"
    type: vmess
    server: "服务器地址"
    port: 端口
    uuid: "UUID"
    cipher: auto

proxy-groups:             # 代理组（自动选择/手动选择）
  - name: Proxy
    type: select
    proxies: ["节点1", "节点2", ...]

rules:                    # 分流规则
  - DOMAIN-SUFFIX,google.com,Proxy
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

启动 Mihomo：

```bash
nohup /usr/bin/verge-mihomo -d ~/.config/mihomo -f ~/.config/mihomo/config.yaml > /tmp/mihomo.log 2>&1 &
```

启动后，7890 端口监听 HTTP/SOCKS 代理，9090 端口暴露 RESTful API 可用于运行时切换节点、查看状态。

#### 通过 API 管理节点

Mihomo 提供了完整的 RESTful API，常用的操作：

**查看所有节点和代理组：**
```bash
curl -s "http://127.0.0.1:9090/proxies" \
  -H "Authorization: Bearer <secret>"
```

**切换当前使用的节点：**
```bash
curl -X PUT "http://127.0.0.1:9090/proxies/Proxy" \
  -H "Authorization: Bearer <secret>" \
  -H "Content-Type: application/json" \
  -d '{"name": "目标节点名称"}'
```

**测试延迟：**
```bash
curl -s "http://127.0.0.1:9090/proxies/<节点名>/delay?timeout=5000&url=http://www.gstatic.com/generate_204" \
  -H "Authorization: Bearer <secret>"
```

---

### 第二步：配置机场订阅

有了 Mihomo 后，需要从机场获取节点列表。

**订阅链接的处理**

机场通常提供一个订阅 URL，访问后返回包含所有节点信息的 YAML/Clash 配置。由于订阅 URL 可能不稳定（域名被墙、IP 被封等），配置了三个备用订阅源：一个 HTTPS 域名、一个 HTTPS IP（自定义端口）、一个 HTTP IP（降级兜底）。

**自动更新脚本**

写了一个 Bash 脚本 `update-sub.sh`，放在 `~/.config/mihomo/` 下，逻辑如下：

```bash
#!/bin/bash
# 三个订阅 URL 依次尝试
SUB_URLS=(
  "https://订阅域名/link/TOKEN?clash=1"      # 首选：HTTPS 域名
  "https://IP:21313/uuid/TOKEN?clash=1"     # 备选：HTTPS IP
  "http://IP:21312/uuid/TOKEN?clash=1"      # 兜底：HTTP IP
)

for url in "${SUB_URLS[@]}"; do
  # curl 下载，-k 忽略证书（自签或过期）
  if curl -s --max-time 10 -o /tmp/sub.yaml -k "$url"; then
    # 用 Python PyYAML 验证有效性
    # 检查 proxies 字段存在且非空
    # 验证通过 → 保存为 rabbitpro-latest.yaml，退出循环
    # 验证失败 → 继续尝试下一个 URL
  fi
done

# 合并配置：将下载的节点和规则更新到 config.yaml
# 保留本地自定义的端口、密钥等设置
python3 << 'EOF'
import yaml

# 读取新订阅
with open('rabbitpro-latest.yaml') as f:
    latest = yaml.safe_load(f)

# 读取当前配置
with open('config.yaml') as f:
    config = yaml.safe_load(f)

# 只更新节点、代理组和规则，保留本地设置
config['proxies'] = latest['proxies']
config['proxy-groups'] = latest['proxy-groups']
config['rules'] = latest['rules']

with open('config.yaml', 'w') as f:
    yaml.dump(config, f)
EOF

# 重启 Mihomo 使新配置生效
kill $(cat mihomo.pid)
nohup /usr/bin/verge-mihomo -d ~/.config/mihomo -f config.yaml &
```

使用方法：
```bash
bash ~/.config/mihomo/update-sub.sh
```

---

### 第三步：让 SearXNG 代理走代理

OpenClaw 的 web_search 实际上调用的是本地的一个 SearXNG 兼容代理（`searxng-proxy.py`），它本质上是一个 Python HTTP 服务，接收 `/search?q=...&format=json` 请求，转发到 Bing 并解析 HTML 返回结构化结果。

问题：这个代理内部使用 Python 标准库 `urllib.request.urlopen()` 发 HTTP 请求，它不会自动使用系统代理。即使系统设了 `HTTP_PROXY` 环境变量，`urlopen()` 也需要显式配置 `ProxyHandler`。

#### 改动 1：启动时读取代理环境变量

```python
import os
from urllib.request import ProxyHandler, build_opener, install_opener

# 读取 HTTP_PROXY（支持大小写）
_proxy_url = os.environ.get("HTTP_PROXY") or os.environ.get("http_proxy")
if _proxy_url:
    handler = ProxyHandler({"https": _proxy_url, "http": _proxy_url})
    install_opener(build_opener(handler))
```

这样脚本启动时如果环境中有 `HTTP_PROXY=http://127.0.0.1:7890`，就会自动通过 Mihomo 访问 Bing。

#### 改动 2：代理失败时 fallback 直连

这是最关键的改动。如果 Mihomo 挂了或者节点过期，之前的代码会直接抛异常导致 web_search 完全不可用。改为先尝试代理，失败后自动降级直连：

```python
def search_bing(query, count=10):
    # ... 构造请求 ...
    try:
        # 先走代理
        resp = urlopen(req, timeout=10)
        raw = resp.read().decode("utf-8", errors="replace")
    except Exception as e:
        if _proxy_url:
            # 代理不通，fallback 直连
            print(f"Proxy failed ({e}), trying direct...")
            direct_opener = build_opener()  # 不带代理的 opener
            resp = direct_opener.open(req, timeout=10)
            raw = resp.read().decode("utf-8", errors="replace")
        else:
            raise  # 没设代理就直接报错
```

这样即使 Mihomo 死了，web_search 至少能直连 Bing 中国版搜国内内容，不会完全中断。

#### 改动 3：适配 Bing 新版 HTML

Bing 国际版在 2026 年更新了搜索结果页的 HTML 结构。原本的 `b_algo` class 使用方式变了，正则表达式需要调整：

```python
# Bing 新版 HTML 示例：
# <li class="b_algo" data-id iid=SERP.12345>
#   <h2><a href="https://...">标题</a></h2>
#   <p>摘要...</p>
# </li>

# 解析代码
blocks = re.findall(r'<li class="b_algo"[^>]*>(.*?)</li>', raw, re.DOTALL)
for block in blocks:
    # 提取链接
    link_match = re.search(r'<a[^>]*href="(https?://[^"]+)"', block)
    # 提取标题
    title_match = re.search(r'<h2[^>]*>(.*?)</h2>', block, re.DOTALL)
    # 提取摘要
    snippet_match = re.search(r'<p[^>]*>(.*?)</p>', block, re.DOTALL)
    # 清理 HTML 标签后返回
```

#### 启动方式

SearXNG 代理需要带 `HTTP_PROXY` 环境变量启动：

```bash
HTTP_PROXY=http://127.0.0.1:7890 \
  nohup python3 ~/.openclaw/tools/searxng-proxy.py > /tmp/searxng-proxy.log 2>&1 &
```

---

### 第四步：让 Gateway 的 web_fetch 走代理（可选）

`web_fetch` 是 OpenClaw Gateway 内部实现的工具，使用 Node.js 的 undici HTTP 客户端。要让它走代理，需要给 Gateway 进程注入代理环境变量。

通过 systemd user service 的 drop-in 配置实现：

```
~/.config/systemd/user/openclaw-gateway.service.d/proxy.conf

[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="http_proxy=http://127.0.0.1:7890"
Environment="https_proxy=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,api.deepseek.com,..."
Environment="no_proxy=localhost,127.0.0.1,api.deepseek.com,..."
```

**注意 🔴 环境变量大小写**：Node.js 的 undici 库只识别小写的 `http_proxy` 和 `https_proxy`，而许多标准工具识别大写的 `HTTP_PROXY` 和 `HTTPS_PROXY`。所以两边都要设，一个都不能少。

然后重载 systemd 并重启 Gateway：
```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

---

## 踩坑记录

### 坑 1：Clash Verge Rev 配置被覆盖

**现象**：手动编辑 `config.yaml` 后重启，改动消失。

**根因**：Clash Verge Rev 使用 SQLite 存储配置，GUI 启动时从数据库读取并写入 YAML 文件，覆盖手动编辑的内容。

**解决**：放弃 GUI，改用 Mihomo CLI，直接操作 YAML。

---

### 坑 2：Mihomo 进程反复挂掉

**现象**：代理突然不可用，7890 端口没在监听。

**根因**：Mihomo 用 `nohup` 启动，作为 Gateway 的子进程。当 Gateway 通过 `systemctl restart` 重启时，systemd 可能会清理整个进程树，nohup 的子进程随之被杀。

**解决方案**（可选）：
- 将 Mihomo 配成独立的 systemd user service，彻底与 Gateway 解耦
- 或者在每次 Gateway 重启后手动检查并重启 Mihomo

---

### 坑 3：Node.js 不认大写的 HTTPS_PROXY

**现象**：systemd drop-in 配置了 `HTTPS_PROXY`，但 web_fetch 仍然直连，`httpbin.org/ip` 返回真实 IP 而非代理 IP。

**根因**：Node.js 内置的 undici HTTP 客户端只读取小写的 `http_proxy` 和 `https_proxy` 环境变量。大多数 Linux 工具和 Python 的 `requests`/`urllib` 识别大写，但 undici 不兼容。

**解决**：在 systemd drop-in 中同时设大小写两套环境变量。

---

### 坑 4：Bing 国际版 HTML 结构更新

**现象**：web_search 能搜但返回 0 个结果。

**根因**：Bing 在 2026 年更新了搜索结果页 HTML，`<li class="b_algo">` 变成了 `<li class="b_algo" data-id iid=SERP.xxxx>`（属性之间没有空格闭合）。

**解决**：调整正则匹配模式，兼容新旧两种格式。

---

### 坑 5：代理挂了导致 web_search 完全不可用

**现象**：节点过期后，web_search 直接返回错误而非搜索结果。

**根因**：SearXNG 代理脚本在代理失败时直接向上抛异常，没有降级方案。

**解决**：加入 fallback 逻辑：代理失败 → 创建一个不带代理的 opener → 直连 Bing 中国版 → 至少能搜国内内容。

---

## 最终配置清单

```
~/.config/mihomo/
  config.yaml              # Mihomo 核心配置：端口、节点、代理组、规则
  Country.mmdb             # GeoIP 数据库
  geoip.dat                # IP 分流数据
  geosite.dat              # 域名分类数据
  update-sub.sh            # 订阅更新 + 切换脚本

~/.openclaw/tools/
  searxng-proxy.py         # SearXNG 兼容代理（已改动：代理支持 + fallback）
```

### 运行时进程

两个后台进程需要保持运行：

| 进程 | 启动命令 | 端口 |
|------|------|------|
| Mihomo | `nohup verge-mihomo -d ~/.config/mihomo -f config.yaml &` | 7890 (代理) / 9090 (API) |
| SearXNG Proxy | `HTTP_PROXY=http://127.0.0.1:7890 nohup python3 searxng-proxy.py &` | 8888 |

---

## 效果

| 场景 | web_search | web_fetch |
|------|:--|:--|
| **代理正常** | 搜 Bing 国际版，返回 The Verge、TechCrunch、CBC News 等英文外网源 | 能抓被墙的外网站点 |
| **代理挂了** | 自动降级直连 Bing 中国版，搜国内内容 | 外网站超时，国内站点正常 |
| **国内 API** | 不受任何影响，直连保持低延迟 | — |

---

## 已知局限

1. **无开机自启**：Mihomo 和 SearXNG 代理都是 `nohup` 手动启动，系统重启后需重新拉起
2. **节点过期需手动处理**：未配定时任务自动更新订阅，节点失效后需要手动运行 `update-sub.sh`
3. **web_fetch 走不走代理取决于 Gateway 是否配了 proxy.conf**：这是一个可选配置，配了能抓外网但会增加 Gateway 复杂度
