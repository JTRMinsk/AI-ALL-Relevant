# Hermes Agent 配置 Google 邮箱和 Drive 完整指南

## 环境信息

- **OS**: Ubuntu 26.04 LTS, GNOME 桌面
- **代理**: mihomo (verge-mihomo), HTTP 代理监听 `127.0.0.1:7890`，系统服务自启
- **Google 账号**: 个人 Gmail，已开启高级保护 (Advanced Protection)
- **Hermes Agent**: 已安装，venv 位于 `~/.hermes/hermes-agent/venv/`

---

## 0. 背景知识

### 为什么不能直接用 IMAP 密码？

Google 邮箱有两条接入路径：

| 方式 | 适用 | 前提 | 能做什么 |
|------|------|------|---------|
| App Password (IMAP) | 只要邮箱 | 未开高级保护 | 收发邮件 |
| OAuth 2.0 | 邮箱 + Drive + Calendar 等全家桶 | 无限制 | 全套 Google API |

开了高级保护的账号无法创建 App Password——Google 会拦截。所以只能用 OAuth。

### OAuth 是什么？

三个角色：
```
用户 (浏览器) ──→ Google 授权服务器 ──→ Hermes (应用)
                     │
               "Hermes 要访问你的 Gmail 和 Drive，同意吗？"
```

Hermes 不接触密码。用户在自己浏览器里登录 Google、确认权限后，Google 给 Hermes 一个 token。token 可以长期刷新，过期不用重新授权。

---

## 1. Google Cloud Console 操作

> **总耗时**: 约 10 分钟，一次性操作。

### 1.1 创建项目

**网址**: https://console.cloud.google.com/projectselector2/home/dashboard

点击 `CREATE PROJECT`，项目名随意（例: `hermes-gmail`），Location 选 `No organization`。

**做什么**: Google 所有 API 都挂在"项目"下面。一个项目=一个应用身份。

**为什么**: 不建项目就无法启用 API、无法创建 OAuth 客户端。Google 的计费和权限都以项目为粒度，虽然我们用的是免费额度。

### 1.2 启用 API

**网址**: https://console.cloud.google.com/apis/library

逐一搜索并点 `ENABLE`：

| API 名称 | API 标识符 | 用途 |
|----------|-----------|------|
| Gmail API | `gmail.googleapis.com` | 读邮件、发邮件、管理标签 |
| Google Drive API | `drive.googleapis.com` | 上传/下载/分享文件 |
| Google Calendar API | `calendar-json.googleapis.com` | 创建/删除/查询日程 |
| Google Sheets API | `sheets.googleapis.com` | 读写电子表格 |
| Google Docs API | `docs.googleapis.com` | 读写在线文档 |
| People API | `people.googleapis.com` | 读联系人 |

**做什么**: 告诉 Google "这个项目要用这些服务"。

**为什么**: 不启用某个 API，后续即使拿到 token 调用也会报 `403: Access Not Configured`。**这是很常见的踩坑点**——以为有 token 就能调用，实际 API 还没开。

> **坑**: Google 要求至少启用一个 API 才能继续创建 Consent Screen。如果发现 Consent Screen 入口不可用，回来检查 API 是否已 enable。

---

### 1.3 配置 OAuth 同意屏幕 (Consent Screen)

**网址**: https://console.cloud.google.com/apis/credentials/consent

**做什么**:

1. User Type 选 **External**（必须——Internal 只适用于 Google Workspace 企业组织，个人账号没有 `Create` 按钮）
2. 填入以下信息:
   - App name: `Hermes Agent`（随意，但会显示在授权页面上）
   - User support email: 你的 Gmail
   - Developer contact information: 同上
3. Scopes 页面: **跳过**，直接 `Save and Continue`（scopes 会在 OAuth 请求 URL 中动态指定）
4. Test users: 点击 `Add Users`，输入你的 Gmail 地址

**为什么每步这样设**:

- **External**: 个人账号只能用 External。Internal 要求同一 Workspace 组织。
- **Scopes 跳过**: 因为 `--auth-url` 会在 URL 里带上 scope 参数，不需要在 Console 里预配。
- **Test users**: 未发布的 External 应用只有在测试用户列表里的账号才能授权。不加这步会看到 `403: Access blocked: Hermes Agent has not completed the Google verification process`。

> **坑**: 多语言界面下，`Audience` 和 `Consent Screen` 之间的导航容易迷路。入口是 `APIs & Services → Credentials → Consent Screen` 或直接访问 `/apis/credentials/consent`。

---

### 1.4 创建 OAuth 2.0 客户端

**网址**: https://console.cloud.google.com/apis/credentials

**做什么**:

1. `Create Credentials` → `OAuth client ID`
2. Application type: **Desktop app**
3. Name: 随意
4. 点击 Create，下载 JSON 文件

**输出**: 一个 JSON 文件，内容大致如下：
```json
{
  "installed": {
    "client_id": "XXXXXXXXXXXX-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com",
    "project_id": "hermes-gmail",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_secret": "GOCSPX-xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "redirect_uris": ["http://localhost"]
  }
}
```

**为什么**: `client_id` 和 `client_secret` 是 OAuth 协议的"身份证"。`client_id` 公开出现在 URL 里（标识应用），`client_secret` 只在服务端交换 token 时用，不应泄露。

> **坑**: 必须先完成 1.3 的 Consent Screen，否则 OAuth client ID 创建按钮是灰色的。

---

## 2. Hermes 端配置

> **总耗时**: 约 5 分钟。

### 2.1 注册客户端凭据

```bash
~/.hermes/hermes-agent/venv/bin/python \
  ~/.hermes/skills/productivity/google-workspace/scripts/setup.py \
  --client-secret /path/to/client_secret_xxx.json
```

**输出**: `OK: Client secret saved to ~/.hermes/google_client_secret.json`

**做什么**: 把下载的 JSON 文件复制到 Hermes 配置目录。后续不需要原始文件了。

**为什么**: setup.py 会验证文件格式——必须包含 `"installed"` 或 `"web"` 键。如果拿错文件（比如下了 Service Account 的 JSON），会直接报错:

```
ERROR: Not a Google OAuth client secret file (missing 'installed' key).
```

> **坑**: Ubuntu 26.04 上系统 `python3` 是 externally-managed（PEP 668），`pip install` 会报错:
> ```
> error: externally-managed-environment
> ```
> 必须用 Hermes 自带的 venv: `~/.hermes/hermes-agent/venv/bin/python`。venv 里已经预装了 `google-api-python-client`、`google-auth-oauthlib`、`google-auth-httplib2`。

---

### 2.2 生成授权链接

```bash
~/.hermes/hermes-agent/venv/bin/python \
  ~/.hermes/skills/productivity/google-workspace/scripts/setup.py \
  --auth-url
```

**输出**: 一个很长的 URL，约 600 字符：

```
https://accounts.google.com/o/oauth2/auth?response_type=code
  &client_id=XXXXXXXXXXXX.apps.googleusercontent.com
  &redirect_uri=http://localhost:1
  &scope=https://www.googleapis.com/auth/gmail.readonly
    https://www.googleapis.com/auth/gmail.send
    https://www.googleapis.com/auth/gmail.modify
    https://www.googleapis.com/auth/drive
    https://www.googleapis.com/auth/calendar
    https://www.googleapis.com/auth/contacts.readonly
    https://www.googleapis.com/auth/spreadsheets
    https://www.googleapis.com/auth/documents
  &state=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  &code_challenge=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  &code_challenge_method=S256
  &access_type=offline
  &prompt=consent
```

**做什么**: 生成 OAuth 2.0 PKCE 授权链接。

**各参数意义**:

| 参数 | 值 | 意义 |
|------|-----|------|
| `response_type=code` | code | 授权码模式（标准 OAuth） |
| `client_id` | xxx | 应用标识，就是刚才 JSON 里的 |
| `redirect_uri` | `http://localhost:1` | 授权后跳转地址。用 `:1` 端口是因为 Google 弃用了传统 `urn:ietf:wg:oauth:2.0:oob` 方式 |
| `scope` | gmail.send calendar drive... | 请求的权限列表。用户在授权页面能看到每一项 |
| `code_challenge` | base64 哈希 | PKCE 安全机制：防止授权码被中间人劫持 |
| `code_challenge_method` | S256 | SHA-256 哈希算法 |
| `access_type=offline` | offline | **关键**：要求 Google 返回 `refresh_token`，允许长期刷新。不设这个过期后就废了 |
| `prompt=consent` | consent | 强制显示授权页面，即使用户之前授权过。保证拿到最新的 scope 授权 |

> **坑**: `setup.py` 不支持 `--services` 参数（技能文档写的 `--services email,drive` 是错的）。脚本默认一次性请求所有 scope。暂时没有按需选择 scope 的功能。

---

### 2.3 浏览器授权

**做什么**:

1. 打开 2.2 生成的 URL
2. 选择 Google 账号登录
3. 看到权限列表（Gmail 读写、Drive 读写等），点 `Allow`
4. 浏览器跳转到 `http://localhost:1/?code=...&state=...&scope=...`
5. 浏览器显示"无法连接此网站"——**正常，不用管**
6. 复制浏览器地址栏里的**完整 URL**（不是页面内容）

**典型 URL**:
```
http://localhost:1/?state=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  &iss=https://accounts.google.com
  &code=4/0AeoWuMxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  &scope=https://www.googleapis.com/auth/gmail.readonly
    https://www.googleapis.com/auth/gmail.send
    ...
```

**为什么跳转 `localhost:1`**: Google OAuth 的老 `urn:ietf:wg:oauth:2.0:oob`（Out-Of-Band）方式已被弃用。现在用 `localhost:1` 是标准做法——用户复制地址栏 URL，脚本从中提取 `code` 参数。端口 `:1` 是为了确保必然失败，迫使用户复制 URL。

---

### 2.4 交换 Token

```bash
~/.hermes/hermes-agent/venv/bin/python \
  ~/.hermes/skills/productivity/google-workspace/scripts/setup.py \
  --auth-code "http://localhost:1/?code=4/0AeoWuM...&state=..."
```

**输出**:
```
OK: Authenticated. Token saved to /home/<user>/.hermes/google_token.json
Profile-scoped token location: ~/.hermes/google_token.json
```

**做什么**: 用授权码向 Google 换取 `access_token` + `refresh_token`，并存到 `~/.hermes/google_token.json`。

**token 文件内容** (脱敏):
```json
{
  "type": "authorized_user",
  "client_id": "XXXXXXXXXXXX.apps.googleusercontent.com",
  "client_secret": "GOCSPX-xxxx",
  "refresh_token": "1//xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "access_token": "ya29.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "token_uri": "https://oauth2.googleapis.com/token",
  "scopes": [
    "https://www.googleapis.com/auth/gmail.readonly",
    "https://www.googleapis.com/auth/gmail.send",
    ...
  ],
  "token_expiry": "2026-05-30T14:22:00+08:00"
}
```

**关键字段**:
- `access_token`: 当前有效 token，每次 API 调用都带着（Authorization: Bearer xxx）。一小时后过期。
- `refresh_token`: 永久刷新凭证。access_token 过期后脚本自动用 refresh_token 换新的。**这个丢了就得重新授权**。
- `scopes`: 实际授权的权限范围。可能比请求的少（如果用户手动取消勾选某些权限）。

> **坑 (最耗时)**: **在中国大陆直接访问 `oauth2.googleapis.com` 会超时**。mihomo 代理默认只配了 GNOME 系统代理（GUI 应用能用），终端 Python 脚本的网络库 (`httplib2`) 不认 GNOME 设置。详见第 3 节。

---

### 2.5 验证

```bash
~/.hermes/hermes-agent/venv/bin/python \
  ~/.hermes/skills/productivity/google-workspace/scripts/setup.py \
  --check
```

**期望输出**: `AUTHENTICATED: Token valid at ~/.hermes/google_token.json`

**三种可能的输出**:

| 输出 | 含义 | 处理 |
|------|------|------|
| `AUTHENTICATED` | ✅ 一切正常 | 开始用 |
| `AUTHENTICATED (partial)` | Token 有效但少了某些 scope | revoke 后重新授权 |
| `NOT_AUTHENTICATED` | 没拿到 token | 走一遍 2.1-2.4 |
| `REFRESH_FAILED` | Token 过期且刷新失败 | 重新走 2.2-2.4 |
| `TOKEN_REVOKED` | 用户手动撤销了授权 | 重新走 2.2-2.4 |

---

## 3. 代理问题详解 (最重要的一节)

### 3.1 现象

在已配置 mihomo 代理的中国大陆网络环境下:

- `curl -x http://127.0.0.1:7890 https://oauth2.googleapis.com/token` → ✅ 工作正常
- Python 用 `requests` 库 + `HTTPS_PROXY` 环境变量 → ✅ 工作正常
- Python 用 `googleapiclient` (底层 `httplib2`) → ❌ 超时 60 秒

### 3.2 根因分析

调用链路:
```
google_api.py
  → googleapiclient.discovery.build()
    → google_auth_httplib2.AuthorizedHttp()
      → httplib2.Http._conn_request()
        → socks.PROXY_TYPE_HTTP (type=2)
          → SOCKS5 握手协商
            → mihomo 端口 7890 不响应 SOCKS5
              → 超时
```

`httplib2` 在检测到 `HTTPS_PROXY` 环境变量后，默认用 `SOCKS5` 方式去协商 HTTP 代理。但 mihomo 的 7890 端口是纯 HTTP 代理，不回应 SOCKS5 握手，导致 socket connect 阶段超时。

对比 `requests` 库，它用 `urllib3`，直接通过 HTTP CONNECT 隧道或 HTTP 转发方式使用代理，和 mihomo 兼容。

### 3.3 解决方案

修改 `google_api.py` 的 `build_service()` 函数，显式指定代理类型为 `PROXY_TYPE_HTTP_NO_TUNNEL` (type=3):

```python
# 原来
def build_service(api, version):
    from googleapiclient.discovery import build
    return build(api, version, credentials=get_credentials())

# 改后
def build_service(api, version):
    from googleapiclient.discovery import build
    from google_auth_httplib2 import AuthorizedHttp
    import httplib2

    creds = get_credentials()
    proxy_info = httplib2.ProxyInfo(
        proxy_type=3,           # PROXY_TYPE_HTTP_NO_TUNNEL
        proxy_host="127.0.0.1",
        proxy_port=7890,
    )
    http = httplib2.Http(proxy_info=proxy_info, timeout=60)
    authed_http = AuthorizedHttp(creds, http=http)
    return build(api, version, http=authed_http)
```

**为什么这样改**:
- `proxy_type=3` (HTTP_NO_TUNNEL): httplib2 把 HTTPS 请求降级为 HTTP 请求，通过 HTTP 代理转发。代理端 (mihomo) 处理 HTTPS 的部分。
- `proxy_type=2` (HTTP): httplib2 用 CONNECT 隧道，但底层走 SOCKS5 握手 → 和 mihomo 不兼容。
- 之所以不用环境变量方案（`HTTPS_PROXY`）：避免全局污染，只影响这一个脚本。

### 3.4 为什么不设系统级代理？

- **GNOME 系统代理** (`gsettings org.gnome.system.proxy`) → 只对 GTK/GNOME 应用生效（浏览器、Nautilus 等），终端程序不读
- **环境变量** (`HTTPS_PROXY` / `HTTP_PROXY`) → 终端程序会认，但需要加到 `~/.profile` 影响所有 session，太粗暴
- **mihomo TUN 模式** → 全流量劫持，过于全局

**最终方案**: 在 `google_api.py` 中硬编码代理。只影响 Google API 调用，不影响其他程序。

---

## 4. 最终架构

```
你的命令 (发邮件/搜文件/查日程)
    │
    ▼
google_api.py ←── 只在 Hermes 内调用，外面也可以手动
    │
    ├── setup.py       ← OAuth 授权 (一次性)
    ├── setup.py --check ← 验证 token 有效
    └── google_api.py gmail search / drive upload / ...
            │
            │   httplib2 (proxy_type=HTTP_NO_TUNNEL)
            │   → 127.0.0.1:7890 (mihomo)
            │   → Google API
            │
            ▼
         JSON 输出
```

### 文件位置

| 文件 | 路径 | 说明 |
|------|------|------|
| OAuth 客户端凭据 | `~/.hermes/google_client_secret.json` | 从 Console 下载，setup.py 复制到此 |
| OAuth Token | `~/.hermes/google_token.json` | 自动刷新，包含 access_token + refresh_token |
| API 脚本 | `~/.hermes/skills/productivity/google-workspace/scripts/google_api.py` | 已 patch 代理 |
| 授权脚本 | `~/.hermes/skills/productivity/google-workspace/scripts/setup.py` | OAuth 流程 |
| Python | `~/.hermes/hermes-agent/venv/bin/python` | 必须用 venv，不能用系统 python3 |

---

## 5. 常用命令速查

```bash
# 定义快捷变量
PY=~/.hermes/hermes-agent/venv/bin/python
GAPI=~/.hermes/skills/productivity/google-workspace/scripts/google_api.py

# ---------- Gmail ----------

# 查收件箱 (前 10 封)
$PY $GAPI gmail search "in:inbox" --max 10

# 查未读邮件
$PY $GAPI gmail search "is:unread" --max 5

# 按发件人搜索
$PY $GAPI gmail search "from:boss@company.com newer_than:7d"

# 读邮件全文
$PY $GAPI gmail get MESSAGE_ID

# 发邮件
$PY $GAPI gmail send \
  --to user@example.com \
  --subject "Test" \
  --body "Hello from Hermes"

# 回复邮件 (自动保留线程)
$PY $GAPI gmail reply MESSAGE_ID --body "Thanks"

# 查标签
$PY $GAPI gmail labels

# ---------- Drive ----------

# 搜索文件
$PY $GAPI drive search "report" --max 10

# 上传
$PY $GAPI drive upload /path/to/file.pdf

# 下载
$PY $GAPI drive download FILE_ID

# 创建文件夹
$PY $GAPI drive create-folder "Projects"

# 查看文件详情
$PY $GAPI drive get FILE_ID
```

---

## 6. 踩坑记录

| # | 坑 | 现象 | 根因 | 解决 |
|---|-----|------|------|------|
| 1 | 系统 Python | `pip install` 报 `externally-managed-environment` | Ubuntu 26.04 PEP 668 | 用 Hermes venv 的 python |
| 2 | httplib2 代理超时 | `google_api.py` 卡住 60s 无响应 | httplib2 SOCKS5 握手不兼容 mihomo HTTP 代理 | 改 `build_service()` 用 `proxy_type=3` |
| 3 | 403: 应用未验证 | 授权页报 "尚未完成验证" | 未添加测试用户 | `Audience → Test users` 加自己的 Gmail |
| 4 | Consent Screen 入口找不到 | 英文界面迷路 | UI 路径不对 | 直接访问 `/apis/credentials/consent` |
| 5 | `setup.py` 参数不存在 | `--services email,drive` 报错 | 技能文档过时 | 不带 `--services`，默认全 scope |
| 6 | 高级保护 | App Password 不可用 | 账号开了 Advanced Protection | 必须走 OAuth（不受影响） |
| 7 | GNOME 代理对终端无效 | `HTTPS_PROXY` 为空 | GNOME 代理只对 GTK 应用生效 | 脚本内硬编码代理 |
