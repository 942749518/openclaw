# OpenClaw 原生 Windows 部署（QQ Bot + 国内模型）

## 1. 前置条件

| 依赖 | 要求 | 安装 |
|------|------|------|
| Node.js | >= 22.14 | https://nodejs.org |
| pnpm | 10.32.1 | `npm i -g pnpm@10.32.1` |
| Git | Git for Windows | https://git-scm.com |
| 智谱 Key | 注册智谱 | https://open.bigmodel.cn |
| QQ Bot | 创建机器人 | https://q.qq.com |

## 2. 构建

```powershell
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build
pnpm build
pnpm link --global              # 可选，链接后全局可用 openclaw
openclaw onboard --non-interactive --skip-health
```

## 3. 配置

### ~/.openclaw/.env

```env
ZAI_API_KEY=your-zai-key.zhipu
QQBOT_APP_ID=your-qq-app-id
QQBOT_CLIENT_SECRET=your-qq-client-secret
```

### ~/.openclaw/openclaw.json

```jsonc
{
  "models": {
    "providers": {
      "zai": {
        "api": "openai-completions",
        "baseUrl": "https://open.bigmodel.cn/api/paas/v4"
      }
    }
  },

  "agents": {
    "defaults": {
      "model": { "primary": "zai/glm-5.1" },

      "contextInjection": "continuation-skip",

      "compaction": {
        "model": "zai/glm-4.5-flash",
        "maxHistoryShare": 0.4,
        "truncateAfterCompaction": true
      },

      "contextPruning": { "mode": "cache-ttl", "ttl": "1h" },
      "thinkingDefault": "off",
      "bootstrapMaxChars": 8000,
      "bootstrapTotalMaxChars": 40000,
      "contextLimits": {
        "memoryGetMaxChars": 8000,
        "toolResultMaxChars": 10000,
        "postCompactionMaxChars": 1200
      },

      "heartbeat": {
        "model": "zai/glm-4.5-flash",
        "isolatedSession": true,
        "lightContext": true,
        "every": "55m"
      }
    }
  },

  "channels": {
    "qqbot": {
      "enabled": true,
      "markdownSupport": true,
      "streaming": { "mode": "partial" },
      "allowFrom": ["*"]
    }
  },

  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "port": 18789,
    "auth": { "mode": "token", "token": "替换为长随机字符串" }
  },

  "tools": {
    "fs": { "workspaceOnly": true },
    "exec": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "strictInlineEval": true
    },
    "elevated": { "enabled": false },
    "applyPatch": { "workspaceOnly": true }
  }
}
```

> `cacheRetention` 对 ZAI 无效，不要设置。GLM-4.5-Flash 免费但需 ZAI_API_KEY。

## 4. QQ Bot 接入

1. https://q.qq.com 扫码登录 → 创建机器人 → 记下 **AppID** 和 **AppSecret**
2. AppSecret 离开页面后无法再查看，只能重新生成

```powershell
openclaw plugins install @tencent-connect/openclaw-qqbot@latest
openclaw channels add --channel qqbot --use-env
openclaw gateway restart
openclaw channels status --probe
```

## 5. 启动

```powershell
# 手动
openclaw gateway run --bind loopback --port 18789 --force

# 注册开机自启（需管理员）
openclaw gateway install
```

## 6. 验证

```powershell
openclaw --version                          # 版本
openclaw doctor                             # 健康检查
openclaw gateway status --deep              # 网关状态
openclaw channels status --probe            # 渠道连通性
# 浏览器打开 http://localhost:18789         # Control UI
# QQ 中给机器人发消息                        # 端到端验证
```

## 7. Exec 白名单

```powershell
openclaw exec-policy preset allowlist-on-miss   # 初始化策略
```

白名单外的命令会弹审批，回复 `allow-always` 记住。常用命令逐步积累。

```powershell
openclaw exec-policy show        # 查看策略
openclaw approvals edit          # 手动编辑白名单
```

## 8. 省 Token 速查

- `/think off` 日常，`/think high` 复杂任务
- 上下文膨胀时 `/compact`
- 不要频繁增减工具（改变 prompt 结构会重复注入）
- compaction/心跳已配免费 GLM-4.5-Flash

## 9. 常见问题

| 问题 | 排查 |
|------|------|
| QQ Bot 连不上 | `channels status --probe` 看错误；确认 AppID/Secret 正确；确认机器人已上线 |
| Exec 被拒绝 | 正常行为，回复 `allow-always` 加入白名单 |
| Gateway 启动后无响应 | `gateway status`；`gateway logs`；确认 `.env` 中 Key 正确 |
| 计划任务创建失败 | 需管理员 PowerShell 重试，否则回退 Startup 文件夹 |
| Token 消耗偏高 | 确认 `contextInjection: "continuation-skip"` 已设置；确认 compaction/心跳用的是 GLM-4.5-Flash |
