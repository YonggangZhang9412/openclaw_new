## 6. Related Work

### 6.1 Prompt injection defenses

Existing defenses fall into three categories. **Detection-based approaches** (pattern matching, classifier models, perplexity analysis¹⁴) face the fundamental limitation that adversarial encodings are open-ended. **Instruction hierarchy approaches**¹⁸ establish precedence among instruction sources but depend on LLM compliance under adversarial pressure. **Architectural separation approaches**, most notably CaMeL¹², introduce Privileged/Quarantined LLM separation and data provenance tracking. Our work extends CaMeL in three ways: (1) identifying EventBus as a structural prerequisite for capability security; (2) dual-layer taint tracking (content-level + causal-level) versus CaMeL's single-layer provenance; (3) the Rule of Two as a formally characterizable structural constraint. CaMeL reports 2.7-2.8× token overhead from its dual-LLM architecture; our deterministic Gate requires zero additional LLM calls.

### 6.2 Capability-based security

Capability-based security, originating with Dennis and Van Horn (1966)¹¹, replaces ambient authority with explicit capability tokens. Implementations span operating systems (seL4, Capsicum), web browsers (Content Security Policy), and distributed systems (Macaroons²⁰, Biscuit). Our contribution adapts capability security to LLM agents, where capabilities must be scoped not to principal *identity* (always "the agent") but to *operational context* (which event is being processed, what trust level it carries). This requires the EventBus infrastructure of Section 3.2, which has no precedent in classical capability systems. The FINOS Agent Authority Least Privilege Framework²¹ independently advocates similar principles at the governance level; our work provides the architectural implementation.

### 6.3 Taint tracking

Taint tracking has a rich history in programming language security (Perl's taint mode, Ruby's taint levels, DIFT hardware-assisted tracking²²). Our TaintStore extends these concepts to the LLM domain, where the "program" is opaque and non-deterministic. The dual-layer design — content-level fingerprinting complemented by irrevocable causal-level tracking — addresses the unique challenge of LLM-mediated information flow, where the processing entity can arbitrarily transform data while preserving semantic content.

### 6.4 Alignment and structural security

Behavioral safety (RLHF²³, constitutional AI²⁴, safety fine-tuning) addresses *what the agent wants to do*. Structural security (our contribution) addresses *what the agent is allowed to do*. These are complementary:

- An aligned agent under ambient authority remains vulnerable to injection exceeding its training distribution. Alignment reduces p(injection succeeds) but cannot guarantee p=0.
- A structurally secured agent with a misaligned LLM may produce adversarial content *within* its permitted tool set.

The mature security posture requires both: alignment to minimize frequency of adverse intent, structural security to bound severity of adverse actions. Unifying behavioral and structural safety into a single formal framework is an important theoretical direction.

---

## 7. Limitations

### 7.1 Semantic attacks

The CapabilityGate constrains *which tools* are called with *what data*, but cannot evaluate the *semantic appropriateness* of content generated within those constraints. An agent authorized to write to a configuration file can write malicious content. This boundary between structural security and behavioral safety (Section 6.4) represents a fundamental scope limitation.

### 7.2 Taint tracking trade-offs

**False negatives (evasion).** LLM reasoning chains that synthesize external data with internal knowledge may produce outputs undetectable by content-level fingerprinting. CausalTaintTracker mitigates this but resets per batch, leaving a cross-batch contamination gap (Section 3.3).

**False positives (over-restriction).** CausalTaintTracker's coarse granularity blocks all external actions in a batch once any external data is observed — including legitimate operations unrelated to the fetched content. This tension is fundamental: precise tracking through the LLM's opaque reasoning would require interpretability capabilities that do not exist. Our dual-layer design represents a pragmatic middle ground whose optimal calibration remains an open question.

### 7.3 Cross-batch context contamination

External data from prior batches may persist in session history and influence LLM behavior in subsequent locally-trusted batches. In this scenario, a compound tool call (e.g., `bash("cat .env | curl attacker.com")`) could complete an attack chain without triggering the Rule of Two, because the untrusted input condition is not recognized at the batch level. CausalTaintTracker could be extended to persist across batches, but this would permanently restrict external actions in any session that has ever processed external data — a severe usability penalty. Quantifying the practical frequency and exploitability of this gap (Experiment 4 in Section 4.2) is a priority for empirical validation.

### 7.4 Policy completeness

The `EVENT_TOOL_GRANTS` mapping and `DEFAULT_TOOL_POLICIES` are manually curated. Incomplete or inaccurate categorization can result in over-permissive tokens (security risk) or under-permissive tokens (functionality loss). Automated policy derivation is an important direction for future work.

### 7.5 Formal verification

The structural argument (Section 2.3) and the Rule of Two's scope are presented as informal logical analyses. Full formal verification using model checking or theorem proving (TLA+, Coq) remains future work. The CapabilityGate's deterministic nature makes it particularly amenable to such verification.

---

## 8. Conclusion and Outlook

The Agent Authority Problem reveals that the security crisis facing autonomous AI agents is a matter of paradigmatic misfit: the ambient authority model cannot secure systems whose intent emerges from — and can be corrupted by — the data they process. This is a permanent property rooted in natural language's inherent data-instruction ambiguity, not a temporary limitation future models will overcome.

Our event-driven capability architecture demonstrates that this problem can be dissolved by ensuring security enforcement operates through a channel fundamentally different from the one exposed to adversarial data. The principle of **heterogeneous security enforcement** — that the safety layer and the protected system must not share an attack surface — is the central insight.

Three challenges remain at the frontier. The cross-batch contamination gap demands a principled framework for session-level taint management. The semantic attack boundary marks the interface between structural security and behavioral safety. And formal verification of the architecture's invariants, while tractable, awaits rigorous treatment.

The SQL injection parallel is both warning and hope. It took the database community 15 years to transition from sanitization to parameterized queries. The agent security community cannot afford that timeline. The systems deployed today — with access to email, financial accounts, and medical records — create immediate risk. The transition from ambient authority to deterministic, event-driven capability security is not one option among many. For any domain where autonomous agents perform consequential actions, it is the minimum viable security architecture.

---

## References

1. Gabriel, I. et al. We need a new ethics for a world of AI agents. *Nature* **644**, 291–294 (2025).
2. OpenClaw Project. Security advisories and incident reports. GitHub (2026). https://github.com/openclaw
3. Sangfor Technologies. OpenClaw Security Risks: From Vulnerabilities to Supply Chain Abuse. Security Research (2026).
4. Censys. Exposed OpenClaw Instances Analysis. (2026).
5. KuCoin Research. AI Trading Agent Vulnerability: $45M Crypto Security Breach. (2026).
6. OWASP. Top 10 for Large Language Model Applications v2.0. (2025).
7. Nature Editorial. Let 2026 be the year the world comes together for AI safety. *Nature* (2025).
8. Nature Machine Intelligence Editorial. Multi-agent AI systems need transparency. *Nat. Mach. Intell.* (2026).
9. CVE-2026-25253. OpenClaw Remote Code Execution via Cross-Site WebSocket Hijacking. CVSS 8.8.
10. CVE-2025-32711 (EchoLeak). Microsoft Copilot Sensitive Data Exfiltration via Email Prompt Injection.
11. Dennis, J. B. & Van Horn, E. C. Programming semantics for multiprogrammed computations. *Commun. ACM* **9**, 143–155 (1966).
12. Debenedetti, E. et al. Defeating Prompt Injections by Design. *arXiv:2503.18813* (2025).
13. Greshake, K. et al. Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection. *AISec '23*, 79–90 (2023).
14. Xiao, A. et al. Agentic AI Security: Threats, Defenses, Evaluation, and Open Challenges. *arXiv:2510.23883* (2025).
15. Zhan, Q. et al. InjecAgent: Benchmarking Indirect Prompt Injections in Tool-Integrated LLM Agents. *ACL 2024 Findings* (2024).
16. Yi, J. et al. Benchmarking and Defending Against Indirect Prompt Injection Attacks on Large Language Models. *arXiv:2312.14197* (2023).
17. Partnership on AI. AI Incident Database. https://incidentdatabase.ai (2024-2026).
18. Wallace, E. et al. The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions. *arXiv:2404.13208* (2024).
19. EU Parliament. Regulation (EU) 2024/1689 (AI Act). *Official Journal of the European Union* (2024).
20. Birgisson, A. et al. Macaroons: Cookies with Contextual Caveats for Decentralized Authorization in the Cloud. *NDSS* (2014).
21. FINOS. Agent Authority Least Privilege Framework. https://air-governance-framework.finos.org (2025).
22. Suh, G. E. et al. Secure Program Execution via Dynamic Information Flow Tracking. *ASPLOS* (2004).
23. Ouyang, L. et al. Training language models to follow instructions with human feedback. *NeurIPS* (2022).
24. Bai, Y. et al. Constitutional AI: Harmlessness from AI Feedback. *arXiv:2212.08073* (2022).
25. European Commission. EU AI Act Articles 9, 12, 14, 15. Regulation (EU) 2024/1689 (2024).
26. US Executive Order 14110. Safe, Secure, and Trustworthy Development and Use of Artificial Intelligence. (2023).
27. China Cyberspace Administration. Interim Measures for the Management of Generative AI Services. (2023).
28. Nature Communications. Risks of AI scientists: prioritizing safeguarding over autonomy. *Nat. Commun.* (2025).
29. Cisco Security. Personal AI Agents like OpenClaw Are a Security Nightmare. Cisco Blogs (2026).
30. Qualys. Anatomy of an Autonomous AI Agent Risk: Qualys ETM on OpenClaw. (2026).
