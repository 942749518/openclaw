# OpenClaw Windows 源码部署指南

---

## 部署方案对比

| 维度 | WSL2 (推荐) | 原生 Windows |
|------|-------------|--------------|
| **Daemon 管理** | systemd service（可靠） | 计划任务 / Startup 文件夹（有局限） |
| **Docker 支持** | 原生 | 需 Docker Desktop |
| **配套 App** | 无 | 无 |
| **管理方式** | CLI + 浏览器 Control UI | CLI + 浏览器 Control UI |
| **Shell 工具链** | 完整 Linux 生态 | PowerShell / cmd，部分命令需适配 |
| **兼容性** | 官方推荐 | 可用但有粗糙点 |

> **两种方案都没有 Windows 原生配套 App**。macOS/iOS/Android 有平台特定的原生应用，Windows 没有。管理统一通过 CLI 和浏览器 Control UI (`http://localhost:18789`)。

---

## 方案 A：WSL2 部署（推荐）

### 1. 安装 WSL2

```powershell
# PowerShell (管理员)
wsl --install -d Ubuntu-24.04
```

### 2. 启用 systemd

```bash
# WSL 内执行
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

```powershell
# PowerShell 重启 WSL
wsl --shutdown
```

### 3. 构建源码

```bash
# WSL 内执行
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build
pnpm build
```

### 4. 初始化与启动

```bash
openclaw onboard
openclaw gateway run --bind loopback --port 18789
```

### 5. 注册后台守护进程

WSL2 拥有完整的 systemd，daemon 管理与 Linux 一致：

```bash
# WSL 内 — 注册为 systemd service，开机自动启动
openclaw gateway install

# 用户未登录也运行
sudo loginctl enable-linger "$(whoami)"
```

```powershell
# Windows 侧 — 开机自动拉起 WSL (管理员 PowerShell)
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec /bin/true" /sc onstart /ru SYSTEM
```

**systemd 的优势**：
- 开机/登录前自动启动
- `systemctl status openclaw-gateway` 查看状态
- 崩溃自动重启
- 日志统一管理 (`journalctl`)

### 6. LAN 访问（可选）

WSL2 有独立的虚拟网络，LAN 访问需在 Windows 侧配置端口转发：

```powershell
# PowerShell (管理员) — 需在每次 WSL 重启后重新设置
$wslIP = wsl hostname -I
netsh interface portproxy add v4tov4 listenport=18789 listenaddress=0.0.0.0 connectport=18789 connectaddress=$wslIP.Trim()
```

---

## 方案 B：原生 Windows 部署

### 前置要求

| 依赖 | 版本 |
|------|------|
| Node.js | >= 22.14（推荐 24） |
| pnpm | 10.32.1 |
| Git | Git for Windows |

### 1. 构建源码

```powershell
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build
pnpm build
```

### 2. 全局链接（可选）

```powershell
pnpm link --global
openclaw onboard
```

或不链接，在仓库内直接运行：

```powershell
pnpm openclaw onboard
```

### 3. Daemon 启动方式

原生 Windows 没有 systemd，`openclaw gateway install` 会尝试两种替代方案：

**方式一：Scheduled Tasks（计划任务）**
- 需要管理员权限
- 真正的开机自启（无需用户登录）
- `openclaw gateway install` 优先尝试此方式

**方式二：Startup 文件夹（启动文件夹）**
- 无需管理员权限
- 用户登录后才启动
- 路径：`%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup`
- 计划任务失败时的回退方案

> 手动启动：`openclaw gateway run --bind loopback --port 18789`

### 4. Windows 注意事项

- `.cmd` 命令通过 `cmd.exe /d /s /c` 包装执行
- npm 优先使用 `npm-cli.js` 直接调用而非 `.cmd` shim
- 安全审计 `--fix` 使用 Windows ACL 替代 POSIX `chmod`
- 安装时建议设置 `SHARP_IGNORE_GLOBAL_LIBVIPS=1`
- PATH 需包含 `npm config get prefix` 输出路径

---

## 安全加固

### Gateway 认证

```json5
// ~/.openclaw/openclaw.json
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: {
      mode: "token",
      token: "<长随机字符串>",
    },
  },
}
```

### 工具安全策略

```json5
{
  tools: {
    profile: "messaging",
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

### 沙箱隔离（需 Docker）

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        backend: "docker",
      },
    },
  },
}
```

### 安全审计

```bash
openclaw security audit          # 基础扫描
openclaw security audit --deep   # 深度扫描
openclaw security audit --fix    # 自动修复（Windows 用 ACL）
```

---

## 省 Token 配置

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      // 后续轮次跳过 bootstrap 重注入（每轮省 12K-60K chars）
      contextInjection: "continuation-skip",

      // Prompt Cache 长缓存
      params: {
        cacheRetention: "long",
      },

      // 心跳用便宜模型 + 隔离会话
      heartbeat: {
        model: "anthropic/claude-haiku-4",
        isolatedSession: true,
        lightContext: true,
        every: "55m",
      },

      // Compaction 用便宜模型
      compaction: {
        model: "anthropic/claude-haiku-4",
        maxHistoryShare: 0.4,
        truncateAfterCompaction: true,
      },

      // 过期缓存自动修剪旧 tool 结果
      contextPruning: {
        mode: "cache-ttl",
        ttl: "1h",
      },

      // 思考级别默认关闭
      thinkingDefault: "off",

      // 限制注入大小
      bootstrapMaxChars: 8000,
      bootstrapTotalMaxChars: 40000,
      contextLimits: {
        memoryGetMaxChars: 8000,
        toolResultMaxChars: 10000,
        postCompactionMaxChars: 1200,
      },
    },
  },
}
```

### 操作习惯

- 日常对话 `/think off`，复杂任务才 `/think high`
- 上下文膨胀时 `/compact`
- 按场景切模型 `/model <别名>`
- 避免频繁增减工具（会打破 prompt cache）

---

## 国内移动端 IM 推荐

### 渠道对比

| 渠道 | 流式支持 | 插件类型 | 推荐场景 |
|------|---------|---------|---------|
| **飞书** | Card Kit 实时流式 | bundled（内置） | **首选** — 流式体验最佳，国内直连 |
| **QQ Bot** | Block 流式 | bundled（内置） | 社交/个人场景，用户基数大 |
| **企业微信** | 流式回复 | 第三方插件 | 企业场景 |
| **钉钉** | Stream 模式 | 社区插件 | 企业场景（功能较少） |
| **微信** | 无流式 | 第三方插件 | 仅私聊，功能受限 |

### 飞书接入（推荐）

```bash
openclaw channels login --channel feishu
```

配置项：
- `FEISHU_APP_ID` — 飞书开放平台应用 ID
- `FEISHU_APP_SECRET` — 飞书开放平台应用密钥
- `domain: "feishu"` — 国内版（国际版用 `"lark"`）

前置步骤：
1. 飞书开放平台创建企业自建应用
2. 订阅事件 `im.message.receive_v1`
3. 获取 App ID 和 App Secret

### QQ Bot 接入

```bash
openclaw channels login --channel qqbot
```

前置步骤：
1. 注册 QQ 开放平台 (`q.qq.com`)
2. 创建机器人，获取 AppID + AppSecret
3. 配置 WebSocket 网关连接

---

## 关键配置文件

| 文件 | 位置 | 用途 |
|------|------|------|
| 主配置 | `~/.openclaw/openclaw.json` | 网关、代理、通道、模型配置（热重载） |
| 环境变量 | `~/.openclaw/.env` | API Key 等密钥 |
| 认证档案 | `~/.openclaw/agents/<id>/agent/auth-profiles.json` | OAuth/API Key 凭证 |
| 会话存储 | `~/.openclaw/agents/<id>/sessions/` | 会话路由与对话记录 |
| 构建输出 | `dist/` | TypeScript 编译结果 |
| UI 构建 | `ui/dist/` | Control UI 静态资源 |

---

## 常用命令

```bash
# 开发
pnpm dev                              # 开发模式
pnpm openclaw gateway run             # 启动网关
pnpm check                            # 格式 + lint + 类型检查
pnpm test                             # 运行测试
pnpm build                            # 构建

# 生产
openclaw onboard                      # 引导配置
openclaw gateway run --bind loopback  # 启动网关
openclaw gateway install              # 注册守护进程
openclaw gateway status --deep        # 深度状态检查
openclaw channels status --probe      # 通道状态
openclaw security audit --deep        # 安全审计
openclaw config set <key> <value>     # 修改配置
```
