## 4. Quantitative Security Analysis

### 4.1 Threat model

We consider an adversary who can embed adversarial instructions in data sources processed by the agent (emails, web pages, API responses). The adversary's goal is data exfiltration: causing sensitive data to cross the system boundary. The adversary succeeds in prompt injection with probability p per batch (empirically p ∈ [0.2, 0.8] for current LLMs¹⁵˒¹⁶). We model token selection as worst-case uniform random (Assumption 1 in Supplementary Notes); the actual deterministic selection is strictly more restrictive.

### 4.2 Main results

**Theorem 1 (Multi-Barrier Exfiltration Bound).** In an agent system with n tools where each event batch is authorized a token of at most k tools, the probability that an adversary successfully exfiltrates sensitive data through a single batch is bounded by:

$$\Pr[\text{Exfil}(b)] \leq p \cdot \min\!\left(\frac{kn_R}{n},\, \frac{kn_X}{n}\right) \cdot (1 - R_2) \cdot q$$

where each factor corresponds to a distinct architectural barrier that the attacker must overcome: p (injection success), $kn_R/n$ and $kn_X/n$ (tool visibility — union bounds on the probability of sensitive-read and external-send tools being granted in the token; these arise from the same token draw and are not probabilistically independent, but represent architecturally distinct constraints), $R_2$ (Rule of Two: zeroes the bound when enforced), q (cross-batch taint evasion). Full proof in Supplementary Notes, Section 2.

**Theorem 2 (Session-Level Bound).** Over $B$ batches: $\Pr[\exists\, b : \text{Exfil}(b)] \leq 1 - (1-\varepsilon)^B$ where $\varepsilon$ is the per-batch bound. Under ambient authority: $\Pr[\exists\, b : \text{Exfil}(b)] \geq 1-(1-p)^B \to 1$ as $B \to \infty$.

**Theorem 3 (Rule of Two: Zero-Probability Guarantee).** When $R_2 = 1$: $\Pr[\text{Exfil}(b)] = 0$, regardless of $p$ or $q$.

**Theorem 4 (Causal Taint Irrevocability).** The causal taint sequence is monotonically non-decreasing ($\tau_i \leq \tau_j$ for i $\leq$ j). Once EXTERNAL data is observed, within-batch taint evasion is exactly zero ($q_{\text{within}} = 0$).

**Theorem 5 (Bound Integrity).** The CapabilityGate is deterministic and LLM-independent. The adversary can influence p and q but cannot manipulate $\alpha_R$, $\alpha_X$, or $R_2$.

**Remark (Conservative bound).** Theorem 1 quantifies only the CapabilityGate's contribution. The following additional security layers are present in the system but not included in the bound:
- EventInjectionGate: per-source rate limiting (10-200 events/s), payload size cap (16-131KB), cascade depth hard limit (20), event type whitelist
- Path scoping: tool operations restricted to `granted_paths` globs derived from event payload
- Trust-level tool restriction: REMOTE_OPEN events have $\alpha_X = 0$ (Section 3.3), yielding $\Pr[\text{Exfil}] = 0$ independent of the bound's other factors
- Context compaction: semantic compression of long session histories, reducing cross-batch contamination surface

The actual system is strictly safer than the bound suggests.

### 4.3 Concrete instantiation

Parameters: n = 55, k = 5, n_R = 8, n_X = 6, p = 0.5, q = 0.1, B = 288.

| Scenario | $\Pr[\text{Exfil}(b)]$ | Session $\Pr[\exists b: \text{Exfil}(b)]$ |
|----------|-------------|--------------------------|
| Ambient authority | $\geq$ 0.5 | $\approx 1.0$ |
| Event-scoped, $R_2$ = 0 | $\leq$ 0.027 | $\leq$ 0.9997 (independence) or $\leq$ min(1, 288·0.027) (union bound) |
| Event-scoped, $R_2$ = 1 | = 0 | = 0 |
| REMOTE_OPEN events | = 0 | = 0 |

Per-batch reduction ($R_2$=0): 0.5 / 0.027 ≈ 18×. With $R_2$=1 or REMOTE_OPEN: infinite (zero vs non-zero).

### 4.4 Proposed empirical evaluation

Results will be reported in a companion paper.

**Experiment 1 (Injection benchmarks).** Evaluate against InjecAgent¹⁵, CaMeL benchmark¹², and BIPIA¹⁶: attack success rate under ambient vs event-scoped authority, with barrier-by-barrier attribution.

**Experiment 2 (CaMeL comparison).** Head-to-head: security efficacy, computational overhead (our zero additional LLM calls vs CaMeL's 2.7-2.8× token overhead), false positive rate.

**Experiment 3 (Taint tracking precision/recall).** CausalTaintTracker false positive rate on realistic workloads; TaintStore false negative rate on paraphrased external data.

**Experiment 4 (Cross-batch contamination frequency).** Fraction of real-world sessions where session history from a prior batch materially influences LLM behavior in subsequent batches.

**Experiment 5 (Rule of Two coverage).** Systematic classification of AI Incident Database¹⁷ and OWASP LLM Top 10⁶ incidents against the three Rule-of-Two conditions.

---

## 5. Societal Implications

### 5.1 The collapse of proxy trust

Human society operates on networks of proxy trust — lawyers, financial advisors, physicians — secured by licensing, fiduciary duty, and legal liability developed over centuries. AI agents represent a fundamentally new class of proxy. Unlike human proxies, their faithfulness can be silently subverted through data channels without breaching authentication: a single crafted email can redirect an agent's subsequent actions while its identity credentials remain intact. When 145,000 users deployed OpenClaw agents with access to their email, calendar, and filesystem, they established proxy trust relationships with entities compromisable by a carefully worded paragraph.

Our architecture addresses this through structural containment. The Rule of Two ensures that even if an agent's proxy faithfulness is fully compromised, exfiltration chains cannot complete. The trust-level system ensures that for the most untrusted sources, exfiltration is structurally impossible (Section 3.3). The proxy can fail; the system bounds the severity of exploitation.

### 5.2 The asymmetry of attack and defense

Prompt injection creates a historically unprecedented asymmetry. Attack cost: composing a natural language sentence — no programming skill or exploit development required. Defense cost: restructuring the entire agent architecture from ambient authority to event-driven capability security. This asymmetry means that every agent deployed under the ambient authority model is structurally vulnerable from deployment, and the vulnerability cannot be patched without architectural change — unlike buffer overflows or SQL injection, which can often be fixed with localized patches.

### 5.3 Enabling high-stakes deployment

Today, responsible organizations restrict AI agents to low-stakes tasks precisely because the ambient authority model cannot guarantee safety for consequential actions. Our architecture changes this calculus. A medical AI agent operating under trust-level-specific token issuance cannot exfiltrate patient records through untrusted sources ($\alpha_X = 0$ for REMOTE_OPEN). A financial agent cannot be tricked into unauthorized transfers because taint tracking prevents externally-sourced account numbers from being used as transfer parameters. A legal AI cannot leak privileged communications because the capability token for processing incoming correspondence does not include tools for outbound communication. These are structural properties verifiable through code inspection, not claims dependent on LLM compliance.

### 5.4 Enabling regulation

Our architecture provides verifiable properties mapping to specific regulatory requirements:

- **Auditable decision trails** (EU AI Act Article 12, Article 14): machine-readable logs of every Gate decision with deterministic justification.
- **Deterministic security bounds** (Article 15: Robustness and Cybersecurity): the Rule-of-Two invariant and trust-level guarantees are verifiable through code inspection, not statistical testing.
- **Per-event accountability** (US Executive Order Section 4.1): each agent action traced to a specific triggering event with its authorization chain.

### 5.5 Adoption considerations

**Developer experience.** The event-driven model requires restructuring from imperative request-response flows — a significant paradigm shift.

**Migration path.** Incremental adoption is possible: existing agents can wrap their tool-calling layer with a CapabilityGate. Full EventBus integration provides the strongest guarantees.

**Usability.** CausalTaintTracker's coarse granularity may block legitimate operations when external data is observed in the same batch as outbound communication. Configurable trust policies are necessary to balance security with productivity.
