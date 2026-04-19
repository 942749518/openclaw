# OpenClaw 配置参考

配置文件：`~/.openclaw/openclaw.json`

---

## models（模型提供商）

```jsonc
"models": {
  "providers": {
    "<id>": {
      "api": "openai-completions",     // 传输协议，OpenAI 兼容 API 用这个值
      "baseUrl": "https://open.bigmodel.cn/api/paas/v4",  // API 端点
      "models": [
        {
            "id": "glm-5.1",
            "name": "GLM 5.1",
            "contextWindow": 200000,
            "maxTokens": 131072
          }
      ]
    }
  }
}
```

---

## agents.defaults（代理默认配置）

```jsonc
"agents": {
  "defaults": {
    // 主模型，格式: provider/model-name
    "model": { "primary": "zai/glm-5.1" },
    // 按 TTL 修剪过期 tool 结果
    "contextPruning": {
      "mode": "cache-ttl",                      // "off" | "cache-ttl"
      "ttl": "1h"                               // 过期时间
    },
    // 心跳探测
    "heartbeat": {
      "isolatedSession": true,                  // 用隔离会话，不污染主会话
      "lightContext": true                     // 用轻量上下文
    },
    // 工作目录
    "workspace": "C:\\Users\\xxx\\.openclaw\\workspace"
  }
}
```

---

## channels（渠道）

```jsonc
"channels": {
  "qqbot": {
    "enabled": true,                            // 启用
    "markdownSupport": true,                    // 支持 markdown 消息
    "streaming": { "mode": "partial" },         // "off" | "partial"（block 流式）
    "allowFrom": ["*"]                          // 允许的用户，["*"] = 所有人
  }
}
```

---

## gateway（网关）

```jsonc
"gateway": {
  "mode": "local",                             // "local" | "remote"
  "bind": "loopback",                          // "loopback" = 只监听 127.0.0.1
  "port": 18789,
  "auth": {
    "mode": "token",                           // token 认证
    "token": "长随机字符串"
  }
}
```

---

## session（会话）

```jsonc
"session": {
  "dmScope": "per-channel-peer"                // 每个渠道+对方独立一个会话
}
```

---

## tools（工具策略）

```jsonc
"tools": {
  "profile": "coding",                         // full | coding | messaging | minimal
  // coding = 文件读写+exec+web+会话+记忆+媒体生成
  "exec": {
    "security": "allowlist",                   // "deny" | "allowlist" | "full"
    "ask": "on-miss",                          // "off" | "on-miss" | "always"
    "strictInlineEval": true                   // python -c / node -e 等仍需审批
  },
  "elevated": { "enabled": false }             // 禁止提权逃逸
}
```

**profile 说明：**

| Profile | 包含的工具 |
|---------|-----------|
| `full` | 全部（无限制） |
| `coding` | 文件读写、命令执行、web搜索、会话管理、记忆、媒体生成 |
| `messaging` | 仅消息收发和会话查看 |
| `minimal` | 仅 `session_status` |

---

## skills（技能安装）

```jsonc
"skills": {
  "install": {
    "nodeManager": "npm"                       // "npm" | "pnpm" | "yarn" | "bun"
  }
}
```

---

## wizard / meta（自动生成，不用手动改）

```jsonc
"wizard": {
  "lastRunAt": "...",                          // 上次向导运行时间
  "lastRunVersion": "...",                     // 上次运行版本
  "lastRunCommand": "onboard",                 // 上次运行命令
  "lastRunMode": "local"                       // "local" | "remote"
},
"meta": {
  "lastTouchedVersion": "...",                 // 最后修改版本
  "lastTouchedAt": "..."                       // 最后修改时间
}
```

---

## 注意事项

- `cacheRetention`（prompt cache）对 ZAI 无效，不要设置
- `compaction.truncateAfterCompaction` 类型定义存在但 Zod schema 缺失，写了可能被丢弃
- `exec.askFallback` 不存在于配置 schema，不要写
