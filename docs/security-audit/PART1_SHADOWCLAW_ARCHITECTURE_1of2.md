# Part 1 (上): ShadowClaw 架构深度分析 — 核心设计与分层体系

> 分析基于 branch: `claude/skill-system-eventbus-review-Z0qF6`

---

## 1. 架构总览

ShadowClaw 是一个基于 **EventBus 驱动 + 能力(Capability)安全模型** 的 Python AI Agent 框架。其核心设计哲学是：

> **Agent 默认拥有零权限，每一次操作都需要通过即时签发的能力令牌(CapabilityToken)获得最小授权。**

这与传统 Agent 框架（如 OpenClaw）的 "Agent = User，继承所有权限" 形成了根本性的对立。

### 1.1 代码规模

| 指标 | 数值 |
|------|------|
| 核心模块文件 | 40 个 (s01 ~ s40) |
| 总代码行数 | ~78,000 行 Python |
| 最大文件 | s23_eventbus.py (6,586 行) |
| 安全核心 | s26_capability_gate.py (4,716 行) |
| 日志系统 | s18_logging.py (3,520 行) |

### 1.2 渐进式分层架构 (s01 → s40)

ShadowClaw 采用了 **30+ 层渐进式模块架构**，每一层在前一层基础上增加能力：

**Phase 1: 基础 Agent (s01-s04)**
- `s01_agent_loop.py` — 基本对话循环，Agent 基类
- `s02_tool_use.py` — 45+ 工具定义与调度 (20,180 行)
- `s03_sessions.py` — JSONL 持久化会话状态
- `s04_multi_channel.py` — 多通道抽象 (CLI/File/Custom)

**Phase 2: 网关与人格 (s05-s06)**
- `s05_gateway.py` — WebSocket 服务器，JSON-RPC 路由 (2,111 行)
- `s06_soul_memory.py` — Agent 人格(SOUL.md)，语义记忆(TF-IDF + BM25)

**Phase 3: 定时与调度 (s07-s08)**
- `s07_heartbeat.py` — 心跳检测，可配置间隔 (默认30分钟)
- `s08_cron.py` — Cron 调度器 (at/every/cron 语法)，SQLite 执行日志

**Phase 4: 基础设施 (s09-s11)**
- `s09_node.py` — 远程设备客户端，设备配对，分布式事件上报
- `s10_skill.py` — Skill 模块系统 (2,529 行)，SKILL.md 前置声明 + 可执行体
- `s11_hooks.py` — 事件钩子，拦截消息/配置变更

**Phase 5: 安全审计层 (s12-s13)**
- `s12_security.py` — 6 层安全体系 (2,754 行)
- `s13_config.py` — 多阶段配置管道 (2,986 行)

**Phase 6: 密钥管理 (s15)**
- `s15_secrets.py` — 完整密钥生命周期管理 (3,121 行)

**Phase 7: 运行时控制面 (s14, s16-s20)**
- `s17_acp.py` — Agentic Control Plane (2,818 行)
- `s18_logging.py` — 7 级日志 + 自动脱敏 (3,520 行)
- `s19_daemon.py` — systemd/launchd 集成 (3,289 行)
- `s20_pairing.py` — 设备配对，挑战-响应认证 (2,366 行)

**Phase 8: 队列与投递 (s21-s22)**
- `s21_async_queue.py` — 有键异步队列
- `s22_delivery_queue.py` — 可靠消息投递 + 指数退避

**Phase 9: EventBus 统一架构 (s23)**
- `s23_eventbus.py` — 所有触发器的统一入口 (6,586 行)

**Phase 10: 代码沙箱 (s24)**
- `s24_code_execution.py` — 子进程隔离 (512MB 内存, 30s 超时, 无 syscall)

**Phase 11: 上下文管理 (s25)**
- `s25_context_compaction.py` — 嵌入式上下文压缩 (3,037 行)

**Phase 12: 安全门 (s26) — 核心安全层**
- `s26_capability_gate.py` — 三层安全验证 (4,716 行)

**Phase 13: 通道适配与扩展 (s27-s40)**
- Telegram、Discord、WhatsApp、Slack、飞书等通道适配器
- 合约系统、MCP 工作流、设备认证、QR 配对

---

## 2. EventBus 统一事件架构

### 2.1 事件数据模型

```
Event (frozen=True, 不可变)
├── id: str                    # UUID (12 hex)
├── source: str                # timer|cron|file|process|custom|node
├── type: str                  # 事件类型 (heartbeat_due, file_modified...)
├── priority: EventPriority    # CRITICAL(0) > HIGH(1) > NORMAL(2) > LOW(3)
├── payload: dict              # 事件特定数据
├── timestamp: float           # Unix 时间戳
├── debounce_key: str          # 去重键
├── origin_chain: tuple        # 传播路径 (安全审计)
├── cascade_depth: int         # 级联深度 (硬限制: 20)
└── injection_gate_decision_id # 审计追踪链接
```

### 2.2 七大事件源

| 事件源 | 触发器 | 用途 |
|--------|--------|------|
| TimerSource | 心跳间隔 | 周期性环境检查 |
| CronSource | 计划任务到期 | 定时报告、清理 |
| FileWatchSource | 文件系统变化 | 配置重载、代码分析 |
| ProcessWatchSource | 进程退出/启动 | 守护进程监控 |
| NetworkWatchSource | 端口状态变化 | 数据库重连 |
| SkillEventSource | Skill 轮询 | 外部数据变化检测 |
| CustomSource | 外部注入 | 第三方集成 |

### 2.3 EventInjectionGate — 事件注入安全门

这是 EventBus 的 **生产者侧安全层**，与 CapabilityGate（消费者侧）形成双层防御：

**5 层验证：**

1. **可信内部源判定** — 9 个内部源 (timer/cron/file/process/network/node/contract/task/skill_source) 可绕过外部限制
2. **事件类型白名单** — 外部源只能注入策略允许的事件类型
3. **级联深度限制** — 每策略最大 10 层，硬限制 20 层，防止事件循环
4. **载荷大小限制** — JSON 序列化大小检查 (默认 65KB)
5. **滑动窗口限速** — 每源 1 秒窗口，超限拒绝

**注入策略 (InjectionPolicy)：**

| 设备范围 | 速率限制 | 载荷限制 | 最高优先级 | 级联深度 |
|----------|----------|----------|------------|----------|
| ADMIN | 200 req/s | 131KB | CRITICAL | 8 |
| CLI | 100 req/s | 65KB | NORMAL | 5 |
| MOBILE | 10 req/s | 16KB | NORMAL | 3 |

**EscalationToken — CRITICAL 优先级升级令牌：**

CRITICAL 事件需要不可伪造的升级令牌，包含：token_id (UUID)、issued_by、reason、max_uses、ttl (300s)。令牌使用后递减剩余次数，防止滥用紧急升级。

### 2.4 三层事件过滤

```
Layer 1: EventInjectionGate  → 安全准入 (类型/速率/载荷/级联)
Layer 2: EventFilter         → 噪声控制 (去重/去抖/优先级路由)
Layer 3: PriorityQueue       → 公平调度 (CRITICAL 立即, LOW 缓存到心跳)
```

**EventFilter 策略：**
- **去重(Dedup)**: 同一 debounce_key 在 0.5s 窗口内合并
- **优先级路由**: CRITICAL/HIGH → 立即入队; NORMAL → 正常入队; LOW → 缓存到下次心跳
- **噪声过滤**: 跳过 .git/、node_modules/、临时文件

**队列管理 — 智能淘汰：**
- 队列满时 (10,000 上限)，比较新事件与最低优先级事件
- 新事件优先级更高 → 淘汰最低，插入新事件
- 新事件优先级更低 → 丢弃新事件 (保护高优先级)

### 2.5 Side Consumers — 11 个并行消费者

与主 EventConsumer 并行运行的侧消费者：

1. **ConfigWatcher** — file.modified → 重载配置 + 验证
2. **SecurityEventConsumer** — 任意事件 → 自动安全扫描
3. **CapabilityGateAuditConsumer** — 所有工具调用 → 不可篡改审计日志
4. **MemoryIndexer** — 会话消息 → 语义索引更新
5. **CronLogWriter** — cron.job → SQLite 日志
6. **NodeHeartbeatHandler** — node.heartbeat → 设备状态
7. **PairingChallengeValidator** — pairing.request → 挑战-响应
8. **WebhookStatusMonitor** — webhook.error → 重试/退避
9. **ToolCallCounter** — tool_call → 频率统计
10. **ProcessExitHandler** — process.exited → 自动重启
11. **ErrorAggregator** — any.error → 错误模式检测

---

## 3. Skill 系统架构

### 3.1 Skill 定义格式 (SKILL.md)

每个 Skill 通过前置声明(frontmatter) + Markdown 正文定义：

```yaml
---
name: weather
description: "通过 wttr.in 获取天气信息"
user-invocable: true
metadata:
  openclaw:
    requires:
      bins: ["curl"]          # 必须全部存在 (AND)
      anyBins: ["python3"]    # 至少一个存在 (OR)
      env: ["OPENWEATHER_KEY"] # 环境变量必须设置
event-triggers:
  - event-type: "file.modified"
    filter: { path: "workspace/weather-query.txt" }
    priority: "HIGH"
    cooldown: 300
command-dispatch: ""  # 空=LLM中介, "tool"=确定性执行
---
# 天气查询
使用 curl 获取天气数据...
```

### 3.2 四源优先级发现

```
优先级 (低→高):
  extra(0) → bundled(1) → managed(2) → workspace(3)

同名 Skill：高优先级覆盖低优先级
```

### 3.3 双执行路径

**路径 A: LLM 中介 (command-dispatch = "")**
- Skill 指令格式化为 prompt → LLM 自行决定调用什么工具
- 灵活但不可预测

**路径 B: 确定性执行 (command-dispatch = "tool")**
- 直接调用指定工具 (如 bash)
- 用户参数通过 `shlex.quote()` 防注入
- 模板替换: `curl wttr.in/{args}` → `curl wttr.in/'London'`
- 安全: 模板由管理员编写(可信)，参数被引用(不可信但已转义)

### 3.4 Skill 事件集成

- **P1-a: 事件触发** — Skill 声明事件触发条件，匹配时自动调用
- **P1-b: 热重载** — skills/ 目录变化时自动重新发现
- **P2: Skill 作为事件源** — Skill 定期轮询外部数据，变化时发射事件
