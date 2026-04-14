# Part 2 (上): OpenClaw 架构分析 — 系统设计与执行模型

---

## 1. 架构总览

OpenClaw 是一个基于 **TypeScript/Node.js** 的个人 AI Agent 框架，允许用户部署持久运行的 Agent，通过即时通讯应用交互，拥有对邮件、日历、文件系统、浏览器、Shell 的广泛访问权限。

### 1.1 信任模型 — 核心假设

OpenClaw 的安全架构建立在一个 **关键假设** 之上（摘自 SECURITY.md）：

> **OpenClaw 不是多租户系统。每个 Gateway 实例的所有认证调用者都被视为可信操作者。**

这意味着：
- Session ID 是路由控制，**不是**权限边界
- 如果一个操作者能看到另一个的数据 → **预期行为**
- 主机/OS 管理员边界被视为可信
- 修改 `~/.openclaw/` 的任何人被视为可信操作者
- 已安装的插件以 Gateway 权限在进程内运行

### 1.2 代码结构

```
openclaw_new/
├── src/                    # 主源码
│   ├── agents/             # Agent 执行运行时
│   ├── gateway/            # WebSocket 控制面
│   ├── plugins/            # 插件发现、加载、验证
│   ├── security/           # 安全模块
│   ├── config/             # 配置类型与验证
│   ├── channels/           # 通道允许列表与路由
│   ├── secrets/            # 密钥运行时
│   ├── sessions/           # 会话管理
│   ├── infra/              # 执行审批、临时文件
│   ├── cli/                # 命令行接口
│   └── logging/            # 日志子系统
├── extensions/             # 89+ 捆绑插件/扩展
├── apps/                   # macOS/iOS/Android 原生客户端
├── packages/               # 共享包
├── skills/                 # 捆绑工作区技能
├── Dockerfile              # 容器化
└── docker-compose.yml      # 编排
```

### 1.3 通信流

```
消息通道 (WhatsApp/Telegram/Slack/Discord/...)
    ↓
Gateway (WebSocket 控制面 @ 127.0.0.1:18789)
    ├─ Pi Agent 运行时 (RPC 模式)
    ├─ CLI 接口
    ├─ WebChat UI
    ├─ Control UI
    └─ Canvas Host (A2UI)
```

---

## 2. Agent 执行模型

### 2.1 核心入口: runEmbeddedPiAgent()

文件: `src/agents/pi-embedded-runner/run.ts`

这是 Agent 执行的主入口点，接收参数包括：sessionId、sessionKey、message、config、provider/model、workspaceDir、agentId、messageChannel 等。

### 2.2 完整执行管道

```
runEmbeddedPiAgent()
  └→ enqueueSession() + enqueueGlobal() [基于 Lane 的任务排队]
      └→ 工作区设置 & 插件引导
          ├→ ensureRuntimePluginsLoaded()
          ├→ resolveModelAsync() [模型选择 + 后备]
          ├→ buildEmbeddedRunPayloads() [准备 LLM 载荷]
          └→ 重试循环 (认证后备 + 过载恢复)
              └→ runEmbeddedAttempt()
                  ├→ resolveSandboxContext()
                  ├→ resolveBootstrapContextForRun()
                  ├→ createAgentSession() [@pi-coding-agent]
                  ├→ SessionManager.open(sessionFile)
                  ├→ createOpenClawTools() [构建工具注册表]
                  ├→ applyToolPolicyPipeline() [安全过滤]
                  ├→ buildEmbeddedSystemPrompt()
                  └→ streamSimple(api, sessionCtx, callbacks)
                      └→ 工具执行循环:
                          ├→ Model 输出 tool_use blocks
                          ├→ runBeforeToolCallHook()
                          ├→ tool.execute(callId, args)
                          ├→ 捕获结果到 SessionManager
                          └→ 循环直到 end_turn
```

### 2.3 工具策略管道 — 多层过滤

工具可用性通过 **7 层策略管道** 过滤：

```
Layer 1: Profile 策略    (tools.<profile>.allow/deny)
Layer 2: Provider 策略   (模型提供商特定限制)
Layer 3: Global 策略     (config.tools.allow/deny)
Layer 4: Agent 策略      (config.agents.<id>.tools.allow/deny)
Layer 5: Group 策略      (DM/群组上下文规则)
Layer 6: Subagent 策略   (深度级别限制)
Layer 7: Gateway HTTP 拒绝列表
```

### 2.4 子 Agent 工具限制

子 Agent 有额外的工具限制：

**始终拒绝 (不论深度)**:
- gateway, agents_list, whatsapp_login, session_status, cron, sessions_send

**叶子节点拒绝 (depth >= maxSpawnDepth)**:
- subagents, sessions_list, sessions_history, sessions_spawn

### 2.5 会话管理

- 基于 JSONL 的会话记录持久化
- 消息 parentId 链接 (DAG 结构)
- 会话修复: `session-transcript-repair.ts` 处理畸形的工具使用/结果配对
- SessionKey 格式: `agent:<agentId>:<channel>:<scope>:<recipient>[#threadId]`

---

## 3. 插件/扩展系统

### 3.1 插件生命周期

```
1. 发现 (discoverOpenClawPlugins)
   → 扫描 plugins/ 目录 + 工作区包
   → 读取清单元数据

2. 验证 (loadPluginManifestRegistry)
   → 针对 schema 验证清单
   → 检查冲突/重复
   → 此时不执行代码

3. 加载 (Jiti 动态导入)
   → 首次使用时导入插件模块
   → 通过 PluginRegistry 缓存 (最大 128 条)

4. 初始化
   → 运行插件默认导出
   → 注册钩子、命令、工具

5. 工具注册
   → 插件通过 api.registerTool(name, factory) 注册
   → 运行时调用工厂函数创建工具实例
```

### 3.2 Jiti 加载器

文件: `src/plugins/loader.ts`

- 使用 **Jiti** 进行动态模块加载
- 每个插件创建隔离的 Jiti 实例
- SDK 别名映射: `openclaw/plugin-sdk/*` → 核心 `src/plugin-sdk/*`
- 阻止跨插件/私有代码导入

**关键问题: 所有插件共享同一 V8 上下文和进程环境。**

### 3.3 插件安装安全扫描

文件: `src/plugins/install-security-scan.ts`

扫描内容：
- 捆绑源、包源、文件源
- 返回 `blocked` 原因如果安装应失败
- 支持安装与更新模式

### 3.4 插件清单格式

```typescript
type OpenClawPluginManifest = {
  id: string;
  version: string;
  name: string;
  config?: OpenClawPluginConfigSchema;
  entryPoint?: string;      // 默认: plugin.ts
  setupEntry?: string;      // 预监听设置阶段
};
```

---

## 4. Gateway 与通信

### 4.1 认证模式

| 模式 | 说明 |
|------|------|
| none | 无认证 (仅推荐 loopback) |
| token | Authorization 头部 Bearer token |
| password | HTTP Basic 认证 |
| tailscale | Tailscale 身份头部 |
| device-token | 引导/设备配对令牌 |
| trusted-proxy | 代理白名单后的 X-Real-IP |

### 4.2 本地直连请求旁路

`isLocalDirectRequest()` — 仅当无转发头部且为回环地址时返回 true。防止通过 X-Forwarded-For 注入的欺骗。

**但**: Gateway 认证对 loopback 是可选的。如果机器被入侵，本地进程可以直接调用 Gateway 而无需令牌/密码。

### 4.3 HTTP 工具调用

端点: `POST /tools/invoke`

```
POST /tools/invoke
  ├→ authorizeGatewayBearerRequestOrReply() [Bearer token 检查]
  ├→ loadConfig()
  ├→ resolveEffectiveToolPolicy() [聚合策略]
  ├→ createOpenClawTools() [完整工具列表]
  ├→ applyToolPolicyPipeline() [安全过滤]
  ├→ Gateway HTTP 拒绝列表过滤
  ├→ 查找工具
  ├→ runBeforeToolCallHook() [执行前验证]
  ├→ tool.execute(callId, args)
  └→ 返回结果
```

### 4.4 通道与发送者授权

**DM 策略:**
- `pairing`: 未知发送者获得配对码，默认阻断
- `open`: 接受所有 DM (需要 `"*"` 在允许列表中)
- `allowlist`: 严格允许列表

**群组策略:**
- `allowlist`: 需要显式允许列表
- `open`: 接受所有群组
- `disabled`: 阻断所有群组

---

## 5. 执行安全 — 沙箱与审批

### 5.1 执行安全模型

```typescript
ExecSecurity = "deny" | "allowlist" | "full"
ExecAsk = "off" | "on-miss" | "always"
```

**默认行为 (关键!)**: `agents.defaults.sandbox.mode = "off"` → 宿主优先执行。

### 5.2 执行审批系统

文件: `src/infra/exec-approvals.ts`

- 审批文件: `~/.openclaw/exec-approvals.json`
- 绑定维度: argv、cwd、agentId、sessionKey、envHash
- 文件操作数哈希: 首个可发现的本地脚本获取 SHA256 快照
- 决策: `allow-once` | `allow-always` | null
- 默认超时: 30 分钟

### 5.3 Docker 沙箱

启用时的默认配置：
- `readOnlyRoot: true`
- `network: "none"`
- `capDrop: ["ALL"]`
- `tmpfs: ["/tmp", "/var/tmp", "/run"]`

**环境变量清洗** (`sanitize-env-vars.ts`):

阻断模式: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `*_TOKEN$`, `*_PASSWORD$` 等
允许模式: `LANG`, `PATH`, `HOME`, `USER`, `SHELL`, `TERM`, `TZ`, `NODE_ENV`
值验证: 拒绝空字节、拒绝 >32KB 值、警告 base64 模式
