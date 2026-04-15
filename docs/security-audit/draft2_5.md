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

We provide formal definitions, theorem statements, and proofs in Supplementary Notes 1-5: the Agent Authority Impossibility (Theorem 1, Note 1), Rule of Two sufficiency (Theorem 2, Note 2), CausalTaintTracker monotonicity (Lemma 1 + Proposition 1, Note 3), privilege exposure bound (Proposition 3, Note 4), and Gate determinism and LLM-independence (Theorem 4, Note 5). Full machine-verified proofs using theorem provers (TLA+, Coq, Lean) remain future work.

---

## 8. Conclusion and Outlook

The Agent Authority Problem reveals that the security crisis facing autonomous AI agents is a matter of paradigmatic misfit: the ambient authority model cannot secure systems whose intent emerges from — and can be corrupted by — the data they process. This property holds for any architecture where data and instructions share a processing channel, which includes all current and foreseeable LLM-based agent systems.

Our event-driven capability architecture demonstrates that this problem can be circumvented — not by making LLMs more robust within the ambient authority model, but by replacing that model with one in which security enforcement operates through a fundamentally different channel than the one exposed to adversarial data. The principle of **heterogeneous security enforcement** — that the safety layer and the protected system must not share an attack surface — is the central insight.

Three challenges remain at the frontier. The cross-batch contamination gap demands a principled framework for session-level taint management. The semantic attack boundary marks the interface between structural security and behavioral safety. And formal verification of the architecture's invariants, while tractable, awaits rigorous treatment.

The SQL injection parallel is both warning and hope. It took the database community 15 years to transition from sanitization to parameterized queries. The agent security community cannot afford that timeline. We argue that the class of architectures providing event provenance, cascade enforcement, batch atomicity, and deterministic gating — of which our EventBus-CapabilityGate system is one concrete instance — represents a necessary direction for any domain where autonomous agents perform consequential actions. Alternative architectures achieving equivalent structural properties (such as CaMeL's Privileged/Quarantined separation¹²) may emerge; the critical requirement is not any specific implementation, but the abandonment of ambient authority as the organizing principle for agent security.

---

## Methods

### Event model and EventBus

All system triggers are normalized into a typed Event — a frozen (immutable) dataclass with fields: `id` (12-character hex UUID), `source` (originating module: timer, cron, file, process, network, custom, etc.), `type` (event class, e.g. `channel.message_received`, `file.modified`), `priority` (CRITICAL > HIGH > NORMAL > LOW), `payload` (event-specific dictionary), `origin_chain` (immutable tuple tracing provenance across event re-emissions), and `cascade_depth` (integer, automatically incremented by the framework when a consumer injects a child event; hard limit: 20).

The EventBus receives events through registered EventSources, applies an EventInjectionGate (per-source policy enforcement: event type whitelist, rate limiting via sliding 1-second window, payload size cap, cascade depth check, CRITICAL priority requires an unforgeable EscalationToken), then routes through an EventFilter (debounce with 0.5s window, priority-based routing) into a priority queue (max capacity: 10,000; overflow: lowest-priority eviction). The EventConsumer dequeues event batches (2-second collection window, max 20 events) for processing.

### CapabilityIssuer: event-to-token mapping

For each event batch, the CapabilityIssuer performs:

1. **Trust classification.** `classify_event_trust(events)` inspects each event's `source` and `origin_chain`. Events from the 10 internal modules (timer, cron, file, process, network, node, discovery, contract, task, skill_source) are classified LOCAL_TRUSTED. Events from authenticated remote devices are REMOTE_VERIFIED. Events from custom/unknown sources without internal module markers in their origin_chain are REMOTE_OPEN.

2. **Tool grant selection.** A static mapping `EVENT_TOOL_GRANTS` maps event types to minimal tool sets (e.g., `timer.heartbeat_due` → {read_file, list_directory, memory_search}; `cron.job_due` → adds write_file, execute_code, bash). For REMOTE_OPEN events, high-risk tools (bash, run_command, daemon_restart, send_email, curl, wget) are removed from the grant regardless of event type.

3. **Path inference.** If the event payload contains a `path` field, the issuer resolves it to an absolute path and grants access to its parent directory via glob pattern.

4. **Rule-of-Two flag evaluation.** The issuer examines whether the granted tool set contains untrusted-input tools (web_fetch, web_search), sensitive-data tools (read_file with sensitive path patterns), and external-action tools (send_email, curl, bash). If all three categories are present, the external-action tools are removed.

5. **Token construction.** A frozen CapabilityToken is created with: `granted_tools` (frozenset), `denied_tools` (frozenset, priority over grants), `granted_paths` (tuple of glob patterns), `ttl` (300 seconds), and Rule-of-Two flags.

### CapabilityGate: three-check verification

For each tool call `(tool_name, tool_args)` proposed by the LLM:

**Check 1 (Token).** Verify: (a) token TTL has not expired (timestamp comparison); (b) `tool_name ∈ granted_tools`; (c) `tool_name ∉ denied_tools`. If any fails → DENY.

**Check 2 (Taint).** For each string value in `tool_args`: (a) query TaintStore by computing SHA-256 of the normalized value and checking against registered fingerprints; (b) if no fingerprint match, check for registered high-taint substrings (emails, URLs, paths extracted during registration); (c) if no substring match, check for boundary markers (`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>`). If the returned taint level exceeds the tool's policy for that parameter (defined in `DEFAULT_TOOL_POLICIES`, e.g., `send_email.to` requires taint ≤ USER) → DENY.

**Check 3 (Structural).** Verify: (a) if the tool requires path checking, the path argument matches at least one `granted_paths` glob; (b) the tool's call count in this round does not exceed `max_calls_per_round` (default: 20); (c) the CausalRuleOfTwo constraint is not violated for this batch.

### TaintStore registration and query

On tool execution, the return value is registered: `TaintStore.register(content, taint_level, origin_tool)`. Registration computes a SHA-256 fingerprint of the normalized content (first 16 hex characters), extracts substrings matching email/URL/path patterns for content >5KB, and stores the record with a TTL of 600 seconds. Maximum capacity: 10,000 records with LRU eviction. Query follows a short-circuit strategy: boundary markers → fingerprint match → substring match → None (unknown, fail-open).

The complementary CausalTaintTracker updates via τ_round(k) = max(τ_round(k-1), τ_result(k)), which we prove is monotonically non-decreasing (Supplementary Note 3, Theorem 3). Once τ_round reaches EXTERNAL, it remains EXTERNAL for the remainder of the batch — formalizing the "irrevocable contamination" property.

### Code and data availability

The architecture is implemented in Python as part of the ShadowClaw project. Source code is available at [repository URL]. The tool enumeration (55 tools across 16 module categories) is provided in Supplementary Table 1.

---

## References

1. Gabriel, I. et al. We need a new ethics for a world of AI agents. *Nature* **644**, 291–294 (2025).
2. OpenClaw Project. Security advisories GHSA-2026-25253 (RCE), GHSA-2026-25254, GHSA-2026-25255 (command injection). GitHub https://github.com/openclaw/openclaw/security/advisories (2026).
3. Sangfor Technologies. OpenClaw Security Risks: From Vulnerabilities to Supply Chain Abuse. Report at https://www.sangfor.com/blog/cybersecurity/openclaw-ai-agent-security-risks-2026 (accessed 10 April 2026).
4. Censys. Exposed OpenClaw Instances Analysis. Report at https://censys.io/openclaw-exposed-instances (accessed 10 April 2026).
5. KuCoin Research. AI Trading Agent Vulnerability: $45M Crypto Security Breach. Report at https://www.kucoin.com/blog (accessed 10 April 2026).
6. OWASP. Top 10 for Large Language Model Applications v2.0. https://owasp.org/www-project-top-10-for-large-language-model-applications/ (2025).
7. Nature Editorial. Let 2026 be the year the world comes together for AI safety. *Nature* **637**, 8–9 (2025).
8. Nature Machine Intelligence Editorial. Multi-agent AI systems need transparency. *Nat. Mach. Intell.* **8**, 1–2 (2026).
9. MITRE. CVE-2026-25253: OpenClaw Remote Code Execution via Cross-Site WebSocket Hijacking. CVSS 8.8. https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-25253 (2026).
10. MITRE. CVE-2025-32711 (EchoLeak): Microsoft Copilot Sensitive Data Exfiltration via Email Prompt Injection. https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-32711 (2025).
11. Dennis, J. B. & Van Horn, E. C. Programming semantics for multiprogrammed computations. *Commun. ACM* **9**, 143–155 (1966).
12. Debenedetti, E. et al. Defeating Prompt Injections by Design. Preprint at https://arxiv.org/abs/2503.18813 (2025).
13. Greshake, K. et al. Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection. *Proc. AISec '23*, 79–90 (2023).
14. Xiao, A. et al. Agentic AI Security: Threats, Defenses, Evaluation, and Open Challenges. Preprint at https://arxiv.org/abs/2510.23883 (2025).
15. Zhan, Q. et al. InjecAgent: Benchmarking Indirect Prompt Injections in Tool-Integrated LLM Agents. *Findings of ACL 2024* (2024).
16. Yi, J. et al. Benchmarking and Defending Against Indirect Prompt Injection Attacks on Large Language Models. Preprint at https://arxiv.org/abs/2312.14197 (2023).
17. Partnership on AI. AI Incident Database. https://incidentdatabase.ai (accessed 10 April 2026).
18. Wallace, E. et al. The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions. Preprint at https://arxiv.org/abs/2404.13208 (2024).
19. EU Parliament. Regulation (EU) 2024/1689 laying down harmonised rules on artificial intelligence (AI Act). *Official Journal of the European Union* L 2024/1689 (2024).
20. Birgisson, A. et al. Macaroons: Cookies with Contextual Caveats for Decentralized Authorization in the Cloud. *Proc. NDSS* (2014).
21. FINOS. Agent Authority Least Privilege Framework. https://air-governance-framework.finos.org (accessed 10 April 2026).
22. Suh, G. E. et al. Secure Program Execution via Dynamic Information Flow Tracking. *Proc. ASPLOS* (2004).
23. Ouyang, L. et al. Training language models to follow instructions with human feedback. *Proc. NeurIPS* **35**, 27730–27744 (2022).
24. Bai, Y. et al. Constitutional AI: Harmlessness from AI Feedback. Preprint at https://arxiv.org/abs/2212.08073 (2022).
25. European Commission. EU AI Act Articles 9, 12, 14, 15. Regulation (EU) 2024/1689 (2024).
26. The White House. Executive Order 14110: Safe, Secure, and Trustworthy Development and Use of Artificial Intelligence. *Federal Register* **88**, 75191 (2023).
27. China Cyberspace Administration. Interim Measures for the Management of Generative AI Services. (2023).
28. Berditchevskaia, A. et al. Risks of AI scientists: prioritizing safeguarding over autonomy. *Nat. Commun.* **16**, 5003 (2025).
29. Cisco Security. Personal AI Agents like OpenClaw Are a Security Nightmare. Report at https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare (accessed 10 April 2026).
30. Qualys. Anatomy of an Autonomous AI Agent Risk: Qualys ETM on OpenClaw. Report at https://blog.qualys.com/product-tech/2026/04/13/anatomy-autonomous-ai-agent-risk-qualys-etm-openclaw (accessed 10 April 2026).
31. MarketsandMarkets. AI Agents Market Size, Share & Industry Trends Analysis Report. Report at https://www.marketsandmarkets.com (accessed 10 April 2026).
