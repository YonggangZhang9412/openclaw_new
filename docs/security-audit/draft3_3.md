## 3. The Event-Driven Capability Architecture

The Agent Authority Problem shows that C1, C2, and C3 cannot coexist under ambient authority. We replace ambient authority with **event-scoped capability authority** — not solving the impossibility within the old model, but replacing the model itself, analogous to how parameterized queries replaced string concatenation for SQL security.

Under event-scoped capability authority, all three conditions are simultaneously satisfied: C1 preserved (agent processes untrusted data autonomously), C2 preserved (agent accesses sensitive resources, but scoped per event batch), C3 satisfied (CapabilityGate is deterministic code with zero LLM dependency).

### 3.1 Design principles

**Principle 1 (Heterogeneous enforcement).** The security layer must operate through a channel immune to natural language manipulation: deterministic code, not LLM judgment.

**Principle 2 (Context-scoped authority).** Authority must be scoped to each processing context — not granted as a session-wide ambient capability.

**Principle 3 (Structural chain interruption).** Rather than detecting all adversarial inputs, structurally prevent completion of attack chains regardless of injection success.

### 3.2 Dual-gate architecture on a unified EventBus

These principles impose requirements on the underlying architecture that necessitate both an event-driven model and a **dual-gate security structure** (Fig. 2).

**EventBus.** All triggers — user messages, timers, file changes, cron jobs, network events — are normalized into typed Event objects (frozen dataclasses with immutable `source`, `origin_chain`, `cascade_depth`). This unified event model provides four structural properties unachievable in request-response architectures:

- **Requirement A (Event provenance).** Each event carries immutable metadata identifying its origin and trust level, set by the framework — not by the caller. A request-response model can authenticate *caller identity* (via JWT, mTLS) but cannot represent *event provenance* — what kind of trigger initiated the processing context.
- **Requirement B (Cascade depth enforcement).** The framework auto-increments `cascade_depth` when consumers inject child events; a hard limit of 20 prevents infinite loops. In request-response models, this requires manual propagation by each handler — a single non-compliant handler breaks the guarantee.
- **Requirement C (Batch atomicity).** Correlated events are processed as atomic batches, enabling the Rule of Two (Section 3.4) to evaluate across the entire batch. Per-request security decisions would fragment the analysis window.
- **Requirement D (Composable monitoring).** 18 independent side consumers — security scanning, capability gate auditing, error aggregation, delivery monitoring — subscribe to the event stream in parallel without mutual coupling.

**Gate 1: EventInjectionGate (producer-side).** Before an event enters the EventBus, the EventInjectionGate enforces per-source security policies:
- **Rate limiting**: sliding 1-second window per source (ADMIN: 200/s, CLI: 100/s, MOBILE: 10/s)
- **Payload size cap**: 16KB (MOBILE) to 131KB (ADMIN) per event
- **Cascade depth enforcement**: enforces Requirement B at the point of event injection, rejecting events that exceed the depth limit
- **Event type whitelist**: external sources may only inject event types permitted by their policy
- **CRITICAL priority control**: requires an unforgeable EscalationToken (UUID-based, time-bounded, use-limited)

The InjectionGate operates entirely before the LLM is invoked. Its decisions depend on event metadata and source identity — not on content processed by the LLM. This constitutes a security layer that is orthogonal to and independent of the CapabilityGate.

**Gate 2: CapabilityGate (consumer-side).** After the EventConsumer invokes the LLM and receives a tool-call proposal, the CapabilityGate performs three-layer verification (Section 3.3).

**Dual-gate security implication.** An attacker must bypass both gates to complete an attack: first, the event must pass the InjectionGate (rate, size, type, depth constraints); then, the LLM's tool call must pass the CapabilityGate (token, taint, Rule of Two). The Theorem 1 probability bound (Section 4) quantifies only the CapabilityGate's contribution; the InjectionGate provides additional unquantified barriers, making the bound conservative.

> **[Figure 2]** Dual-gate event-driven capability architecture. **(a)** All triggers normalized into typed Events flowing through a unified EventBus. **(b)** Security pipeline: Gate 1 (EventInjectionGate: rate limiting, cascade depth, payload cap, type whitelist) → EventFilter (debounce, deduplication) → PriorityQueue → EventConsumer issues JIT CapabilityToken → LLM invocation with only granted tools visible → Gate 2 (CapabilityGate: token check, taint check, Rule of Two). **(c)** 18 independent side consumers operate in parallel for monitoring, auditing, and anomaly detection.

### 3.3 Trust-level-specific token issuance

The CapabilityIssuer classifies each event batch into one of three trust levels based on immutable event metadata, then issues a token with trust-appropriate tool grants:

| Trust level | Criteria | Tool grant policy | Security consequence |
|-------------|----------|-------------------|---------------------|
| LOCAL_TRUSTED | Internal sources (timer, cron, file, process, network) | Full tool set relevant to event type | Standard Theorem 1 bound applies |
| REMOTE_VERIFIED | Authenticated device via HMAC challenge-response | High-risk tools removed (bash, daemon_restart) | $\alpha_R$, $\alpha_X$ reduced; tighter bound |
| REMOTE_OPEN | Unverified external source (custom webhook, unauthenticated WS) | All external-action tools removed (bash, curl, wget, send_email, run_command) | $\alpha_X = 0 \Rightarrow \Pr[\text{Exfil}] = 0$ |

**Corollary (Trust-Level Zero Guarantee).** For REMOTE_OPEN events, no external-action tool is granted ($\alpha_X$ = 0). Since exfiltration requires transmitting data externally, and no external-send tool is available, exfiltration is structurally impossible regardless of injection success rate, Rule of Two status, or taint evasion capability. As we formalize in Theorem 1 (Section 4): $\Pr[\text{Exfil} \mid \text{REMOTE\_OPEN}] \leq p \cdot \min(\alpha_R, 0) \cdot (1-R_2) \cdot q = 0$.

This guarantee is strictly stronger than the Rule of Two (Theorem 3, Section 4), which requires $R_2 = 1$ and applies only within a single batch. The trust-level guarantee holds regardless of $R_2$ and regardless of cross-batch contamination.

### 3.4 Three-layer CapabilityGate

**Layer 1 (Token scoping).** Each event batch receives a JIT CapabilityToken — a frozen object specifying `granted_tools` (frozenset of 2-5 tools), `granted_paths` (filesystem globs derived from event payload), and `ttl` (300s). The LLM sees only schemas for granted tools; ungranted tools are invisible.

**Layer 2 (Taint tracking: registration, propagation, verification).**

Taint tracking operates in three stages:

*Registration.* When a tool returns external data (e.g., web_fetch result), the framework — before the LLM sees the result — wraps the content in cryptographically tagged boundary markers (random 16-hex ID, $2^{64}$ marker space) and registers it in TaintStore with a SHA-256 fingerprint and taint level EXTERNAL. The wrapping process includes marker sanitization (preventing nested boundary spoofing) and Unicode normalization (preventing homoglyph attacks that bypass marker detection). Implementation details are described in Methods.

*Propagation.* CausalTaintTracker maintains a monotonically non-decreasing taint level per batch (Theorem 4): once EXTERNAL data is observed, the batch is permanently marked as contaminated.

*Verification.* When the LLM proposes a tool call, CapabilityGate Check 2 queries TaintStore (content-level: fingerprint and substring matching) and CausalTaintTracker (context-level: round taint $\leq$ policy threshold). Both must pass for the call to proceed.

**Layer 3 (Rule of Two).** Exfiltration requires three simultaneous conditions: untrusted input (U), sensitive data access (S), and external action (X). The Rule of Two constrains the token so at most two of {U, S, X} hold per batch. By Theorem 3, $\Pr[\text{Exfil}] = 0$ when enforced. The Rule of Two is enforced by default for all batches. The $R_2 = 0$ case in the quantitative analysis (Section 4.3) represents a conservative worst-case scenario for theoretical completeness — for example, if the tool classification $\mathcal{T}_U, \mathcal{T}_R, \mathcal{T}_X$ is incomplete and a tool is miscategorized, the Rule of Two may fail to detect the triple condition. The $R_2 = 0$ bound characterizes the architecture's residual security when this particular layer is ineffective.

### 3.5 Security properties

**Determinism and prompt injection immunity.** The CapabilityGate function G is deterministic and LLM-independent (Theorem 5). An adversary who injects the LLM can influence *which* tool call is proposed, but cannot influence *how* G evaluates it.

**Automatic privilege decay.** Tokens expire after 300 seconds. Each batch starts from zero authority.

**Fail-safe defaults.** TaintStore is fail-open (unknown sources allowed), but CausalTaintTracker is fail-safe (contamination irrevocable within batch). Unmapped event types receive the most restrictive token.

**Auditable decisions.** Every Gate decision includes a machine-readable reason (e.g., "DENY: param 'to' taint=EXTERNAL, policy requires $\leq$ USER").

### 3.6 Case analysis: the architecture against the OpenClaw hazard topology

**Malicious supply-chain skills (12% of ClawHub compromised).** Under event-scoped capability authority, **token scoping** (Layer 1) restricts the skill's available tools to those relevant to its declared function, and **taint tracking** (Layer 2) registers all skill-introduced data as EXTERNAL, preventing its use as arguments to sensitive tools.

**CVE-2026-25253: cross-site WebSocket hijacking RCE.** Under our architecture, an unauthenticated WebSocket message is classified as REMOTE_OPEN. By the trust-level zero guarantee (Section 3.3), $\alpha_X = 0$ and $\Pr[\text{Exfil}] = 0$ — the RCE chain is structurally impossible because code execution tools are never granted for untrusted sources.

**Persistent hallucination loops and intent drift.** **Automatic privilege decay** ensures each batch receives a fresh token with 300-second TTL. Previous context errors do not accumulate privileges.

**Over-authorization (500 unsolicited iMessages).** **Context-scoped authority** (Principle 2) ensures that tools are granted per event type. A timer event does not warrant messaging tools. **EventInjectionGate rate limiting** provides an additional quantitative bound on event throughput.
