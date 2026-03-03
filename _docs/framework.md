# OpenClaw CLI 启动流程架构

本文档详细描述从执行 `openclaw` 命令到 Gateway 服务器最终运行的完整流程。

---

## 架构概览

OpenClaw 采用分层架构设计，支持命令按需加载（Lazy Loading）、插件化扩展和热重载配置。

```
┌─────────────────────────────────────────────────────────────┐
│                         用户层                                │
│                    $ openclaw gateway run                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       入口包装层                              │
│              openclaw.mjs → dist/entry.js                    │
│         (编译缓存、警告过滤、环境初始化)                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        CLI 核心层                             │
│              runCli() → buildProgram()                       │
│       (命令注册、参数解析、子 CLI 路由)                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       命令处理层                              │
│              gateway-cli → run.ts → startGatewayServer()     │
│              (选项解析、端口管理、依赖注入)                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       Gateway 核心层                          │
│              server.impl.ts (WebSocket + HTTP)               │
│    (配置加载、插件初始化、频道连接、会话管理)                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 阶段 1：入口层 (Wrapper)

### 1.1 openclaw.mjs (生产环境入口)

**文件位置**: `openclaw.mjs`

**职责**:
- 启用 Node.js 编译缓存 (`module.enableCompileCache()`)
- 安装进程警告过滤器
- 尝试导入构建产物 (`dist/entry.js` 或 `dist/entry.mjs`)

```javascript
// 伪代码
if (await tryImport("./dist/entry.js")) {
  // 使用 CommonJS 构建产物
} else if (await tryImport("./dist/entry.mjs")) {
  // 使用 ESM 构建产物
}
```

### 1.2 entry.ts (实际入口)

**文件位置**: `src/entry.ts`

**关键操作**:
1. 设置进程标题: `process.title = "openclaw"`
2. 处理 Windows argv 规范化 (`normalizeWindowsArgv`)
3. 检查并抑制实验性警告（respawn 机制）
4. 解析 CLI profile (`--profile dev`)
5. 导入并执行 `runCli()`

**Respawn 机制**:
如果检测到需要抑制实验性警告，会 respawn 子进程并添加 `--disable-warning=ExperimentalWarning` 标志。

---

## 阶段 2：CLI 初始化

### 2.1 runCli 主函数

**文件位置**: `src/cli/run-main.ts`

**执行流程**:

```typescript
export async function runCli(argv: string[] = process.argv) {
  // 1. 环境准备
  const normalizedArgv = normalizeWindowsArgv(argv);
  loadDotEnv({ quiet: true });           // 加载 .env 文件
  normalizeEnv();                         // 规范化环境变量
  
  // 2. 运行时检查
  assertSupportedRuntime();              // 检查 Node.js >= 22
  
  // 3. 子 CLI 路由（特殊命令前置处理）
  if (await tryRouteCli(normalizedArgv)) {
    return;
  }
  
  // 4. 日志系统初始化
  enableConsoleCapture();                // 捕获控制台输出到结构化日志
  
  // 5. 构建 Commander 程序
  const { buildProgram } = await import("./program.js");
  const program = buildProgram();
  
  // 6. 错误处理
  installUnhandledRejectionHandler();
  process.on("uncaughtException", handler);
  
  // 7. 主命令注册（延迟加载）
  const primary = getPrimaryCommand(parseArgv);
  if (primary) {
    await registerCoreCliByName(program, ctx, primary, parseArgv);
    await registerSubCliByName(program, primary);
  }
  
  // 8. 插件命令注册
  if (!shouldSkipPluginRegistration) {
    registerPluginCliCommands(program, loadConfig());
  }
  
  // 9. 解析并执行命令
  await program.parseAsync(parseArgv);
}
```

### 2.2 构建 Commander 程序

**文件位置**: `src/cli/program/build-program.ts`

```typescript
export function buildProgram() {
  const program = new Command();
  const ctx = createProgramContext();      // 创建程序上下文
  
  setProgramContext(program, ctx);
  configureProgramHelp(program, ctx);      // 配置帮助文本
  registerPreActionHooks(program, ctx.programVersion);
  registerProgramCommands(program, ctx, argv);
  
  return program;
}
```

---

## 阶段 3：命令注册 (按需加载)

### 3.1 延迟注册策略

**文件位置**: `src/cli/program/command-registry.ts`

OpenClaw 采用智能的延迟加载策略：

1. **根据 argv[2] 识别主命令**
2. **只注册识别到的主命令**，而非所有命令
3. **动态 import() 加载命令处理器**

**核心命令映射**:

| 命令 | 注册函数 | 文件 |
|------|----------|------|
| `setup` | registerSetupCommand | register.setup.ts |
| `onboard` | registerOnboardCommand | register.onboard.ts |
| `configure` | registerConfigureCommand | register.configure.ts |
| `config` | registerConfigCli | config-cli.ts |
| `gateway` | registerGatewayCli | gateway-cli/register.ts |
| `agent` | registerAgentCommand | register.agent.ts |
| `channels` | registerChannelsCli | register.channels.ts |
| `message` | registerMessageCli | register.message.ts |
| `models` | registerModelsCli | register.models.ts |

### 3.2 Gateway 命令注册示例

**文件位置**: `src/cli/gateway-cli/register.ts`

```typescript
export function registerGatewayCli(program: Command) {
  const gateway = addGatewayRunCommand(
    program
      .command("gateway")
      .description("Run, inspect, and query the WebSocket Gateway")
  );

  // 子命令: gateway run
  addGatewayRunCommand(
    gateway.command("run").description("Run the WebSocket Gateway (foreground)")
  );

  // 子命令: gateway status
  addGatewayServiceCommands(gateway, { statusDescription: "..." });

  // 子命令: gateway call
  gateway
    .command("call")
    .description("Call a Gateway method")
    .action(async (method, opts) => { ... });
}
```

---

## 阶段 4：具体命令执行 (以 gateway run 为例)

### 4.1 运行命令处理器

**文件位置**: `src/cli/gateway-cli/run.ts`

**函数**: `runGatewayCommand(opts)`

**执行步骤**:

```typescript
async function runGatewayCommand(opts: GatewayRunOpts) {
  // 1. Dev 模式检查
  const isDevProfile = process.env.OPENCLAW_PROFILE === "dev";
  const devMode = opts.dev || isDevProfile;
  
  // 2. 日志配置
  setConsoleTimestampPrefix(true);
  setVerbose(opts.verbose);
  setGatewayWsLogStyle(wsLogStyle);
  
  // 3. Dev 模式配置
  if (devMode) {
    await ensureDevGatewayConfig({ reset: opts.reset });
  }
  
  // 4. 端口解析
  const cfg = loadConfig();
  const port = parsePort(opts.port) ?? resolveGatewayPort(cfg);
  
  // 5. 强制释放端口
  if (opts.force) {
    const { killed } = await forceFreePortAndWait(port, {
      timeoutMs: 2000,
      sigtermTimeoutMs: 700
    });
  }
  
  // 6. 启动 Gateway 服务器
  await startGatewayServer(port, { ... });
}
```

### 4.2 端口冲突处理

使用 `--force` 标志时，会强制终止占用端口的进程：

```typescript
const { killed, waitedMs, escalatedToSigkill } = await forceFreePortAndWait(port, {
  timeoutMs: 2000,        // 总超时时间
  intervalMs: 100,        // 检查间隔
  sigtermTimeoutMs: 700   // SIGTERM 等待时间，之后使用 SIGKILL
});
```

---

## 阶段 5：Gateway 服务器启动

### 5.1 startGatewayServer 主函数

**文件位置**: `src/gateway/server.impl.ts`

**初始化流程**:

```typescript
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {}
): Promise<GatewayServer> {
  
  // 1. 环境设置
  process.env.OPENCLAW_GATEWAY_PORT = String(port);
  
  // 2. 配置加载与验证
  let configSnapshot = await readConfigFileSnapshot();
  
  // 3. 自动迁移旧版配置
  if (configSnapshot.legacyIssues.length > 0) {
    const { config: migrated, changes } = migrateLegacyConfig(configSnapshot.parsed);
    await writeConfigFile(migrated);
  }
  
  // 4. 插件自动启用
  const autoEnable = applyPluginAutoEnable({ config, env: process.env });
  if (autoEnable.changes.length > 0) {
    await writeConfigFile(autoEnable.config);
  }
  
  // 5. Secrets 激活
  await activateRuntimeSecrets(config, { reason: "startup", activate: true });
  
  // 6. Gateway 认证初始化
  const authBootstrap = await ensureGatewayStartupAuth({ cfg, persist: true });
  
  // 7. 运行时配置解析
  const runtimeConfig = await resolveGatewayRuntimeConfig({ cfg, port, ... });
  
  // 8. 插件系统初始化
  const { pluginRegistry, gatewayMethods } = loadGatewayPlugins({ cfg, ... });
  
  // 9. 频道客户端初始化
  const channelLogs = Object.fromEntries(listChannelPlugins().map(...));
  const channelRuntimeEnvs = Object.fromEntries(...);
  
  // 10. HTTP/WebSocket 服务器启动
  const { server, wsServer } = await startHttpServer({ port, bindHost, ... });
  
  // 11. 频道连接启动
  await connectChannels(channelPlugins, channelContexts);
  
  // 12. CRON 调度器启动
  startCronScheduler();
  
  // 13. 关闭处理器注册
  registerShutdownHandlers();
  
  return { ... };
}
```

### 5.2 配置验证流程

```
readConfigFileSnapshot()
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 检查文件存在  │────▶│ 验证 JSON    │────▶│ 模式验证     │
└──────────────┘     └──────────────┘     └──────────────┘
       │                                           │
       ▼                                           ▼
  文件不存在                               发现 legacyIssues
       │                                           │
       ▼                                           ▼
  创建默认配置                          migrateLegacyConfig()
                                                          │
                                                          ▼
                                                  写入迁移后配置
```

### 5.3 插件系统初始化

**文件位置**: `src/gateway/server-plugins.ts`

```typescript
export function loadGatewayPlugins(params: {
  cfg: OpenClawConfig;
  workspaceDir: string;
  log: Logger;
  coreGatewayHandlers: CoreGatewayHandlers;
  baseMethods: string[];
}): { pluginRegistry: PluginRegistry; gatewayMethods: string[] } {
  
  // 1. 扫描插件目录
  const pluginDirs = scanPluginDirectories();
  
  // 2. 加载每个插件
  const plugins = pluginDirs.map(dir => {
    const manifest = loadPluginManifest(dir);
    const module = loadPluginModule(dir);
    return { manifest, module };
  });
  
  // 3. 过滤启用的插件
  const enabledPlugins = plugins.filter(p => isPluginEnabled(cfg, p.manifest));
  
  // 4. 初始化插件服务
  const pluginServices = enabledPlugins.map(p => p.module.initialize({
    config: cfg,
    workspaceDir,
    logger: log.child(p.manifest.name)
  }));
  
  // 5. 注册 Gateway 方法
  const gatewayMethods = [
    ...baseMethods,
    ...enabledPlugins.flatMap(p => p.module.gatewayMethods ?? [])
  ];
  
  return { pluginRegistry, gatewayMethods };
}
```

### 5.4 频道连接初始化

支持的频道类型:

| 频道 | 实现文件 | 协议/库 |
|------|----------|---------|
| WhatsApp | `src/whatsapp/` | Baileys |
| Telegram | `src/telegram/` | grammY |
| Discord | `src/discord/` | discord.js |
| Slack | `src/slack/` | Bolt |
| Signal | `src/signal/` | signal-cli |
| iMessage | `src/imessage/` | 本地集成 |
| WebChat | `src/web/` | WebSocket |

**初始化流程**:

```typescript
async function connectChannels(plugins, contexts) {
  for (const plugin of plugins) {
    if (!plugin.isEnabled()) continue;
    
    try {
      const client = await plugin.initialize(contexts[plugin.id]);
      plugin.setClient(client);
      
      // 设置消息处理器
      client.onMessage((msg) => handleInboundMessage(msg));
      
      log.info(`Channel ${plugin.id} connected`);
    } catch (err) {
      log.error(`Channel ${plugin.id} failed: ${err}`);
    }
  }
}
```

---

## 阶段 6：依赖注入系统

### 6.1 createDefaultDeps

**文件位置**: `src/cli/deps.ts`

CLI 使用依赖注入容器管理外部服务调用：

```typescript
export type CliDeps = {
  sendMessageWhatsApp: typeof sendMessageWhatsApp;
  sendMessageTelegram: typeof sendMessageTelegram;
  sendMessageDiscord: typeof sendMessageDiscord;
  sendMessageSlack: typeof sendMessageSlack;
  sendMessageSignal: typeof sendMessageSignal;
  sendMessageIMessage: typeof sendMessageIMessage;
};

export function createDefaultDeps(): CliDeps {
  return {
    // 使用动态 import 实现延迟加载
    sendMessageWhatsApp: async (...args) => {
      const { sendMessageWhatsApp } = await import("../channels/web/index.js");
      return await sendMessageWhatsApp(...args);
    },
    sendMessageTelegram: async (...args) => {
      const { sendMessageTelegram } = await import("../telegram/send.js");
      return await sendMessageTelegram(...args);
    },
    // ... 其他频道
  };
}
```

**设计优势**:
- **延迟加载**: 只在需要时加载模块
- **可测试性**: 测试时可以注入 mock 实现
- **解耦**: CLI 逻辑与具体实现分离

---

## 完整调用链示例

以 `openclaw gateway run --port 8080 --verbose` 为例：

```
1. Shell
   └── $ openclaw gateway run --port 8080 --verbose

2. openclaw.mjs
   ├── 启用编译缓存
   ├── 安装警告过滤器
   └── import("./dist/entry.js")

3. entry.ts
   ├── process.title = "openclaw"
   ├── normalizeWindowsArgv()
   ├── ensureExperimentalWarningSuppressed() [可能 respawn]
   └── runCli(process.argv)

4. run-main.ts (runCli)
   ├── loadDotEnv()
   ├── normalizeEnv()
   ├── assertSupportedRuntime()
   ├── tryRouteCli() [false]
   ├── enableConsoleCapture()
   ├── buildProgram()
   │   └── createProgramContext()
   └── registerProgramCommands()

5. command-registry.ts
   ├── getPrimaryCommand() → "gateway"
   ├── registerCoreCliByName("gateway")
   │   └── import("./register.gateway.js")
   └── program.parseAsync()

6. gateway-cli/register.ts
   ├── program.command("gateway")
   └── addGatewayRunCommand(gateway.command("run"))

7. gateway-cli/run.ts (Action Handler)
   ├── parse opts: { port: 8080, verbose: true }
   ├── setVerbose(true)
   ├── loadConfig()
   ├── parsePort(8080) → 8080
   ├── forceFreePortAndWait(8080) [if --force]
   └── startGatewayServer(8080, opts)

8. gateway/server.impl.ts
   ├── readConfigFileSnapshot()
   ├── migrateLegacyConfig() [if needed]
   ├── applyPluginAutoEnable()
   ├── activateRuntimeSecrets()
   ├── ensureGatewayStartupAuth()
   ├── loadGatewayPlugins()
   ├── resolveGatewayRuntimeConfig()
   ├── startHttpServer(port, bindHost)
   │   ├── createExpressApp()
   │   ├── setupWebSocketServer()
   │   └── server.listen(port, bindHost)
   ├── connectChannels()
   ├── startCronScheduler()
   └── registerShutdownHandlers()

9. Gateway 运行中
   ├── WebSocket Server: ws://localhost:8080
   ├── HTTP API Endpoints
   ├── Channel Connections
   │   ├── WhatsApp Client
   │   ├── Telegram Bot
   │   └── ...
   ├── Plugin Services
   └── Session Management
```

---

## 关键设计特点

### 1. 延迟加载 (Lazy Loading)

```typescript
// 使用动态 import 避免启动时加载所有模块
const { handler } = await import("./handler.js");
```

**优势**:
- 减少启动时间
- 降低内存占用
- 支持热重载

### 2. 插件化架构

- **自动发现**: 扫描 `extensions/` 和 `src/plugins/`
- **配置驱动**: 通过 `openclaw.json` 启用/禁用插件
- **热重载**: 支持运行时重新加载插件

### 3. 错误处理

```typescript
// 全局错误处理器
installUnhandledRejectionHandler();

process.on("uncaughtException", (error) => {
  console.error("[openclaw] Uncaught exception:", formatUncaughtError(error));
  process.exit(1);
});
```

### 4. 配置管理

- **多层配置**: 环境变量 > 命令行参数 > 配置文件
- **验证**: JSON Schema 验证配置有效性
- **迁移**: 自动升级旧版配置到新格式
- **热重载**: 监听配置文件变化并重新加载

### 5. 端口管理

```typescript
// 智能端口处理
if (opts.force) {
  await forceFreePortAndWait(port, {
    timeoutMs: 2000,
    sigtermTimeoutMs: 700  // 优雅关闭超时后强制 kill
  });
}
```

---

## 扩展阅读

- **Gateway 架构**: `docs/concepts/architecture.md`
- **插件开发**: `docs/reference/plugins.md`
- **配置参考**: `docs/gateway/configuration.md`
- **CLI 命令**: `docs/cli/gateway.md`
