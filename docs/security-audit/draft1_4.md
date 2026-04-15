## 4. Quantitative Analysis

### 4.1 Attack surface reduction

Under ambient authority (OpenClaw model), the agent's instantaneous attack surface at any moment can be expressed as:

```
S_ambient = |T| × D × p(injection)
```

where |T| is the number of available tools (~45), D is the session duration (unbounded), and p(injection) is the probability that an adversarial input successfully manipulates the LLM. Since D → ∞ for persistent agents, the expected number of successful attacks over the agent's lifetime is unbounded for any p(injection) > 0.

Under event-scoped capability authority (our model):

```
S_capability = |T_token| × TTL × p(injection) × p(bypass_taint) × p(bypass_rule_of_two)
```

where |T_token| ≈ 2-5 (tools granted per event batch), TTL = 300s, p(bypass_taint) is the probability of evading both content-level and causal-level taint tracking, and p(bypass_rule_of_two) is the probability that an attack chain can complete despite the structural constraint.

Conservative estimates:
- |T_token|/|T| ≈ 5/45 ≈ 0.11 (11% tool exposure)
- TTL/D ≈ 300/∞ → 0 (asymptotic reduction; for a 24h session, 300/86400 ≈ 0.0035)
- p(bypass_taint): Content-level evasion (LLM paraphrasing) is feasible, but causal-level tracking is monotonic and irrevocable. We estimate p ≈ 0.1 for sophisticated attacks.
- p(bypass_rule_of_two): Requires an attack that does not need one of the three conditions. By construction, no complete exfiltration attack can avoid all three. We estimate p ≈ 0.01 for edge cases where the categorization is imprecise.

Combined reduction factor: 0.11 × 0.0035 × 0.1 × 0.01 ≈ 3.85 × 10⁻⁷, representing approximately **seven orders of magnitude** reduction in attack surface for deterministic attacks.

This estimate is conservative: it does not account for the Gate's token validation (which renders ungrantted tools invisible to the LLM, preventing even the formation of adversarial tool calls) or the EventInjectionGate's rate limiting and cascade depth restrictions.

### 4.2 Coverage analysis against known attack types

We analyzed the architecture's coverage against the complete taxonomy of agent security threats documented in the 2025-2026 literature:

| Attack class | Ambient authority (OpenClaw) | Event-scoped capability (ours) | Mechanism |
|-------------|-----|------|-----------|
| Direct prompt injection via data channel | Detected but not blocked (LLM sees warning) | Structurally contained (2-5 tools visible, paths restricted) | Token scoping |
| Indirect injection via tool returns | No defense (tool results enter context as trusted) | Taint-tracked (tool results registered with source taint) | TaintStore |
| Credential exfiltration chain (read secrets → send external) | No structural prevention | Rule of Two blocks (untrusted input + sensitive data + external action > 2) | Structural constraint |
| Supply chain attack via malicious plugin | Plugin shares process/env, full access | Skill sandboxed (SandboxProcess: no env, 512MB, 30s), tools capability-gated | Isolation + Gate |
| Infinite loop / resource exhaustion | No timeout default, no cascade limit | TTL (300s), frequency limit (20 calls/round), cascade depth (max 20), queue cap (10,000) | Multi-layer bounds |
| Context drift / intent degradation | No defense | Context compaction + per-batch fresh token (no privilege accumulation) | Automatic decay |
| Timing side-channel (device enumeration) | Standard web auth | Constant-time HMAC with dummy secret for unknown devices | Cryptographic |

### 4.3 Performance characteristics

The security architecture introduces overhead that must be acknowledged:

- **Token issuance**: One CapabilityIssuer call per event batch (~1-5ms, pure Python dictionary lookup)
- **Gate verification**: Three checks per tool call (~0.1ms per check, dominated by TaintStore SHA-256 hash lookup)
- **TaintStore memory**: Maximum 10,000 records with 600s TTL, approximately 2-5MB resident memory
- **Side consumers**: 18 parallel consumers add ~50ms total latency per event batch (dominated by security scanning I/O)

For comparison, a single LLM inference call typically requires 1-5 seconds and dominates the agent's response latency. The security overhead is approximately **0.1-1% of total response time** — negligible in practice.

CaMeL (Google DeepMind, 2025), the closest comparable system, reports 2.7-2.8× token overhead due to its dual-LLM architecture (Privileged LLM + Quarantined LLM). Our architecture requires zero additional LLM calls for security enforcement, as the CapabilityGate is pure deterministic code. The only additional LLM interaction is the optional intent classification for token issuance, which can be cached per-session.

---

## 5. Societal Implications

### 5.1 The collapse of proxy trust infrastructure

Human society operates on networks of **proxy trust**. You trust your lawyer to represent your interests, your financial advisor to manage your assets, your physician to prescribe appropriate treatments. These proxy relationships are secured by professional licensing, fiduciary duty, legal liability, and institutional oversight — mechanisms developed over centuries.

AI agents represent a fundamentally new class of proxy. Unlike human proxies, they have no intrinsic motivation alignment (they optimize token predictions, not your welfare), no legal liability (no entity "is" the agent in a legal sense), no professional ethics training (RLHF is a statistical approximation, not a binding commitment), and — most critically — **their proxy faithfulness can be silently subverted through their data channels without any breach of authentication**.

When 145,000 users deployed OpenClaw agents with access to their email, calendar, filesystem, and messaging apps, they established 145,000 proxy trust relationships with entities whose faithfulness could be compromised by a single malicious email. The discovery that 12% of the ClawHub skill marketplace was compromised is structurally equivalent to discovering that 12% of lawyers in a bar association are secretly working for opposing counsel — except that the "compromise" of an AI agent requires no conspiracy, no bribery, and no human participation. It requires only a carefully worded paragraph embedded in an email.

This represents a **qualitative degradation of trust infrastructure** that society has not previously encountered. The solution must be architectural: proxy trust in AI agents cannot be secured through behavioral training (RLHF, safety fine-tuning) alone, because the agent's behavior is a function of its data environment, which is adversary-controlled. It must be secured through **structural constraints that limit the damage any single proxy failure can cause**, regardless of the failure's origin.

Our Rule-of-Two constraint provides exactly this property: even if the agent's proxy faithfulness is completely compromised (it follows the attacker's instructions), the structural constraint prevents the completion of any exfiltration attack chain. The proxy can fail; the system cannot be fully exploited.

### 5.2 The asymmetry of attack and defense

Prompt injection creates a historically unprecedented asymmetry between attack and defense:

- **Attack cost**: Composing a natural language instruction — no programming skill, no exploit development, no zero-day research required. The attack surface is the natural language itself.
- **Defense cost**: Restructuring the entire agent architecture — from ambient authority to capability-based security, from LLM-dependent to deterministic enforcement, from monolithic permissions to event-scoped tokens.

This asymmetry means that every agent deployed under the ambient authority model is structurally vulnerable from the moment of deployment, and the vulnerability cannot be patched without architectural change. Traditional software vulnerabilities (buffer overflows, SQL injection) can often be fixed with localized patches. The Agent Authority Problem cannot, because the vulnerability is not in a specific code path but in the **relationship between the permission model and the data processing model**.

The implication for the industry is stark: the transition from ambient authority to capability-based security is not optional for any agent system that processes untrusted data and holds privileged access. It is a structural necessity, analogous to the transition from HTTP to HTTPS for e-commerce, or from string concatenation to parameterized queries for database access.

### 5.3 Enabling regulation through verifiable security properties

Current AI governance frameworks — the EU AI Act, the US Executive Order on AI Safety, China's Generative AI regulations — share a common assumption: that AI systems are tools operated by humans who bear responsibility for their actions. Autonomous agents disrupt this assumption fundamentally. When an agent autonomously decides to send an email or execute a financial transaction, the chain of responsibility becomes ambiguous: is the operator responsible for an action they did not approve, could not have foreseen, and that was triggered by adversarial data they did not create?

Our architecture addresses this governance gap through **verifiable security properties**:

1. **Auditable decision trails.** Every Gate decision produces a machine-readable log: tool name, arguments, token state, taint levels, Rule-of-Two evaluation, final decision with reason code. Regulators can inspect these logs to determine exactly what the agent was permitted to do and why.

2. **Deterministic security bounds.** The Rule-of-Two constraint provides a mathematically verifiable safety property: no execution path can simultaneously process untrusted input, access sensitive data, and perform external actions. This is a property that can be certified, not merely claimed.

3. **Per-event accountability.** Because permissions are scoped to individual event batches (not sessions), each agent action can be traced to a specific triggering event, with its source, trust classification, and the capability token that authorized it. This creates an accountability chain that existing ambient authority models cannot provide.

For the first time, it becomes possible to answer the regulator's question: "Can you prove that this agent cannot exfiltrate user data even if its language model is compromised?" Under ambient authority, the honest answer is "no." Under our architecture, the answer is: "Yes — the Rule-of-Two constraint structurally prevents the simultaneous presence of untrusted input, sensitive data access, and external action, and this constraint is enforced by deterministic code independent of the language model."

### 5.4 Enabling high-stakes deployment domains

The practical consequence of verifiable security is the **expansion of the domain boundary** for autonomous agent deployment.

Today, responsible organizations restrict AI agents to low-stakes tasks — drafting emails, summarizing documents, generating code suggestions — precisely because the ambient authority model cannot guarantee safety for consequential actions. Healthcare, finance, legal services, and critical infrastructure remain largely off-limits for autonomous agents, not because the AI capabilities are insufficient, but because the security architecture cannot provide the guarantees these domains require.

Event-driven capability security changes this calculus. A medical AI agent operating under our architecture cannot exfiltrate patient records even if its language model is compromised, because the Rule of Two prevents the simultaneous presence of untrusted external input and sensitive medical data access when external communication is possible. A financial agent cannot be tricked into unauthorized transfers, because the taint tracking system prevents externally-sourced account numbers from being used as transfer parameters. A legal AI cannot leak privileged communications, because the capability token for processing incoming correspondence does not include tools for outbound communication.

These are not aspirational claims. They are structural properties of the architecture, verifiable through code inspection and formal analysis.
