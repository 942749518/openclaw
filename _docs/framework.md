# OpenClaw 框架概览

**OpenClaw** 是一个自托管的多渠道 AI 网关，将 WhatsApp、Telegram、Slack 等 20+ 通讯渠道与 OpenAI、Anthropic 等 AI 模型连接，打造个人专属的智能助手。

---

## 系统架构

```
┌─────────────────────────────────────────────────────────┐
│   Clients (macOS / iOS / Android / Web / CLI)          │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼ WebSocket
┌─────────────────────────────────────────────────────────┐
│   Gateway (控制平面)                                      │
│   ├─ Channel Registry   通道注册中心                       │
│   ├─ Provider Registry  模型提供商注册中心                  │
│   ├─ Agent Runtime      AI 代理运行时                      │
│   └─ Control UI         Web 控制界面                       │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│   Plugins                                               │
│   ├─ Channel Plugins    (Discord, Matrix, Telegram...)   │
│   ├─ Provider Plugins   (OpenAI, Anthropic, Google...)   │
│   └─ Capability Plugins (TTS, Image Gen, Search...)      │
└─────────────────────────────────────────────────────────┘
```

---

## 目录结构

```
openclaw/
├── src/                    # 核心源代码
│   ├── plugin-sdk/         # 插件 SDK (公共 API)
│   ├── channels/           # 通道系统实现
│   ├── plugins/            # 插件运行时
│   ├── gateway/            # 网关核心
│   ├── cli/                # 命令行入口
│   └── agents/             # AI 代理引擎
├── extensions/             # 插件包 (100+)
├── apps/                   # 原生应用
│   ├── ios/                # iOS (SwiftUI)
│   ├── android/            # Android (Compose)
│   ├── macos/              # macOS (SwiftUI)
│   └── shared/             # OpenClawKit 共享框架
├── ui/                     # Web 控制界面
├── docs/                   # 文档 (Mintlify)
└── skills/                 # 预置技能
```

---

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| **后端** | TypeScript + Node.js | 核心网关与插件系统 |
| **类型验证** | TypeBox + Zod | JSON Schema 与运行时校验 |
| **移动** | SwiftUI / Jetpack Compose | iOS / Android 原生 UI |
| **构建** | pnpm + tsdown | 包管理与 TypeScript 编译 |
| **测试** | Vitest | 单元测试与集成测试 |
| **代码质量** | Oxlint + Oxfmt | 静态检查与格式化 |

---

## 核心概念

### 1. 插件系统

插件是 OpenClaw 的扩展单元，通过 `openclaw.plugin.json` 声明能力：

```json
{
  "id": "anthropic",
  "providers": ["anthropic"],
  "cliBackends": ["claude-cli"],
  "modelSupport": {
    "modelPrefixes": ["claude-"]
  }
}
```

**插件类型**:
- `registerProvider()` — AI 模型提供商
- `registerChannel()` — 消息通道
- `registerSpeechProvider()` — 语音合成
- `registerImageGenerationProvider()` — 图像生成

### 2. 插件边界

- 插件代码**只能**从 `openclaw/plugin-sdk/*` 导入
- 禁止直接访问 `src/**` 内部实现
- 禁止相对路径逃逸当前包目录

### 3. 通道架构

所有通道实现统一接口：

```typescript
interface ChannelPlugin {
  id: ChannelId;
  setup: ChannelSetupAdapter;
  send: ChannelSendAdapter;
  monitor?: ChannelMonitorAdapter;
}
```

### 4. 网关协议

- **WebSocket** 双向实时通信
- **TypeBox Schema** 类型安全的消息协议
- **设备配对** 基于身份的认证模型

---

## 关键模块

| 模块 | 路径 | 职责 |
|------|------|------|
| **Gateway** | `src/gateway/` | WebSocket 服务器、客户端管理、认证 |
| **Channels** | `src/channels/` | 消息路由、状态监控、通道注册 |
| **Plugins** | `src/plugins/` | 插件发现、加载、运行时管理 |
| **Plugin SDK** | `src/plugin-sdk/` | 公共 API、类型定义 |
| **CLI** | `src/cli/` | 命令行界面与命令实现 |
| **Agents** | `src/agents/` | AI 代理运行与工具调度 |
| **Media** | `src/media/` | 图像/视频生成、TTS、媒体理解 |

---

## 应用架构

### OpenClawKit (共享框架)

```
OpenClawProtocol    # 网关协议模型
OpenClawKit         # 设备命令与连接
OpenClawChatUI      # 共享 UI 组件
```

### 设备命令

- `BrowserCommands` — 浏览器控制
- `CameraCommands` — 相机/屏幕录制
- `CanvasCommands` — A2UI 画布渲染
- `SystemCommands` — 系统通知/执行
- `TalkCommands` — 语音对话

---

## 开发命令

```bash
# 依赖安装
pnpm install

# 开发模式
pnpm dev
pnpm gateway:dev

# 构建
pnpm build

# 测试
pnpm test
pnpm test:coverage

# 代码检查
pnpm check
pnpm format
pnpm lint

# 移动应用
pnpm ios:run
pnpm android:run
```

---

## 架构原则

1. **核心与插件分离** — `src/` 与 `extensions/` 严格隔离
2. **向后兼容** — 第三方插件 API 稳定性优先
3. **类型优先** — TypeScript + TypeBox 端到端类型安全
4. **懒加载** — 热路径优化，运行时延迟加载
5. **注册表模式** — 动态发现与加载

---

## 相关链接

- **官网**: https://openclaw.ai
- **文档**: https://docs.openclaw.ai
- **GitHub**: https://github.com/openclaw/openclaw
- **Discord**: https://discord.gg/clawd
