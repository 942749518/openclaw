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
| 私有 LAN | `ws://` | 192.168.x.x / 10.x.x.x (RFC 1918, link-local, mDNS `.local`) |
| Tailscale | `wss://` | 强制加密 |
| 公网/外网 | `wss://` | 强制加密 |

> **移动端外网必须 `wss://`**
>
> 私有 LAN 的 `ws://` 明文连接仅用于缺乏 PKI 身份的局域网，不视为安全风险。

---

## 安全机制

### SSRF 防护

Gateway 对所有出站 HTTP 请求实施 SSRF 防护：

- **RFC2544 私网策略** — `web_fetch` 工具支持 `rfc2544` opt-in，控制私网地址访问
- **浏览器 SSRF 加固** — 非导航文档跳转的 SSRF 重定向守卫
- **浏览器 proxy 配置** — 硬化 proxy profile 变更守卫
- **DNS Pinning** — 信任环境代理调度前跳过 DNS pinning

### 执行审批

代理工具调用 (如 `bash`、`node`) 受执行审批子系统管控：

- **Allow-from 白名单** — 限制可执行的来源路径
- **TOCTOU 防护** — 脚本预检使用原子 pinned-fd open 替代 check-then-read
- **远程节点执行** — 与未信任处理对齐的系统消息
- **Startup Replay** — 网关重启后自动重放挂起的审批请求

### 环境变量安全

- **Host-exec 环境过滤** — 扩展黑名单覆盖 Java、Rust、Cargo 工具链
- **Dotenv 隔离** — 阻断工作区运行时环境变量注入
- **凭证脱敏** — `browser.cdpUrl` 配置路径中的凭证自动脱敏

### 会话安全

- **密钥轮换失效** — 共享令牌/密码 WebSocket 会话在密钥轮换后自动失效
- **工具使用中止保护** — 防止 compaction 后的无限循环调用

---

## 设备配对

```bash
openclaw devices list          # 查看待审批
openclaw devices approve <id>  # 审批设备
```

本地回环自动审批，远程需显式审批。

### Android QR 配对

Android 端支持 QR 码扫描配对流程：

- 优先使用存储的设备认证
- Bootstrap 认证作为 QR 配对回退
- 新 setup code 自动重置认证
- 配对审批后自动恢复

### 节点重连

- 命令升级（command upgrades）需重新配对
- 网关在节点断开后保留会话状态

---

## 多账号与线程路由

### WhatsApp 多账号

- 入站状态按账号隔离
- 重连后自动投递排队消息
- 回复跨重连保留

### 线程路由

Gateway 在投递上下文中保留 `threadId`，确保跨通道线程回复准确：

- Slack — `thread_ts` 保留
- Telegram — `replyToMessageId` 保留
- Mattermost — root post ID 保留
- MS Teams — `replyToId` 隔离频道线程会话

---

## 参考

- `docs/gateway/protocol.md` — 协议详情
- `docs/gateway/remote.md` — 远程访问
- `docs/gateway/pairing.md` — 设备配对
