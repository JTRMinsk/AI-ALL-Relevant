# OpenClaw 网页远程访问配置指南

## 目标

通过浏览器从其他设备（局域网内）访问 OpenClaw Gateway 的 WebChat 界面，直接与 AI 对话。

## 主线思路

```
浏览器 → http://<IP>:18789 → Gateway → AI 模型
```

要让这个链路跑通，需要解决三道关卡：

1. **CORS（跨域）**：浏览器跨域访问被拦截
2. **设备身份（Device Identity）**：非安全上下文（HTTP）下浏览器无法创建 WebCrypto 设备身份
3. **Token 认证**：Gateway 要求提供 token 才能登录

按顺序逐一打通即可。

---

## 操作步骤

### 第一步：确认 Gateway 监听地址和端口

```bash
openclaw gateway status
```

关键信息：
- `bind=lan (0.0.0.0)` → 监听所有网卡，局域网可访问
- `port=18789` → 端口号

如果 bind 是 `loopback`，需要改为 `lan`：
```json
// ~/.openclaw/openclaw.json
"gateway": {
  "bind": "lan"
}
```

**坑 ⚠️**：如果 bind 是 `loopback`，外部设备根本无法访问，必须先改。

---

### 第二步：配置 CORS 允许的来源

浏览器从 `http://10.179.137.24:18789` 访问时，Gateway 需要明确允许这个来源。

配置位置：`~/.openclaw/openclaw.json`

```json
"gateway": {
  "controlUi": {
    "allowedOrigins": [
      "http://10.179.137.24:18789",
      "http://192.168.3.222:18789",
      "http://localhost:18789",
      "http://127.0.0.1:18789"
    ]
  }
}
```

**坑 ⚠️**：不要配到 `gateway.cors.origins`，正确的字段是 `gateway.controlUi.allowedOrigins`。配错了不会生效，日志里会明确提示正确字段名。

---

### 第三步：关闭设备身份认证（仅限远程 HTTP 访问）

浏览器要求 HTTPS 或 localhost 才能创建设备身份（WebCrypto API 限制）。

**安全方案（推荐）**：用 Tailscale Serve 获得 HTTPS，走 `https://<your-tailnet>.ts.net:18789/`

**跳过方案（内网用）**：在配置中加两行关闭设备认证：

```json
"gateway": {
  "controlUi": {
    "allowInsecureAuth": true,
    "dangerouslyDisableDeviceAuth": true
  }
}
```

**坑 ⚠️**：`allowInsecureAuth: true` 只对 localhost（`127.0.0.1` / `localhost`）的 HTTP 访问生效，远程 HTTP 访问会直接忽略。必须同时设 `dangerouslyDisableDeviceAuth: true` 才能让远程 HTTP 绕过设备身份检查。

---

### 第四步：登录时输入 Token

以上配置完成后重启，浏览器会显示登录界面，需要输入 Gateway Token。

**坑 ⚠️**：你可能以为改完配置就能直接进，但实际上 Gateway 还设了 `auth.mode: "token"`，需要登录认证。Token 不是 API Key，是你 Gateway 的访问密码。

Token 在这里：
```json
// ~/.openclaw/openclaw.json
"gateway": {
  "auth": {
    "mode": "token",
    "token": "你的token在这里"
  }
}
```

---

### 第五步：重启 Gateway

```bash
systemctl --user restart openclaw-gateway
```

**坑 ⚠️**：如果通过 WebChat 发命令重启，Gateway 会随进程一起死掉然后自动恢复，会话会短暂中断几秒。这是正常的。

---

## 踩坑总结

| 序号 | 现象 | 原因 | 解决 |
|------|------|------|------|
| 1 | `origin not allowed` | CORS origin 没配 | `gateway.controlUi.allowedOrigins` 加上浏览器来源 |
| 2 | 配了 `gateway.cors.origins` 不生效 | 字段名错误 | 正确字段是 `gateway.controlUi.allowedOrigins` |
| 3 | `control ui requires device identity` | 远程 HTTP 不是安全上下文 | 同时设 `allowInsecureAuth: true` + `dangerouslyDisableDeviceAuth: true` |
| 4 | 设了 `allowInsecureAuth` 仍报 identity 错误 | 该字段只对 localhost 生效 | 加 `dangerouslyDisableDeviceAuth: true` |
| 5 | 绕过 identity 后报 `token_missing` | Gateway 有 token 认证 | 在浏览器登录页面输入 token |

---

## 最终有效配置摘要

```json
"gateway": {
  "bind": "lan",
  "port": 18789,
  "auth": {
    "mode": "token",
    "token": "<你的token>"
  },
  "controlUi": {
    "allowInsecureAuth": true,
    "dangerouslyDisableDeviceAuth": true,
    "allowedOrigins": [
      "http://<你的IP>:18789"
    ]
  }
}
```

---

## 安全提醒

- `dangerouslyDisableDeviceAuth: true` 降低了安全性，仅在受信任的内网环境使用
- Token 等同于密码，不要泄露
- 长期使用建议配置 Tailscale Serve 走 HTTPS，避免关闭设备认证

---

> 生成时间：2026-05-23 | 环境：OpenClaw v2026.5.12
