# OpenClaw 集成指南

## 本地 App → Gateway

### Operator 模式（完整控制）

```
Your App                      OpenClaw Gateway
├─ WebSocket Client  ◄──►     ws://127.0.0.1:18789
└─ State Management           ├─ Auth
                              ├─ Session Mgmt
                              └─ Agent Runtime
```

**连接：**
```javascript
{
  role: 'operator',
  scopes: ['operator.read', 'operator.write'],
  auth: { token: 'your-gateway-token' },
  device: { id, publicKey, signature, signedAt, nonce }
}
```

---

## 移动端 → Gateway

### 方案 A：中转架构（推荐）

```
Mobile App          Local App           OpenClaw Gateway
├─ Camera/Location  ◄──► WS Server  ◄──► WS Client
├─ Filesystem           HTTP API        (Operator)
└─ Push Notif.          (Operator)
```

本地 App 作为代理层，移动端通过 HTTP/WebSocket 连接本地 App。

**适用：**照片备份、文件同步、主动数据推送

**优点：**移动端无需 Token，本地可预处理

---

### 方案 B：Node 模式（能力暴露）

```
Mobile App (Node)             OpenClaw Gateway
├─ Camera                     ├─ node.invoke camera.snap
├─ Location        ◄────────► ├─ node.invoke location.get
└─ Screen Record              └─ (被动响应)
```

移动端作为 Node 被动响应 Gateway 请求。

**适用：**相机、位置、屏幕录制

**限制：**无法主动操作 Gateway

---

## 外网连接方案

移动端外网访问 Gateway 需配置 TLS 反向代理。

### 反向代理配置

使用 nginx 配置 HTTPS：

```nginx
server {
    listen 443 ssl;
    server_name gateway.example.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
}
```

**移动端连接：**
```javascript
const ws = new WebSocket('wss://gateway.example.com:18789');
```

### 其他方案

| 方案 | 说明 |
|------|------|
| **frp** | 内网穿透工具，支持 TLS |
| **ngrok** | 临时公网隧道，自动生成 HTTPS |
| **cloudflare tunnel** | 免费隧道，自带 HTTPS |

---

## 协议要求

| 场景 | 协议 | 说明 |
|------|------|------|
| 本地回环 | `ws://` | 127.0.0.1 / localhost |
| 私有 LAN | `ws://` | 192.168.x.x / 10.x.x.x |
| 公网/外网 | `wss://` | 强制加密 |

> **移动端外网必须 `wss://`**

---

## 设备配对

```bash
openclaw devices list          # 查看待审批
openclaw devices approve <id>  # 审批设备
```

本地回环自动审批，远程需显式审批。

---

## 参考

- `docs/gateway/protocol.md` — 协议详情
- `docs/gateway/remote.md` — 远程访问
- `docs/gateway/pairing.md` — 设备配对
