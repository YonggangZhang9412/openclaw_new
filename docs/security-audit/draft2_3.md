## 3. The Event-Driven Capability Architecture

The Agent Authority Problem shows that C1, C2, and C3 cannot coexist under ambient authority. Our resolution does not attempt to satisfy C1, C2, and C3 within the ambient authority model — the structural argument shows this is not possible. Instead, we **replace ambient authority with event-scoped capability authority**, a different authorization model under which C1, C2, and C3 can coexist.

This is analogous to how parameterized queries resolved SQL injection: not by making string concatenation safe, but by replacing the string-concatenation model with a structured-parameter model. The vulnerability was not "solved within" the old model — the model itself was recognized as the root cause and replaced. Similarly, ambient authority is not a law of nature for agent systems; it is a design choice inherited from traditional software, and the Agent Authority Problem demonstrates it is the wrong choice for systems whose intent is data-dependent.

Under event-scoped capability authority, all three conditions are simultaneously satisfied:

- **C1 preserved**: The agent continues to autonomously process untrusted data without per-item human approval. No functionality is sacrificed.
- **C2 preserved**: The agent retains access to sensitive resources, but this access is event-scoped (granted per event batch, for specific paths and tools, automatically revoked after 300 seconds) rather than ambient.
- **C3 satisfied**: The CapabilityGate is pure deterministic code with zero LLM dependency.

The key mechanism: the agent can access sensitive resources (C2) and process untrusted data (C1), but the event-scoped token ensures these never occur with the same authorization in a configuration enabling a complete attack chain.

### 3.1 Design principles

Three principles follow from the structural argument:

**Principle 1 (Heterogeneous enforcement).** The security layer must operate through a channel immune to natural language manipulation: deterministic code, not LLM judgment.

**Principle 2 (Context-scoped authority).** Authority must be scoped to each processing context — not granted as a session-wide ambient capability.

**Principle 3 (Structural chain interruption).** Rather than detecting all adversarial inputs, structurally prevent completion of attack chains regardless of injection success.

### 3.2 EventBus as structural necessity

These principles impose specific requirements on the underlying architecture that necessitate an event-driven model (Fig. 2).

**Requirement A (Event provenance).** Context-scoped authority requires knowing each event's origin and trust level through metadata that is set by the framework and immutable to consumers. A request-response model can authenticate *caller identity* (via JWT, mTLS), but cannot represent *event provenance* — what kind of trigger (user message, cron job, file change, webhook) initiated the processing context. This distinction requires a typed event model, not caller-callee exchanges.

**Requirement B (Cascade depth enforcement).** Agent actions trigger secondary events (file write → file-watch → agent action). Without structural depth limits, attackers induce infinite loops. In request-response models, cascade depth requires manual propagation by each handler — a single non-compliant handler breaks the guarantee. Our EventBus enforces depth increments at the framework level: consumers cannot circumvent the invariant.

**Requirement C (Batch atomicity).** Real-world triggers arrive in correlated batches. The Rule of Two (Section 3.3) must evaluate across entire batches to detect attack chain conditions. Per-request security decisions fragment the analysis window, creating TOCTOU race conditions.

**Requirement D (Composable monitoring).** Security monitoring must operate in parallel with agent execution without explicit coupling. Our EventBus supports 18 independent side consumers — security scanning, capability gate auditing, error aggregation — each processing events without mutual awareness.

> **[Figure 2]** Event-driven capability architecture. **(a)** All triggers (user messages, timers, file changes, cron jobs, network events) are normalized into typed Event objects (frozen dataclasses) flowing through a unified EventBus. **(b)** Security pipeline: EventInjectionGate (producer-side: type whitelist, rate limiting, cascade depth) → EventFilter (noise control: debounce, deduplication) → PriorityQueue → EventConsumer, which issues a JIT CapabilityToken and invokes the LLM with only the granted tools visible. **(c)** Each tool call passes through the three-layer CapabilityGate before execution.

### 3.3 The CapabilityGate: three-layer deterministic enforcement

**Layer 1 (Token scoping).** Each event batch receives a JIT CapabilityToken — a frozen object specifying `granted_tools` (frozenset of 2-5 tools), `granted_paths` (filesystem globs derived from event payload), and `ttl` (300s). The CapabilityIssuer classifies event trust as LOCAL_TRUSTED, REMOTE_VERIFIED, or REMOTE_OPEN, and progressively restricts tool grants for lower trust levels. The LLM sees only schemas for granted tools; ungranted tools are invisible.

**Layer 2 (Taint tracking).** Tool return values are registered in a TaintStore with origin and taint level (USER=0, INTERNAL=1, EXTERNAL=2). When the LLM uses a value as a tool argument, the Gate checks whether its taint exceeds the tool's policy. A complementary CausalTaintTracker marks the entire processing round as contaminated once any EXTERNAL data enters the LLM's context — providing a fail-safe backstop against LLM paraphrasing that evades content-level fingerprinting. TaintStore provides precision (parameter-level blocking); CausalTaintTracker provides recall (context-level blocking). Together, an attacker must evade both layers simultaneously.

**Layer 3 (Rule of Two).** We prove formally (Supplementary Note 2, Theorem 2) that every exfiltration attack — defined as a tool invocation sequence that causes sensitive data to cross the system boundary — necessarily requires three conditions: (1) untrusted input processed (U), (2) sensitive data accessed (S), (3) external action executed (X). The Rule of Two constrains the token so at most two of {U, S, X} hold simultaneously within a single event batch. By the contrapositive of Theorem 2, no exfiltration attack can complete under this constraint (see Section 7.3 for cross-batch scope limitations).

> **[Figure 3]** **(a)** Rule of Two constraint space: three conditions as a Venn diagram; any two may overlap, but the triple intersection (complete attack chain) is structurally excluded. **(b)** Attack chain comparison: under ambient authority, a single successful injection yields full tool access and completes the chain; under event-scoped capability, the chain must pass through five barriers (tool visibility, TTL, path scoping, taint check, Rule of Two), of which three are **deterministic** (tool visibility, TTL, path scoping — guaranteed by code invariants) and two are **structural** (taint tracking, Rule of Two — dependent on correct policy categorization).

### 3.4 Security properties

The architecture achieves five properties:

**Determinism and prompt injection immunity.** We prove (Supplementary Note 4, Theorem 5) that the CapabilityGate function G is total, deterministic, and LLM-independent — it performs set membership checks, hash comparisons, and boolean arithmetic without invoking any LLM inference. Since G has no natural language processing surface, adversarial prompt injections cannot influence its decisions.

**Automatic privilege decay.** Tokens expire after 300 seconds. We prove (Supplementary Note 3, Theorem 4) that event-scoped authority reduces cumulative privilege exposure to at most 9.1% of the ambient authority model under worst-case assumptions (k=5 tools per batch, 55 system-wide tools, 24h session).

**Fail-safe defaults.** TaintStore is fail-open (unknown sources allowed), but CausalTaintTracker is fail-safe (contamination is irrevocable within a batch). Unmapped event types receive the most restrictive token.

**Auditable decisions.** Every decision includes a machine-readable reason (e.g., "DENY: param 'to' taint=EXTERNAL, policy requires ≤USER"), transforming security from opaque LLM judgment to verifiable code decisions.

### 3.5 Case analysis: the architecture against the OpenClaw hazard topology

We return to the four failure modes documented in Section 1.2 and trace how each is addressed by the architectural properties established above. We argue at the architecture level; implementation-specific parameters (sandbox resource limits, specific escaping functions) are detailed in Methods.

**Malicious supply-chain skills (12% of ClawHub compromised).** Under ambient authority, a malicious skill executes with the agent's full tool set and environment access. Under event-scoped capability authority, two architectural properties contain this threat: (1) **Token scoping** (Layer 1) — a skill invocation event generates a token granting only tools relevant to the skill's declared function (e.g., `web_fetch` and `read_file` for a weather skill), not system-level tools; (2) **Taint tracking** (Layer 2) — any data the malicious skill introduces is registered as EXTERNAL, preventing its use as arguments to sensitive tools. The architectural guarantee: even if the skill's content successfully injects the LLM, the set of exploitable tools is bounded by the token, and data flow from the skill to sensitive parameters is tracked and restricted.

**CVE-2026-25253: cross-site WebSocket hijacking RCE.** This vulnerability allowed unauthenticated command execution via WebSocket. Under our architecture, **event provenance** (Requirement A) provides the defense: every event carries immutable `source` and trust classification set by the framework. An unauthenticated WebSocket message is classified at the lowest trust level (REMOTE_OPEN) and receives a token stripped of all tools capable of code execution or file modification. The RCE attack chain is structurally impossible because the authorization to execute code is never granted for events from untrusted sources.

**Persistent hallucination loops and intent drift.** Under ambient authority, accumulated context errors compound with no privilege decay. Under event-scoped capability authority, **automatic privilege decay** prevents this: each event batch receives a fresh token with a 300-second TTL. Previous context errors do not accumulate privileges — each batch starts from zero authority. Compounding is bounded by the token window, not by session length.

**Over-authorization (500 unsolicited iMessages, insurance dispute).** Under ambient authority, an agent retains messaging access for the entire session regardless of current context. Under event-scoped capability authority, **context-scoped authority** (Principle 2) ensures that tools are granted per event type. A timer or heartbeat event does not warrant messaging tools; they are simply absent from the token. Rate limiting at the EventInjectionGate provides an additional bound on event throughput per source.
