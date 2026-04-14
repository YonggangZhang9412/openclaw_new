# Part 2 (下): OpenClaw 架构分析 — 安全模块与已知局限

---

## 6. 安全模块

### 6.1 外部内容保护

文件: `src/security/external-content.ts`

**强项 — 边界标记系统：**
- 使用唯一随机 ID 的 XML 式边界标记包裹不可信内容
- 检测 12+ 种可疑 Prompt Injection 模式
- 防御 Unicode 同形字欺骗边界标记
- 剥离可能分割标记的不可见格式字符
- 注入安全警告通知 Agent 内容不可信

```
外部内容源类型:
email | webhook | api | browser | channel_metadata
| web_search | web_fetch | unknown
```

**局限 — 仅检测不阻断：**
- `detectSuspiciousPatterns()` 记录匹配但仍包裹内容传递给 LLM
- SECURITY NOTICE 告诉 LLM 忽略注入，但 LLM **不保证可靠遵从**
- 对高置信度注入 (如 "ignore all previous instructions") 没有硬阻断

### 6.2 危险配置标志

文件: `src/security/dangerous-config-flags.ts`

追踪操作者启用的风险标志：
- `gateway.controlUi.dangerouslyDisableDeviceAuth`
- `gateway.controlUi.allowInsecureAuth`
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback`
- `hooks.gmail.allowUnsafeExternalContent`
- `tools.exec.applyPatch.workspaceOnly=false`

### 6.3 危险工具定义

文件: `src/security/dangerous-tools.ts`

```
HTTP 拒绝列表 (Gateway HTTP 调用始终拒绝):
  sessions_spawn    — 远程 Agent 生成 (RCE)
  sessions_send     — 跨会话消息注入
  cron              — 自动化控制面变更
  gateway           — Gateway 重配置
  whatsapp_login    — 交互式终端

ACP 危险工具:
  exec, spawn, shell, sessions_spawn, sessions_send,
  gateway, fs_write, fs_delete, fs_move, apply_patch
```

### 6.4 安全审计工具

文件: `src/security/audit.ts` (1,504 行)

`openclaw security audit [--deep]` 提供：
- Gateway 认证检查
- DM 策略检查
- 执行安全检查
- 沙箱配置检查
- 工具策略检查
- 文件系统检查
- 通道安全检查

### 6.5 Skill 扫描器

文件: `src/security/skill-scanner.ts`

逐行规则：
- `dangerous-exec`: subprocess/os 调用 → CRITICAL
- `dynamic-code-execution`: eval()/exec()/compile() → CRITICAL
- `crypto-mining`: stratum+tcp, coinhive → CRITICAL
- `suspicious-network`: 非标准端口 → WARN

全文件规则：
- `potential-exfiltration`: 文件读取 + 网络发送组合 → WARN
- `obfuscated-code`: hex/base64 混淆 → WARN
- `env-harvesting`: os.environ + 网络发送 → CRITICAL

**限制**: 最大扫描 500 文件，>1MB 文件被跳过，扫描是异步的且不阻止加载。

---

## 7. 密钥管理

### 7.1 SecretRef 框架

文件: `src/config/types.secrets.ts`

三种密钥源：

| 源 | 机制 | 说明 |
|----|------|------|
| env | process.env 读取 | 环境变量 |
| file | 文件读取 | singleValue 或 JSON 模式 |
| exec | 命令执行 | 运行命令捕获 stdout |

环境变量名验证: `^[A-Z][A-Z0-9_]{0,127}$`

### 7.2 密钥运行时

文件: `src/secrets/runtime.ts`

- `prepareSecretsRuntimeSnapshot()` 在 Agent 执行前解析所有匹配配置的密钥
- `clearActiveSecretsRuntimeState()` 仅在显式调用时清除

**关键问题**: 所有解析后的密钥对所有 Agent 代码可用，没有每插件的密钥范围限制。

### 7.3 凭证存储

- Web 提供商凭证: `~/.openclaw/credentials/`
- 认证配置存储: `src/agents/auth-profiles.ts`
- API 密钥掩码: `src/utils/mask-api-key.ts`

---

## 8. 配置系统

### 8.1 配置加载

- 从 `~/.openclaw/config.json5` 或 `$OPENCLAW_CONFIG_DIR` 加载
- Zod schema 验证
- 环境变量覆盖
- 配置迁移 (旧设置格式)

### 8.2 安全相关配置选项

```
agents.<id>.dmPolicy:      "pairing" | "open" | "allowlist"
agents.<id>.groupPolicy:   "allowlist" | "open" | "disabled"
tools.allow/deny:          全局工具策略
agents.<id>.tools.*:       每 Agent 工具策略
tools.byProvider.*:        每模型提供商策略
tools.subagents.*:         子 Agent 策略
sandbox.docker.enabled:    Docker 沙箱开关
gateway.trustedProxies:    可信代理列表
```

---

## 9. 部署安全

### 9.1 Docker 部署

- 非 root 执行: `USER node`
- 权限硬化: `--chown=node:node`
- 基础镜像: `node:24-bookworm` + SHA256 固定
- CLI 容器: `cap_drop: [NET_RAW, NET_ADMIN]`, `no-new-privileges:true`
- GPG 指纹验证 Docker apt key

### 9.2 沙箱系统依赖

- 可选: Playwright/Chromium (浏览器自动化)
- 可选: Docker CLI (沙箱管理)

---

## 10. OpenClaw 设计模式与优势

### 10.1 优势

| 方面 | 描述 |
|------|------|
| 显式信任边界 | SECURITY.md 文档化操作者信任模型，明确范围外定义 |
| 纵深防御 (执行) | deny → allowlist → approval → ask 多层执行安全 |
| 沙箱能力 | Docker 隔离 + 环境清洗 + tmpfs |
| 审计可见性 | `openclaw security audit --deep` 综合检查 |
| 通道灵活性 | 25+ 消息平台，每通道允许列表 + 配对 + DM 策略 |
| 插件生态 | 89+ 捆绑扩展，丰富的功能覆盖 |

### 10.2 核心局限 (安全视角)

| 方面 | 风险级别 | 说明 |
|------|----------|------|
| 插件隔离 | CRITICAL | 无进程沙箱，所有插件共享 V8 上下文 |
| 代码签名 | CRITICAL | 插件未签名，扫描器检测但不阻止 |
| 密钥范围 | CRITICAL | 所有解析密钥对所有 Agent 代码可用 |
| 成本控制 | CRITICAL | 无 API 预算、速率限制或成本估算 |
| 注入防御 | HIGH | 模式检测无硬执行，LLM 可被社会工程 |
| 审批范围 | HIGH | 仅执行需要审批，高级操作缺乏检查 |
| 内存中密钥 | HIGH | 整个 Agent 生命周期内密钥可用，无过期 |
| 上下文漂移 | MEDIUM | 无显式上下文溢出防御或意图漂移检测 |
| 审计不可变性 | MEDIUM | 本地日志可编辑/删除，无防篡改机制 |

这些局限将在 Part 3 中与 ShadowClaw 的对应机制进行详细对比。
