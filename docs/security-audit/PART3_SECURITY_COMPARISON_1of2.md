# Part 3 (上): 安全对比分析 — OpenClaw 的十大安全缺陷

> 基于 ShadowClaw (branch: claude/skill-system-eventbus-review-Z0qF6) 与 OpenClaw (main) 的深度对比

---

## 总体安全范式差异

| 维度 | OpenClaw | ShadowClaw |
|------|----------|------------|
| 权限模型 | 环境权限 (Ambient Authority) | 能力安全 (Capability-Based) |
| 默认权限 | Agent 继承用户全部权限 | Agent 默认零权限 |
| 授权粒度 | 会话级 (整个会话期间有效) | 请求级 (每事件批次 JIT 签发) |
| 安全决策者 | LLM + 配置策略 | 确定性 Python 代码 (无 LLM) |
| 数据流追踪 | 无 | 值级污染追踪 (TaintStore) |
| 攻击链阻断 | 无结构性保证 | Rule of Two 结构性约束 |
| 令牌生命周期 | 无 (持久化) | TTL 300-600s 自动过期 |
| 审计完整性 | 本地日志 (可篡改) | 哈希链审计日志 (防篡改) |

---

## 缺陷 1: 环境权限 vs 能力安全 — 根本性架构差异

### OpenClaw 的问题

OpenClaw 采用 **环境权限 (Ambient Authority)** 模型：Agent 被视为操作者的代理，继承操作者的全部权限。这意味着：

- Agent 在整个会话期间可以访问 **所有已授权工具** (45+)
- 工具可用性仅通过静态策略配置控制 (allow/deny 列表)
- 一旦会话建立，权限不随上下文变化而收缩
- LLM 始终能看到所有可用工具的 schema

**攻击场景**: 攻击者通过邮件向 Agent 发送包含 Prompt Injection 的内容。Agent 处理该邮件时，LLM 看到 45+ 个工具的完整列表，包括 `bash`、`fs_write`、`send_email`。注入的指令可以利用任何可见工具。

### ShadowClaw 的解决方案

**能力安全 (Capability-Based Security)**：

1. 每个事件批次获得 **JIT 签发的 CapabilityToken**
2. 令牌仅包含 **2-5 个相关工具** (vs 45+)
3. LLM 只看到令牌授权的工具 schema
4. 令牌 300s 后自动过期
5. 远程事件自动剥离高风险工具

**效果**: 即使 LLM 被注入，它也无法调用未在令牌中的工具。攻击面从 45+ 个工具缩减到 2-5 个。

---

## 缺陷 2: 插件隔离缺失 — 共享进程空间

### OpenClaw 的问题

所有插件通过 Jiti 动态加载，**共享同一 V8 上下文和进程环境**：

- 插件可以访问 `process.env` (所有环境变量，包括 API keys)
- 插件可以修改模块缓存
- 插件可以检查其他插件状态
- 插件在 Gateway 权限下运行
- 无进程级隔离、无 Worker Thread 隔离

**攻击场景**: 恶意插件伪装为 "翻译工具"，实际上在初始化时读取 `process.env.ANTHROPIC_API_KEY` 和 `process.env.OPENAI_API_KEY`，通过 HTTP 请求发送到攻击者服务器。因为没有进程隔离，这完全在插件的能力范围内。

**代码证据**: `src/plugins/loader.ts:149-177` — Jiti 创建隔离导入但共享 V8 上下文。

### ShadowClaw 的对比

- Skill 通过 SKILL.md 声明式定义，不直接执行代码
- 确定性执行路径使用 `shlex.quote()` 防注入
- LLM 中介路径通过 CapabilityGate 限制工具访问
- 代码执行通过 SandboxProcess 隔离 (512MB 内存, 30s 超时, 无 syscall, 无 env)
- 源码扫描器在加载前检测危险模式

---

## 缺陷 3: 无代码签名 — 供应链攻击

### OpenClaw 的问题

插件 **没有密码学签名验证**：

- `src/plugins/loader.ts:1-70` 通过 Jiti 动态加载，无签名检查
- 扫描器是启发式的 **事后检测**，不是事前防止
- 扫描器异步运行，结果不阻止加载 (除非设置 `throwOnLoadError`)
- 最大扫描 500 文件，>1MB 文件跳过

**攻击场景**: npm 供应链攻击修改了某个流行插件的依赖。修改后的代码通过 base64 编码 + split 逻辑规避扫描器。插件正常加载并执行恶意代码。

### ShadowClaw 的对比

- Skill 扫描器 (s12_security.py) 在加载前执行 6 层检查
- 检测: 动态代码执行、加密挖矿、可疑网络、环境变量收割、混淆代码
- 全文件分析: 检测 "文件读取 + 网络发送" 组合 (外泄模式)
- 审计报告集成: 所有发现记录到 SecurityAuditReport

---

## 缺陷 4: 密钥范围缺失 — 全局可见

### OpenClaw 的问题

所有解析后的密钥对整个 Agent 运行时可用：

- `src/secrets/runtime.ts:161-200` — `prepareSecretsRuntimeSnapshot()` 解析所有匹配配置的密钥
- 插件不声明需要哪些密钥，获取所有可用密钥
- 子 Agent 继承附件 (可包含凭证)
- 密钥在整个 Agent 生命周期内驻留内存，无过期
- 无密钥轮换/撤销机制 (需重启)
- `~/.openclaw/credentials/` 凭证文件未加密

**攻击场景**: 一个仅需要 Slack API 的插件实际上可以访问 ANTHROPIC_API_KEY、OPENAI_API_KEY、AWS_SECRET_ACCESS_KEY 等所有密钥。

### ShadowClaw 的对比

- `s15_secrets.py` — 目标注册表定义哪些配置路径可包含密钥
- SecretRef 对象: `{"$secret": "KEY"}` 引用而非明文
- 深度审计: 检测明文、未解析引用、遮蔽引用、遗留残留
- 设备认证: HMAC 密钥 0600 权限，原子写入防 TOCTOU
- 恒定时间比较: 防止时序侧信道

---

## 缺陷 5: 无成本控制 — 失控消耗

### OpenClaw 的问题

**无任何 API 调用预算或成本追踪机制：**

- 无每 Agent 或每用户配额
- 无执行前成本估算
- `sessions-spawn-tool.ts:88-90` — `runTimeoutSeconds` 可选且默认 undefined
- 工具执行可无限期挂起
- `web-fetch.ts:40` — `DEFAULT_FETCH_MAX_RESPONSE_BYTES = 2,000,000` (2MB 通过 LLM 处理代价高昂)

**攻击场景**: 被注入的 Agent 或循环引用导致反复调用 LLM API。一天内可能产生数百美元成本。用户直到收到账单才发现问题。

### ShadowClaw 的对比

- CapabilityToken TTL: 300-600 秒自动过期，限制时间窗口
- 频率限制: 每轮最多 20 次工具调用
- EventInjectionGate: 滑动窗口限速 (ADMIN 200/s, CLI 100/s, MOBILE 10/s)
- 载荷大小限制: 16KB-131KB 按设备类型
- 级联深度限制: 防止事件循环 (最大 20 层)
- 队列容量: 10,000 上限 + 智能淘汰
