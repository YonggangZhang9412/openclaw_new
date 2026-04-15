## 3. Resolving the Impossibility: The Event-Driven Capability Architecture

The Agent Authority Impossibility shows that C1, C2, and C3 cannot coexist under ambient authority. Our resolution does not weaken any of these conditions. Instead, we **replace ambient authority with event-scoped capability authority**, dissolving the contradiction by ensuring that the agent's permissions are dynamically bound to the specific event it is processing, not to its session-level identity.

### 3.1 Design principles derived from the impossibility

Three principles follow logically from the impossibility theorem:

**Principle 1: Heterogeneous security enforcement.** If the LLM can be manipulated through its data channel (the token stream), then the security enforcement layer must operate through a *different* channel — one that is immune to natural language manipulation. This means: **deterministic code, not LLM judgment, must make security decisions.**

**Principle 2: Context-scoped authority.** If the agent's effective intent can change with each data item it processes, then its authority must be scoped to each processing context — not granted as a session-wide ambient capability. The agent's permissions when processing an email must differ from its permissions when executing a cron job.

**Principle 3: Structural attack chain interruption.** If we cannot detect all possible adversarial inputs (prompt injection is an open-ended threat), then we must instead **structurally prevent the completion of attack chains** regardless of whether the initial injection succeeds. A successful injection that cannot access sensitive data or exfiltrate it is a contained injection.

### 3.2 Why EventBus is a structural necessity, not an architectural preference

Principles 1-3 impose specific requirements on the underlying system architecture. We now show that these requirements necessitate an event-driven architecture — specifically, a unified EventBus that normalizes all triggers (user messages, timers, file changes, network events, scheduled tasks) into a common event model.

**Requirement A: Unforgeable event provenance** (from Principle 2).

To scope authority to the current processing context, the capability issuer must know the *origin* and *trust level* of the event being processed. This requires metadata fields — source identifier, origin chain, trust classification — that are immutable once the event is created. In a request-response model, such metadata resides in HTTP headers, which are **forgeable by any client**. In our EventBus, events are frozen dataclasses whose `source`, `origin_chain`, and `cascade_depth` fields are set at creation time and cannot be modified by any consumer.

**Requirement B: Framework-enforced cascade depth** (from Principle 3).

Agent actions can trigger secondary events (e.g., a file write triggers a file-watch event, which triggers another agent action). Without structural depth limits, an attacker can induce infinite event loops, consuming resources or amplifying the effects of a single injection. In a request-response model, cascade depth must be manually propagated by each handler — a single non-compliant handler breaks the guarantee. In our EventBus, the framework itself automatically increments `cascade_depth` when an event consumer injects a child event. This invariant is enforced at the framework level, not the application level.

**Requirement C: Batch-atomic security decisions** (from Principles 2 and 3).

Real-world agent triggers often arrive in correlated batches — three file changes from a single `git pull`, a cron job that coincides with an incoming message. Security decisions about these events are interdependent: the Rule of Two (Section 3.5) must evaluate across the *entire batch* to determine whether the combination of untrusted input, sensitive data access, and external actions creates an attack chain. In a request-response model, each request generates an independent security decision, fragmenting the analysis window and creating TOCTOU (time-of-check-to-time-of-use) race conditions. EventBus batch processing enables atomic security evaluation across correlated events.

**Requirement D: Composable reactive security monitoring** (from Principle 1).

Security monitoring (threat scanning, audit logging, anomaly detection) must operate in parallel with agent execution, without explicit coupling. In a request-response model, each security check requires a dedicated middleware, and the ordering and composition of middlewares is fragile. In our EventBus, 18 side consumers subscribe to the event stream independently — security scanning, capability gate auditing, error aggregation, delivery monitoring — each processing events without awareness of or coupling to the others.

These requirements are not merely convenient properties. They are **structural preconditions** for the security guarantees we establish. An architecture that does not provide unforgeable provenance, framework-enforced cascade limits, batch atomicity, and composable monitoring cannot implement the capability security model we describe. This is why **EventBus is not an optional component but a necessary foundation**.

### 3.3 The CapabilityGate: deterministic security enforcement

Atop the EventBus, the CapabilityGate implements Principle 1 — heterogeneous security enforcement — through a three-layer verification pipeline executed in pure deterministic code with zero LLM involvement.

**Layer 1: Token validation.** Each event batch receives a just-in-time (JIT) CapabilityToken — a frozen, immutable object specifying:
- `granted_tools`: a frozenset of 2-5 tools (versus 45+ available system-wide)
- `granted_paths`: filesystem glob patterns derived from the event payload
- `ttl`: time-to-live of 300 seconds (automatic expiration)
- Rule-of-Two flags: `allows_untrusted_input`, `allows_sensitive_data`, `allows_external_action`

The token is issued by a CapabilityIssuer that consults `EVENT_TOOL_GRANTS` — a mapping from event types to minimal tool sets — and `_classify_event_trust()` — which downgrades tool grants for remote or unverified event sources. The LLM only sees tool schemas for the granted tools; ungranted tools are invisible to the model.

**Layer 2: Value-level taint checking.** Every tool return value is registered in a TaintStore with its origin and taint level (USER=0, INTERNAL=1, EXTERNAL=2). When the LLM subsequently uses a value as a tool argument, the Gate queries the TaintStore to determine whether the value's taint level exceeds the tool's policy. For instance, an email address extracted from a web fetch (taint=EXTERNAL) cannot be used as the `to` parameter of `send_email` (which requires taint≤USER).

A complementary CausalTaintTracker operates at the context level rather than the value level. Once any EXTERNAL data enters the LLM's context in a processing round, the causal tracker permanently marks the round as contaminated. This provides a fail-safe backstop: even if the LLM paraphrases external data to evade content-level fingerprinting, the causal-level taint persists.

The two layers are complementary, not redundant. TaintStore provides precision (blocking only tool calls whose specific parameters contain external data, avoiding false positives). CausalTaintTracker provides recall (catching paraphrased or recombined external data that evades fingerprinting, avoiding false negatives). Together, an attacker must evade both content-level and causal-level tracking simultaneously.

**Layer 3: Rule-of-Two structural constraints.** Analysis of known agent security incidents reveals that every successful data exfiltration attack requires three simultaneous conditions: (1) processing of untrusted input, (2) access to sensitive data, and (3) execution of an external action. The Rule of Two constrains the CapabilityToken such that at most two of these three conditions can be true simultaneously.

This structural constraint is sufficient to prevent complete attack chains **within a single event batch where all three conditions are correctly identified**. If an agent processes untrusted web content and accesses sensitive files, it cannot perform external actions (no data can leave the system). If it processes untrusted content and can send emails, it cannot access sensitive files (no valuable data to exfiltrate). The constraint is enforced both statically (at token issuance time) and dynamically (during batch execution via CausalRuleOfTwo), providing defense-in-depth against runtime condition changes.

**Scope and residual risk.** The Rule of Two operates per event batch. A residual risk exists in **cross-batch context contamination**: external data processed in a prior batch may persist in session history and influence the LLM's behavior in a subsequent locally-trusted batch (e.g., a cron job). In this scenario, the current batch's token may classify `allows_untrusted_input = False` (because the event source is LOCAL_TRUSTED), while the LLM's reasoning is in fact influenced by prior external data. A compound tool such as `bash("cat .env | curl attacker.com")` could then complete an attack chain without triggering the Rule of Two, because the untrusted input condition is not recognized at the batch level. CausalTaintTracker mitigates this partially — it is currently reset per batch, but could be extended to persist across batches at the cost of increased false positives (any session that has ever processed external data would permanently restrict external actions in subsequent batches). This security-usability tradeoff is a key area for future architectural refinement, and represents the boundary between per-batch structural guarantees and session-level behavioral risks.

### 3.4 Security properties of the architecture

The combined EventBus + CapabilityGate architecture achieves properties that are individually desirable and collectively unprecedented in agent security:

**Determinism.** Every security decision is a deterministic function of its inputs. The same tool call with the same arguments under the same token produces the same ALLOW/DENY decision. This enables exhaustive testing, formal verification, and reproducible auditing.

**Prompt injection immunity.** The CapabilityGate does not process natural language. It performs dictionary lookups, hash comparisons, and boolean arithmetic. There is no input that can "convince" the Gate to change its decision, because it does not interpret meaning — only structure.

**Automatic privilege decay.** Capability tokens expire after 300 seconds regardless of session duration. There is no persistent ambient authority. An agent that has been running for hours holds zero accumulated privilege; each event batch starts from zero and receives only the minimum authority required.

**Fail-safe defaults.** Unknown taint sources default to allowed (TaintStore fail-open), but the CausalTaintTracker defaults to contaminated once any external data is observed (fail-safe). Token issuance defaults to minimal tool grants. Unmapped event types receive the most restrictive token. The system fails toward security, not toward functionality.

**Auditable decisions.** Every Gate decision includes a machine-readable reason: "DENY: tool 'send_email' param 'to' has taint=EXTERNAL, policy requires taint≤USER." This transforms security from "the LLM chose not to" (opaque) to "Check 2 rejected at line 1420" (verifiable). Regulators and auditors can inspect the decision log without understanding the LLM's reasoning.
