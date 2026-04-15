## 4. Evaluation Framework

### 4.1 Structural barrier classification

We distinguish between two categories of security barriers, which must not be conflated:

**Deterministic barriers** — enforced by code with mathematical certainty:
- *Tool visibility*: Ungranted tools are absent from the LLM's tool schema. The LLM cannot invoke a tool it does not know exists. Reduction: 2-5 of 55 system-wide tools per batch (55 tools distributed across 16 module categories as enumerated in the tool registry; ~11-27× reduction).
- *Temporal window*: Token TTL of 300s is enforced by timestamp comparison. For a 24h agent, this represents ~288× temporal reduction per token window.
- *Path scoping*: File operations outside `granted_paths` are rejected by glob matching.

**Structural barriers** — enforced by policy categorization, correct as long as policies are accurately defined:
- *Taint tracking*: Content-level tracking is evadable through LLM paraphrasing; causal-level tracking is irrevocable within a batch but resets between batches.
- *Rule of Two*: Prevents complete attack chains within a batch, but depends on accurate categorization of tools into the three condition categories.

This classification is critical: deterministic barriers provide guarantees independent of attacker sophistication; structural barriers provide guarantees contingent on policy correctness.

### 4.2 Proposed empirical evaluation

The following evaluation protocol is designed to validate the architecture's claims. Results will be reported in a companion paper.

**Experiment 1: Prompt injection benchmark.** We will evaluate both ambient authority and event-scoped capability implementations against the InjecAgent benchmark¹⁵, the CaMeL benchmark¹², and the BIPIA benchmark¹⁶, measuring: (a) attack success rate under both models, (b) number of attacks that bypass the first barrier (injection succeeds) but are caught by subsequent barriers, and (c) the specific barrier that blocks each attack.

**Experiment 2: CaMeL head-to-head comparison.** Using the CaMeL benchmark and codebase (open-source¹²), we will compare: (a) security efficacy (attack success rate), (b) computational overhead (LLM token usage, latency), and (c) false positive rate on benign workloads. Our architecture predicts zero additional LLM calls for security enforcement versus CaMeL's reported 2.7-2.8× token overhead.

**Experiment 3: False positive/negative rates.** Using a realistic workload corpus (email processing, web research, file management, scheduled tasks), we will measure: (a) CausalTaintTracker false positive rate (legitimate external actions blocked), (b) TaintStore false negative rate (paraphrased external data not detected), and (c) combined dual-layer performance.

**Experiment 4: Cross-batch contamination frequency.** We will analyze real-world agent session logs to determine: (a) what fraction of sessions span multiple event batches with mixed trust levels, (b) how often session history from a prior batch materially influences LLM behavior in a subsequent batch, and (c) the practical attack surface of the cross-batch contamination gap identified in Section 3.3.

**Experiment 5: Rule of Two coverage.** We will systematically analyze the AI Incident Database¹⁷, OWASP LLM Top 10⁶, OpenClaw security advisories⁹, and the EchoLeak disclosure¹⁰ to classify each documented agent security incident against the three Rule-of-Two conditions, measuring what fraction of real-world attacks require all three conditions simultaneously.

### 4.3 Performance characteristics (estimated)

The following are engineering estimates based on the architecture's computational primitives. Measured benchmarks will be reported in the companion evaluation paper.

| Component | Estimated overhead | Basis |
|-----------|-------------------|-------|
| Token issuance | ~1-5ms per batch | Dictionary lookup + frozen dataclass construction |
| Gate verification | ~0.3ms per tool call | 3 checks: set membership, SHA-256 hash lookup, boolean comparison |
| TaintStore memory | ~2-5MB resident | 10,000 records × ~200-500 bytes per record |
| Side consumers | ~50ms per batch | Dominated by security scanner I/O |
| Total vs. LLM inference | ~0.1-1% of response time | LLM inference typically 1-5s |

CaMeL reports 2.7-2.8× token overhead from its dual-LLM architecture¹². Our architecture requires zero additional LLM calls, as the CapabilityGate is pure deterministic code.

---

## 5. Societal Implications

### 5.1 Trust infrastructure

Human society operates on networks of proxy trust — lawyers, financial advisors, physicians — secured by licensing, fiduciary duty, and legal liability developed over centuries. AI agents represent a new class of proxy whose faithfulness can be silently subverted through data channels without breaching authentication. When 145,000 users deployed OpenClaw agents with access to their digital lives, they established proxy trust relationships with entities compromisable by a single crafted email.

Our architecture provides a structural response: the Rule of Two ensures that even if the agent's faithfulness is fully compromised, the structural constraint prevents completion of exfiltration chains within an event batch. The proxy can fail; the system limits the severity of exploitation.

### 5.2 Enabling regulation

Current governance frameworks — the EU AI Act, US Executive Order on AI Safety, China's Generative AI regulations — assume AI systems are tools operated by responsible humans. Autonomous agents disrupt this. Our architecture addresses this through verifiable properties mapping to specific regulatory requirements:

- **Auditable decision trails** (EU AI Act Article 12: Record-Keeping; Article 14: Human Oversight): machine-readable logs of every Gate decision with deterministic justification.
- **Deterministic security bounds** (Article 15: Robustness and Cybersecurity): the Rule-of-Two invariant is verifiable through code inspection, not statistical testing.
- **Per-event accountability** (US Executive Order Section 4.1): each agent action traced to a specific triggering event with its authorization chain.

For the first time, a regulator can verify: "this agent cannot exfiltrate user data even if its language model is compromised" — a claim enforced by deterministic code, not LLM compliance.

### 5.3 Adoption considerations

Transitioning from ambient authority to event-driven capability security entails practical barriers that must be acknowledged:

**Developer experience.** The event-driven model requires restructuring agent applications from imperative request-response flows to event-driven architectures — a significant paradigm shift for developers accustomed to frameworks like LangChain or OpenClaw.

**Migration path.** Incremental adoption is possible: existing agents can wrap their tool-calling layer with a CapabilityGate without restructuring the entire application. Full EventBus integration provides the strongest guarantees but can be adopted progressively.

**Usability impact.** As discussed in Section 7.2, the CausalTaintTracker's coarse granularity may block legitimate operations. User-facing permission prompts and configurable trust policies are necessary to balance security with productivity.
