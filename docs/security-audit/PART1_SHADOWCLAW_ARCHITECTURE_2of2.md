# Part 1 (下): ShadowClaw 架构深度分析 — 安全核心与执行管道

---

## 4. CapabilityGate 三层安全架构 (s26)

这是 ShadowClaw 最核心的安全创新，共 4,716 行纯 Python 代码，**不包含任何 LLM 调用**。

### 4.1 设计哲学

```
OpenClaw 模型:  Agent = User → 继承全部权限 → 事后检查
ShadowClaw 模型: Agent = 零权限 → JIT 最小授权 → 事前验证
```

### 4.2 Layer 1: CapabilityToken — JIT 最小权限令牌

每个事件批次获得一个不可变(frozen)的能力令牌：

| 属性 | 说明 | 典型值 |
|------|------|--------|
| granted_tools | 允许的工具集 (frozenset) | 2-5 个 (vs OpenClaw 45+) |
| denied_tools | 显式拒绝列表 (优先于授权) | bash, curl |
| granted_paths | 文件路径 glob 模式 | /workspace/memory/* |
| max_taint_for_args | 参数最大污染等级 | TaintLevel.USER |
| ttl | 生存时间 (秒) | 300 (自动过期) |
| allows_untrusted_input | Rule of Two 标志 | True/False |
| allows_sensitive_data | Rule of Two 标志 | True/False |
| allows_external_action | Rule of Two 标志 | True/False |

**关键特性：**
- `frozen=True`: 令牌签发后不可修改，免疫运行时篡改
- TTL 自动过期: 300-600 秒后权限自动撤销
- `check_rule_of_two()`: 三标志约束，最多两个为 True

### 4.3 CapabilityIssuer — 动态令牌签发

签发流程：
1. 事件类型 → `EVENT_TOOL_GRANTS` 字典查找最小工具集
2. `_classify_event_trust()` 分类信任等级:
   - `LOCAL_TRUSTED`: 内部定时器、cron、文件监控
   - `REMOTE_VERIFIED`: 已认证 WebSocket 连接
   - `REMOTE_OPEN`: 无内部模块标记的自定义事件
3. 信任等级 → 工具裁剪: REMOTE_OPEN 移除更多高风险工具
4. 路径从事件载荷推断 (如 file.modified → 该文件路径)

### 4.4 Layer 2: TaintStore — 值级污染追踪

```
TaintLevel 枚举:
  USER (0)     ← 最可信 (用户直接输入)
  INTERNAL (1) ← 系统生成 (文件、数据库)
  EXTERNAL (2) ← 最不可信 (网页、邮件、API)
```

**追踪机制：**
- **指纹注册**: 工具返回数据 → SHA-256 哈希(前16字符) → 存入 TaintStore
- **子串索引**: 从大内容(>5KB)中提取邮箱、URL、文件路径，单独索引
- **查询策略 (短路优先)**:
  1. 边界标记检测: 内容是否被 `wrap_external_content()` 包裹
  2. 精确匹配: 内容指纹查找
  3. 子串匹配: 高污染子串是否存在于内容中
  4. 默认: 返回 None (未知来源)

**CausalTaintTracker — 因果污染追踪：**
- 追踪 LLM 上下文是否曾接触 EXTERNAL 数据
- 一旦污染，**不可逆** (单调递增)
- 双层防御: 内容级(TaintStore) + 因果级(CausalTaintTracker)

**容量管理**: 最大 10,000 条记录，TTL 600 秒自动清理，LRU 淘汰。

### 4.5 Layer 3: CapabilityGate — 确定性三重验证

纯 Python 实现，**零 LLM 依赖**，免疫 Prompt Injection：

**Check 1 — 令牌验证:**
- TTL 是否过期
- 工具是否在 granted_tools 中
- 工具是否在 denied_tools 中 (拒绝优先)

**Check 2 — 值级污染检查:**
- 遍历工具参数的每个值
- 查询 TaintStore 获取污染等级
- 比对工具策略的 `max_arg_taint` 和 `sensitive_params`
- 例: send_email 的 `to` 参数要求 taint ≤ USER，EXTERNAL 数据被阻断

**Check 3 — 结构性约束:**
- 路径范围: 参数路径是否在 token.granted_paths 内
- 频率限制: 每轮最多 20 次工具调用
- Rule of Two: 三条件(不可信输入 + 敏感数据 + 外部动作)最多满足两个

### 4.6 Rule of Two — 结构性攻击链阻断

三个互斥的安全条件：

| 条件 | 含义 | 示例 |
|------|------|------|
| 不可信输入 | 来自 web_search、邮件等 | 读取网页内容 |
| 敏感数据 | 匹配 .env、.ssh、*.key 等 | 读取密钥文件 |
| 外部动作 | send_email、bash、curl 等 | 发送邮件 |

**约束: 任意两个可共存，第三个被阻断。**

- 读网页 + 读 .env → 不能发邮件 (阻断数据外泄)
- 读网页 + 发邮件 → 不能读密钥 (阻断凭证窃取)
- 读密钥 + 发邮件 → 不能读网页 (阻断注入触发)

**双层执行**: 令牌签发时静态检查 + 批处理运行时动态检查(因果级)。

### 4.7 工具风险分级

基于 DEFAULT_TOOL_POLICIES (s26:538-683) 中已定义的工具:

```
LOW:      read_file, list_directory, memory_search, sandbox_variables
MEDIUM:   write_file, edit_file, execute_code
HIGH:     curl, wget, send_email, daemon_restart, security_scan
```

注: 工具风险通过 ToolPolicy 的多个属性组合表达 (is_external_action, is_sensitive_reader, max_arg_taint 等)，而非单一 "risk_level" 字段。例如 send_email 的 `to`/`cc`/`bcc` 参数被标记为 sensitive_params，curl 被标记为 is_external_action。

---

## 5. 完整执行管道

从外部事件到工具执行的完整路径：

```
[1] 外部事件 (如 file.modified)
     ↓
[2] EventInjectionGate.verify_injection()
     → 类型白名单 / 速率限制 / 载荷大小 / 级联深度
     ↓
[3] EventFilter.receive()
     → 去重 / 去抖 / 优先级路由
     ↓
[4] PriorityQueue (堆排序)
     ↓
[5] EventConsumer._process_batch()
     ├─ 加载会话上下文
     ├─ CapabilityIssuer.issue_for_events() → CapabilityToken
     ├─ filter_tools_by_token() → 仅传递授权工具给 LLM
     ├─ LLM 调用 (仅能看到令牌授权的工具)
     ├─ LLM 返回 tool_call
     ├─ CapabilityGate.verify()
     │    ├─ Check 1: 令牌验证
     │    ├─ Check 2: 污染检查
     │    └─ Check 3: 结构约束
     ├─ DENY → 错误返回 LLM (LLM 可调整策略)
     └─ ALLOW → 执行工具 → 注册结果污染 → 继续循环
```

**关键点**: LLM 在 Step 5 中只能看到令牌授权的 2-5 个工具，而非全部 45+ 个。即使 LLM 被 Prompt Injection 攻击，它也无法调用未授权的工具。

---

## 6. 输入验证与反注入

### 6.1 外部内容包裹 (s12_security.py)

```
<<<EXTERNAL_UNTRUSTED_CONTENT id="[16位随机hex]">>>
[内容]
<<</EXTERNAL_UNTRUSTED_CONTENT id="[16位随机hex]">>>
```

- 随机 ID (2^64 可能性) 防止标记欺骗
- 内部标记被 `_sanitize_markers()` 替换为 `[[MARKER_SANITIZED]]`
- 包含安全警告通知 Agent 内容不可信

### 6.2 Unicode 安全

- **同形字折叠**: 全角 `＜` → ASCII `<`，CJK `〈` → `<`
- **不可见字符清除**: 13 类零宽字符被剥离
- 防止: `<<<EXTER​NAL_...` (零宽连接符破坏正则)

### 6.3 可疑模式检测

9 个正则模式检测常见注入: "ignore all previous instructions"、"you are now a..."、"system: prompt override" 等。

### 6.4 安全正则编译

`compile_safe_regex()` 拒绝嵌套重复量词，防止 ReDoS 攻击。测试窗口限制 2048 字符。

---

## 7. 密钥与设备认证

### 7.1 密钥管理 (s15_secrets.py)

- **目标注册表**: 定义哪些配置路径可以包含密钥
- **三种形态**: SECRET_INPUT (`{"$secret": "KEY"}`)、环境变量引用 (`${ENV_VAR}`)、认证配置
- **深度审计 7 码**: PLAINTEXT_FOUND / REF_UNRESOLVED / REF_SHADOWED / LEGACY_RESIDUE 等
- **迁移计划**: 明文 → SecretRef 的原子写入，支持回滚

### 7.2 设备认证 (s39_device_auth.py)

- **HMAC 挑战-响应**: 每设备 32 字节随机密钥
- **恒定时间比较**: `hmac.compare_digest()` 防止时序攻击
- **未知设备保护**: 使用 `_DUMMY_SECRET` 占位符，所有路径做相同计算量
- **时钟偏差容忍**: 300 秒防重放
- **原子写入**: `O_EXCL | O_CREAT` 防止 TOCTOU 竞态
- **文件权限**: 0600 仅当前用户可读

---

## 8. 审计与可观测性

### 8.1 审计日志

capability_gate_audit 和 eventbus_audit 两个 Side Consumer 记录所有门控决策和事件总线操作。InjectionGateDecision 结构包含:

- decision_id (UUID hex [:12])
- allowed (bool)
- source_id, event_type, priority, reason
- timestamp

CapabilityGate 内部维护 `_denial_log` 列表和 `_call_counts` 字典，每轮使用 round_id (UUID hex [:8]) 进行关联追踪。审计日志容量上限 10,000 条 (FIFO)。

### 8.2 日志脱敏 (s18_logging.py)

- 17 种内置模式: sk-xxx, ghp_xxx, Bearer tokens, PEM keys 等
- 掩码规则: 保留前 6 + 后 4 位，中间省略
- 分块处理: 防止大文件上的 ReDoS

### 8.3 诊断心跳

每 30 秒报告: 活跃/等待/排队会话数、工具调用历史(循环检测)、Webhook 统计、死锁检测(处理 >120s)。

---

## 9. 资源管理

### 9.1 令牌 TTL

每个事件批次的令牌有效期 300-600 秒。过期后必须签发新令牌。

### 9.2 频率限制

每轮最多 20 次工具调用 (`max_calls_per_round`)，防止失控循环。

### 9.3 上下文压缩 (s25)

当上下文接近令牌限制时: 嵌入早期对话 → 语义聚类 → 替换为事实摘要 → 回收 30-50% token。

### 9.4 自愈错误恢复

```
连续错误 2 次 → sanitize_session (修复消息链)
连续错误 4 次 → trim_session (保留最近 20 条)
连续错误 6 次 → reset_session (从空白开始)
```

---

## 10. 架构设计模式总结

| 模式 | 应用 |
|------|------|
| 责任链 | EventFilter → PriorityQueue → EventConsumer |
| 策略模式 | TaintStore 查询: 边界匹配 → 指纹 → 子串 |
| 装饰器 | wrap_external_content 包裹不可信字符串 |
| 工厂 | CapabilityIssuer 根据事件类型创建令牌 |
| 不可变值对象 | CapabilityToken frozen=True |
| 仅追加日志 | FIFO 审计日志 (10,000 条上限) |
| 观察者/发布订阅 | 11 个 Side Consumer 订阅事件流 |
| 渐进式分层 | 30 层模块，每层增加能力 |
