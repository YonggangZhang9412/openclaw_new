# Part 2 (上): OpenClaw 架构分析 — 系统设计与执行模型

---

## 1. 架构总览

OpenClaw 是一个基于 **TypeScript/Node.js** 的个人 AI Agent 框架，允许用户部署持久运行的 Agent，通过即时通讯应用交互，拥有对邮件、日历、文件系统、浏览器、Shell 的广泛访问权限。

### 1.1 信任模型 — 核心假设及其后果

通过对 OpenClaw 的 Gateway 认证逻辑 (`src/gateway/auth.ts`)、Session 路由 (`src/sessions/`)、插件加载器 (`src/plugins/loader.ts`) 的代码分析，可以明确其安全架构建立在一个 **单操作者信任模型** 之上：

> **OpenClaw 将每个 Gateway 实例视为单一操作者的私有空间。所有通过认证的调用者共享同一信任域，不存在调用者之间的隔离边界。**

这个模型在传统软件中是合理的——一个人的笔记本电脑上，所有程序共享同一用户权限。**但在自主 Agent 系统中，它引入了一个根本性问题: Agent 本身可以被外部数据操纵，而它又拥有操作者的全部权限。** 换句话说，攻击者不需要入侵操作者的账户——只需要让 Agent 处理一封精心构造的邮件或访问一个恶意网页，就等同于获得了操作者的完整权限。

具体体现和后果：

| 设计选择 | 后果 |
|----------|------|
| Session ID 是路由控制，不是权限边界 | 一个被注入的 Session 可以访问其他 Session 的数据，没有隔离墙 |
| 插件共享 `process.env` 和 Node.js 运行时 | 一个恶意插件可以窃取所有 API 密钥和凭证，包括与它功能无关的密钥 |
| 已安装插件以 Gateway 权限运行 | 插件能做 Gateway 能做的一切：读写文件、执行命令、发送消息 |
| `~/.openclaw/` 修改者被视为可信 | Agent 若被诱导写入该目录，等同于获得了永久的配置级控制权 |

**核心矛盾**: 该模型假设 "操作者的意图始终一致"，但 AI Agent 的行为可以被其处理的数据改变——这使得 "操作者的代理" 可能变成 "攻击者的代理"，而系统无法区分两者。

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
OpenClaw (Gateway-Primary, 同步请求-响应):

  消息通道 (WhatsApp/Telegram/Slack/Discord/...)
      ↓
  Gateway (WebSocket 控制面 @ 127.0.0.1:18789)
      ├─ Pi Agent 运行时 (RPC 模式)
      ├─ CLI 接口
      ├─ WebChat UI
      ├─ Control UI
      └─ Canvas Host (A2UI)
```

**与 ShadowClaw 通信流的关键差异:**

ShadowClaw 采用 **EventBus-Primary 的混合架构**，Gateway 退化为薄传输层:

```
ShadowClaw (EventBus-Primary, 异步事件驱动):

  消息通道 (Telegram/Discord/...)
      ↓
  EventSource (TelegramSource/DiscordSource, 各自后台线程)
      ↓ 产生 Event(type="channel.message_received", frozen=True)
  EventInjectionGate (安全准入: 类型/速率/载荷)
      ↓
  EventBus PriorityQueue
      ↓
  EventConsumer._process_batch() → Agent 处理
      ↓ 产生 delivery.enqueue 事件
  DeliveryEventConsumer (SideConsumer)
      ↓
  Channel.send() → Telegram sendMessage API
```

**架构含义**: OpenClaw 的 Gateway 是消息的中心枢纽，所有通道都汇入 Gateway。ShadowClaw 的 EventBus 是事件的中心枢纽，Gateway 仅负责 WebSocket 管理，而通道消息通过各自的 EventSource 直接进入 EventBus。这意味着 ShadowClaw 中所有消息——无论来自用户、定时器、还是文件系统——都经过 **同一个安全管道** (EventInjectionGate + CapabilityGate)。

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

**与 ShadowClaw Skill 扫描的实质对比:**

经过代码级对比，两者在扫描能力上 **功能等价**:

| 维度 | OpenClaw | ShadowClaw |
|------|----------|------------|
| 扫描时机 | 安装时 (加载前) | 安装时 (加载前) |
| 是否阻断加载 | 否 (仅警告; hook 可选阻断) | 否 (仅警告) |
| 逐行规则数 | 4 条 (exec/eval/mining/network) | 4 条 (几乎相同) |
| 全文件规则数 | 4 条 (外泄/混淆/env收割) | 4 条 (几乎相同) |
| 扫描上限 | 500 文件, 1MB/文件 | 500 文件, 1MB/文件 |
| 证据截断 | 120 字符 | 120 字符 |

**关键洞察**: 扫描层本身几乎没有差异。两个系统都是启发式检测 + 警告。**真正的安全差异不在扫描，而在扫描之后** — OpenClaw 中通过扫描的恶意代码获得完整的进程内权限和所有密钥访问；ShadowClaw 中即使恶意 Skill 通过扫描，CapabilityGate 仍然限制它能调用的工具和能触及的文件路径。扫描是第一道防线，但不是唯一防线。

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
