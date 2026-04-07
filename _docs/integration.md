# OpenClaw 集成指南

## 本地 App → Gateway（Operator 模式）

适用于桌面应用或同设备运行的客户端，拥有完整控制权限。

### 架构

```
Your App (Operator)         OpenClaw Gateway
├─ WebSocket Client  ◄──►   ws://127.0.0.1:18789
└─ State Management         ├─ Auth
                            ├─ Session Mgmt
                            ├─ Channel Registry
                            └─ Agent Runtime
```

### 连接流程

```javascript
const ws = new WebSocket('ws://127.0.0.1:18789');

// 1. 接收 challenge
// 2. 发送 connect
{
  type: 'req',
  method: 'connect',
  params: {
    role: 'operator',
    scopes: ['operator.read', 'operator.write'],
    auth: { token: 'your-gateway-token' },
    device: { id, publicKey, signature, signedAt, nonce }
  }
}
```

### 首次配对

```bash
openclaw devices list
openclaw devices approve <request-id>
```

> 本地回环自动审批，远程需显式审批。

---

## 移动端 → Gateway（Node 模式）

适用于 iOS/Android App，作为节点暴露设备能力（相机、位置、通知等）。

### 架构

```
Mobile App (Node)           OpenClaw Gateway
├─ WebSocket Client  ◄──►   ws://gateway-host:18789
├─ Camera/Location            (LAN / Tailscale / Public)
└─ Push Notification
```

### 网络方案对比

| 场景 | 配置 | 说明 |
|------|------|------|
| **同 Wi-Fi** | `gateway.bind: lan` | 最简单，Bonjour 自动发现 |
| **跨网络** | `gateway.tailscale.mode: serve` | 最安全，推荐 |
| **公网访问** | `plugins.entries.device-pair.config.publicUrl` | 需反向代理 |

### 接入步骤

**1. Gateway 配置（选择一种）**

```bash
# 方案 A：同 Wi-Fi
openclaw config set gateway.bind lan

# 方案 B：Tailscale（推荐）
openclaw config set gateway.tailscale.mode serve

# 方案 C：公网 URL
openclaw config set plugins.entries.device-pair.config.publicUrl https://your-domain.com
```

**2. 生成设置码**

```bash
openclaw qr --json
```

**3. 移动端连接**

- 扫描二维码，或
- 手动输入 Gateway 地址（host:port）

**4. 设备配对**

```bash
openclaw devices list          # 查看待审批设备
openclaw devices approve <id>  # 审批
```

**5. 连接参数（Node 角色）**

```javascript
{
  type: 'req',
  method: 'connect',
  params: {
    role: 'node',
    scopes: [],
    caps: ['camera', 'location', 'notification'],
    commands: ['camera.snap', 'location.get'],
    device: { id, publicKey, signature, signedAt, nonce }
  }
}
```

### 能力声明

Node 向 Gateway 暴露的设备能力：

| 能力 | 命令 |
|------|------|
| `camera` | `camera.snap` |
| `location` | `location.get` |
| `notification` | `push.apns.register` |
| `screen` | `screen.record` |

---

## 核心 RPC 方法

| 功能 | Operator | Node |
|------|----------|------|
| 聊天 | `chat.send`, `chat.history` | - |
| 会话 | `sessions.list` | - |
| 节点调用 | `node.list`, `node.invoke` | 接收 invoke |
| 配置 | `config.get`, `config.set` | - |
| 定时任务 | `cron.list`, `cron.add` | - |

---

## 参考

- 协议详情：`docs/gateway/protocol.md`
- 远程访问：`docs/gateway/remote.md`
- 设备配对：`docs/gateway/pairing.md`
- iOS 集成：`docs/platforms/ios.md`
