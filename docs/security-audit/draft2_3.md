## 3. The Event-Driven Capability Architecture

The Agent Authority Problem shows that C1, C2, and C3 cannot coexist under ambient authority. Our resolution replaces ambient authority with **event-scoped capability authority**, dissolving the contradiction without weakening any condition:

- **C1 preserved**: The agent continues to autonomously process untrusted data without per-item human approval.
- **C2 preserved**: The agent retains access to sensitive resources, but this access is event-scoped (granted per event batch, for specific paths and tools, automatically revoked after 300 seconds) rather than ambient.
- **C3 satisfied**: The CapabilityGate is pure deterministic code with zero LLM dependency.

The contradiction dissolves because **the precondition — ambient authority — is removed**. The agent can access sensitive resources and process untrusted data, but never with the same token in a configuration enabling a complete attack chain.

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

**Layer 3 (Rule of Two).** Every successful data exfiltration attack requires three conditions: (1) untrusted input processed, (2) sensitive data accessed, (3) external action executed. The Rule of Two constrains the token so at most two conditions hold simultaneously within a single event batch. This is sufficient to prevent complete attack chains within that batch scope (see Section 7.3 for cross-batch scope limitations).

> **[Figure 3]** **(a)** Rule of Two constraint space: three conditions as a Venn diagram; any two may overlap, but the triple intersection (complete attack chain) is structurally excluded. **(b)** Attack chain comparison: under ambient authority, a single successful injection yields full tool access and completes the chain; under event-scoped capability, the chain must pass through five barriers (tool visibility, TTL, taint check, Rule of Two, path scoping), of which the first three are **deterministic** (guaranteed by code) and the latter two are **structural** (dependent on correct policy categorization).

### 3.4 Security properties

The architecture achieves five properties:

**Determinism.** Every Gate decision is a deterministic function of its inputs — enabling exhaustive testing, formal verification, and reproducible auditing.

**Prompt injection immunity.** The Gate performs dictionary lookups, hash comparisons, and boolean arithmetic. It does not interpret natural language and cannot be "convinced" to change its decision.

**Automatic privilege decay.** Tokens expire after 300s. Each event batch starts from zero authority.

**Fail-safe defaults.** TaintStore is fail-open (unknown sources allowed), but CausalTaintTracker is fail-safe (contamination is irrevocable within a batch). Unmapped event types receive the most restrictive token.

**Auditable decisions.** Every decision includes a machine-readable reason (e.g., "DENY: param 'to' taint=EXTERNAL, policy requires ≤USER"), transforming security from opaque LLM judgment to verifiable code decisions.

### 3.5 Case analysis: the architecture against the OpenClaw hazard topology

We now return to the four failure modes documented in Section 1.2 and trace how each is addressed by the architecture — closing the loop between the motivating evidence and the proposed solution.

**Malicious supply-chain skills (12% of ClawHub compromised).** Under ambient authority, a malicious skill executes in the same process with full access to `process.env` and all system tools. Under our architecture, skills execute through two constrained paths: (a) deterministic execution via `shlex.quote()`-escaped templates in a SandboxProcess (512MB memory, 30s timeout, empty environment — no access to credentials), or (b) LLM-mediated execution where all tool calls pass through CapabilityGate. In path (b), even if a malicious skill's instructions successfully inject the LLM, the token scoping (Layer 1) restricts available tools to 2-5 relevant to the event type, and the taint tracking (Layer 2) prevents external data from flowing to sensitive parameters. A keylogger cannot be installed because the token for a skill invocation event does not grant `bash` or `write_file` to system directories.

**CVE-2026-25253: cross-site WebSocket hijacking RCE.** This vulnerability allowed an attacker to send commands through the Control UI's WebSocket without authentication. Under our architecture, every event entering the EventBus carries immutable `source` and `origin_chain` metadata set by the framework. An unauthenticated WebSocket message would be classified as REMOTE_OPEN by `_classify_event_trust()` — the lowest trust level — and receive a token stripped of all high-risk tools (`bash`, `run_command`, `daemon_restart`, `write_file`). Even if the attacker's message reaches the LLM, the CapabilityGate enforces that the LLM can only invoke read-only tools. The RCE attack chain (execute arbitrary code on the host) is structurally impossible because the code execution tool is not in the token.

**Persistent hallucination loops and intent drift.** Under ambient authority, accumulated context errors compound over time with no privilege decay. Under our architecture, each event batch receives a fresh CapabilityToken with a 300-second TTL. Previous context errors do not accumulate privileges — each batch starts from zero authority. Additionally, context compaction (Section 3 of the implementation) automatically compresses long session histories, reducing the influence of early erroneous context on later behavior.

**Over-authorization (500 unsolicited iMessages, insurance dispute).** Under ambient authority, an agent granted messaging access retains it for the entire session regardless of what it is currently doing. Under our architecture, the `send_message` tool is only granted when the event type and trust classification warrant it. A timer-triggered heartbeat event, for instance, would receive a token without messaging tools — preventing the agent from spontaneously sending messages during background processing. The EventInjectionGate's rate limiting (10 requests/second for MOBILE devices) provides an additional quantitative bound on message volume.
