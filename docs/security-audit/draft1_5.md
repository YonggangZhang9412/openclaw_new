## 6. Related Work

### 6.1 Prompt injection defenses

Prompt injection has been studied extensively since its identification as the #1 LLM vulnerability (OWASP, 2024). Existing defenses fall into three categories:

**Detection-based approaches** use pattern matching, classifier models, or perplexity analysis to identify adversarial inputs before they reach the LLM. These approaches face a fundamental limitation: the space of possible adversarial encodings is open-ended, and detection is necessarily incomplete. Our architecture does not attempt to detect all injections; instead, it structurally contains the damage any successful injection can cause.

**Instruction hierarchy approaches** attempt to establish precedence among different instruction sources (system prompt > user prompt > external data). While principled, these approaches ultimately depend on the LLM's ability to maintain the hierarchy under adversarial pressure — a capability that has been empirically shown to degrade with sophisticated attacks and long contexts.

**Architectural separation approaches**, most notably CaMeL (Google DeepMind, 2025), represent the closest related work. CaMeL introduces the Privileged/Quarantined LLM separation and data provenance tracking through a custom Python interpreter. Our work extends this direction in three critical ways: (1) we identify EventBus as a structural prerequisite for capability security, which CaMeL does not address; (2) our taint tracking operates at two complementary layers (content-level TaintStore + causal-level CausalTaintTracker), compared to CaMeL's single-layer provenance tracking; and (3) our Rule-of-Two provides a formally characterizable structural constraint that bounds the attack surface independently of injection detection efficacy. Furthermore, CaMeL reports 2.7-2.8× token overhead from its dual-LLM architecture; our deterministic Gate requires zero additional LLM calls.

### 6.2 Capability-based security

Capability-based security, originating with Dennis and Van Horn (1966), replaces ambient authority with explicit capability tokens that must be presented to access resources. The principle has been implemented in operating systems (seL4, Capsicum), web browsers (Content Security Policy), and distributed systems (Macaroons, Biscuit).

Our contribution is the adaptation of capability-based security to the unique challenges of LLM agents — specifically, the need to scope capabilities not merely to the *identity* of the principal (which is always "the agent") but to the *operational context* (which event is being processed, what trust level it carries, what data has entered the context). This requires the EventBus infrastructure described in Section 3.2, which has no precedent in classical capability systems.

### 6.3 Information flow control and taint tracking

Taint tracking has a rich history in programming language security (Perl's taint mode, Ruby's taint levels, DIFT hardware-assisted taint tracking). Our TaintStore extends these concepts to the LLM agent domain, where the "program" is the LLM's token generation process — opaque, non-deterministic, and not amenable to traditional static or dynamic analysis.

The key innovation is the dual-layer design: content-level fingerprinting (which can be evaded by paraphrasing) complemented by causal-level tracking (which is irrevocable once external data enters the context). This addresses the unique challenge of LLM-mediated information flow, where the processing entity can arbitrarily transform data while preserving its semantic content.

### 6.4 Alignment and behavioral safety

A significant body of work addresses agent safety through **behavioral approaches**: reinforcement learning from human feedback (RLHF), constitutional AI, safety fine-tuning, and instruction hierarchy training. These methods aim to make the LLM itself more reliable — more likely to follow safety instructions, less susceptible to adversarial manipulation, better calibrated in its refusal of dangerous requests.

Our work is **complementary, not competitive** with this research program. The distinction can be stated precisely: behavioral safety (alignment) addresses *what the agent wants to do* — shaping its intent to be benign. Structural security (our contribution) addresses *what the agent is allowed to do* — bounding its actions regardless of intent. Neither alone is sufficient:

- An aligned agent under ambient authority remains vulnerable to sophisticated prompt injection that exceeds its alignment training distribution. Alignment reduces p(injection succeeds) but cannot drive it to zero. Our architecture limits the *consequence* of successful injection regardless of p.
- A structurally secured agent with a misaligned LLM may produce unhelpful or adversarial outputs *within* its permitted tool set. The CapabilityGate constrains *which tools* and *which data flows*, but cannot evaluate the *semantic appropriateness* of content generated within those constraints.

The mature security posture for autonomous agents will require both: alignment to minimize the frequency of adverse intent, and structural security to bound the severity of adverse actions. Our architecture provides the structural half of this equation. The integration of behavioral and structural safety into a unified formal framework — where alignment properties interact with capability constraints — is an important direction for future theoretical work.

### 6.5 Agent governance and accountability

Nature's editorial "We need a new ethics for a world of AI agents" (2025) argues that agentic systems capable of generating and revising their own goals cannot be governed solely by ex-ante specification. Our work provides a technical foundation for this insight: the CapabilityGate's per-event token issuance and deterministic audit trail create an ex-post accountability mechanism that does not require predicting the agent's goals, but instead constrains the *actions* the agent can take regardless of its goals.

The Nature Machine Intelligence editorial "Multi-agent AI systems need transparency" (2026) calls for comprehensive documentation of agent frameworks, cost-benefit analysis of automation, and baseline comparisons. Our architecture directly enables the transparency this editorial demands: every security decision is logged with machine-readable justification, every token issuance is traceable to a specific event batch, and the deterministic Gate allows direct comparison of security properties with alternative approaches.

---

## 7. Limitations and Future Work

### 7.1 Semantic attacks

The CapabilityGate enforces structural constraints — *which tools* can be called with *what data*. It does not evaluate the *semantic appropriateness* of the content written or actions taken within those constraints. An agent authorized to write to a configuration file can write malicious configuration content. Defense against semantic attacks requires application-layer validation (which our system supports through its config validation pipeline) or future advances in LLM alignment — and represents the boundary between structural security (our contribution) and behavioral safety (an orthogonal research direction).

### 7.2 Taint tracking: evasion and usability trade-offs

The dual-layer taint tracking system presents two distinct limitations that pull in opposite directions:

**Evasion risk (false negatives).** Sophisticated LLM reasoning chains that synthesize external data with internal knowledge in multi-step processes may produce outputs whose connection to the original external data is undetectable at the content level (TaintStore fingerprints do not match). CausalTaintTracker mitigates this at the context level, but — as discussed in Section 3.3 — it operates per event batch and resets between batches, leaving a cross-batch contamination gap.

**Over-restriction risk (false positives).** CausalTaintTracker's coarse granularity creates a practical usability tension: once any external data enters the LLM's context within a single batch (e.g., the agent calls `web_fetch` to check weather), *all* subsequent external actions in that batch are blocked — including legitimate operations unrelated to the fetched content (e.g., sending a pre-composed work email). In practice, this means that batches mixing information retrieval and outbound communication will experience frequent false positives. Users may perceive the system as obstructive, potentially leading to workarounds that undermine the security model.

This tension is fundamental, not incidental. A taint tracker that is precise enough to avoid false positives (tracking exactly which tokens derive from external data through the LLM's opaque reasoning process) would require interpretability capabilities that do not yet exist. A tracker that is conservative enough to eliminate false negatives (blocking all external actions whenever external data has ever been observed) would render the agent unusable for most practical workflows. Our dual-layer design — TaintStore for precision, CausalTaintTracker for recall — represents a pragmatic middle ground, but the optimal calibration of this trade-off remains an open research question.

### 7.3 Policy completeness

The `EVENT_TOOL_GRANTS` mapping and `DEFAULT_TOOL_POLICIES` are manually curated. Incomplete mappings can result in either over-permissive tokens (security risk) or under-permissive tokens (functionality loss). Automated policy derivation — potentially through static analysis of tool schemas or empirical observation of tool usage patterns — is an important area for future work.

### 7.4 Formal verification

While we provide a proof sketch for the Agent Authority Impossibility and argue for the sufficiency of the Rule of Two, full formal verification using model checking or theorem proving frameworks (TLA+, Coq, Lean) remains future work. The deterministic nature of the CapabilityGate makes it particularly amenable to such verification.

---

## 8. Conclusion and Outlook

The Agent Authority Problem reveals that the security crisis facing autonomous AI agents is not a matter of insufficient engineering effort but of **paradigmatic misfit**: the ambient authority model, inherited from an era when executing principals had deterministic intent, cannot secure systems whose intent emerges from — and can be corrupted by — the data they process. This is a permanent structural property rooted in the inherent ambiguity of natural language, not a temporary limitation that future models will overcome.

Our event-driven capability architecture demonstrates that this impossibility can be dissolved — not by making LLMs more robust, but by ensuring that security enforcement operates through a fundamentally different channel than the one exposed to adversarial data. The principle of **heterogeneous security enforcement** — that the safety layer and the protected system must not share an attack surface — is, we believe, the central insight for the field going forward.

**Open questions.** Several challenges remain at the frontier. First, the cross-batch context contamination gap (Section 3.3) demands a principled framework for managing session-level taint that balances security with usability — a problem that may ultimately require advances in LLM interpretability to solve optimally. Second, the semantic attack boundary (Section 7.1) — where the Gate can constrain *which* tools are invoked but not *what content* is produced — marks the interface between structural security (our contribution) and behavioral safety (alignment research). Whether these two research programs can be unified into a single formal framework is an open theoretical question. Third, formal verification of the Rule of Two's sufficiency and the CapabilityGate's correctness (Section 7.4), while tractable given the architecture's deterministic nature, awaits rigorous treatment.

**A call to the field.** The SQL injection parallel (Section 2.2) is both a warning and a source of hope. It took the database security community 15 years to transition from sanitization-based defenses to the architectural solution of parameterized queries. The agent security community cannot afford a 15-year transition. The systems being deployed today — with access to email, financial accounts, medical records, and critical infrastructure — create immediate, consequential risk. Every month of delay is measured not in theoretical vulnerability but in real credentials exfiltrated, real funds misdirected, and real trust eroded.

We submit that the transition from ambient authority to deterministic, event-driven capability security is not one option among many. For any domain where autonomous agents perform consequential actions, it is the minimum viable security architecture. The question is not *whether* this transition will occur, but whether it will occur proactively — guided by architectural insight — or reactively, forced by a crisis that makes the OpenClaw incident look like a rehearsal.

---

## References

*(To be completed with full citations)*

1. Dennis, J. B. & Van Horn, E. C. Programming semantics for multiprogrammed computations. *Commun. ACM* 9, 143–155 (1966).
2. Google DeepMind. Defeating Prompt Injections by Design. *arXiv:2503.18813* (2025).
3. Nature Editorial. We need a new ethics for a world of AI agents. *Nature* 644, 8075 (2025).
4. Nature Editorial. Let 2026 be the year the world comes together for AI safety. *Nature* (2025).
5. Nature Machine Intelligence Editorial. Multi-agent AI systems need transparency. *Nat. Mach. Intell.* (2026).
6. OWASP. Top 10 for Large Language Model Applications. (2024).
7. Nature Communications. Risks of AI scientists: prioritizing safeguarding over autonomy. *Nat. Commun.* (2025).
8. Xiao et al. Agentic AI Security: Threats, Defenses, Evaluation, and Open Challenges. *arXiv:2510.23883* (2025).
9. CVE-2026-25253. OpenClaw Remote Code Execution via Cross-Site WebSocket Hijacking. CVSS 8.8 (2026).
10. CVE-2025-32711. EchoLeak: Microsoft Copilot Sensitive Data Exfiltration via Email Prompt Injection. (2025).
