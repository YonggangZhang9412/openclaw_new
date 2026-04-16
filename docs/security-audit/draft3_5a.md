## 6. Related Work

### 6.1 Prompt injection defenses

Existing defenses fall into three categories. **Detection-based approaches** (pattern matching, classifier models, perplexity analysis¹⁴) face the fundamental limitation that adversarial encodings are open-ended. **Instruction hierarchy approaches**¹⁸ establish precedence among instruction sources but depend on LLM compliance under adversarial pressure. **Architectural separation approaches**, most notably CaMeL¹², introduce Privileged/Quarantined LLM separation and data provenance tracking. Our work extends CaMeL in three ways: (1) identifying EventBus as a structural prerequisite for capability security, providing unforgeable event provenance and cascade enforcement that CaMeL's architecture does not address; (2) dual-layer taint tracking (content-level + causal-level) versus CaMeL's single-layer provenance; (3) closed-form theoretical bounds on attack probability (Theorem 1), complementing CaMeL's empirical evaluation (67% task success with provable security on AgentDojo). Additionally, CaMeL's dual-LLM architecture incurs 2.7-2.8× token overhead; our CapabilityGate is pure deterministic code requiring zero additional LLM calls for security enforcement.

### 6.2 Capability-based security

Capability-based security, originating with Dennis and Van Horn (1966)¹¹, replaces ambient authority with explicit capability tokens. Our contribution adapts capability security to LLM agents, where capabilities must be scoped not to principal *identity* but to *operational context* — requiring the EventBus infrastructure of Section 3.2. The FINOS Agent Authority Least Privilege Framework²¹ independently advocates similar principles at the governance level; our work provides the architectural implementation with formal guarantees.

### 6.3 Taint tracking and information flow control

Taint tracking has a rich history (Perl's taint mode, DIFT²²). Costa and Köpf³² present Fides, which tracks confidentiality and integrity labels forming a lattice with deterministic policy enforcement for AI agents. Our work differs in two respects: (1) our dual-layer design (content-level TaintStore + causal-level CausalTaintTracker) provides complementary precision and recall, whereas Fides uses a single-layer label system; (2) we provide quantitative probability bounds on attack success (Theorem 1), whereas Fides characterizes enforceable properties qualitatively. Siu et al.³³ propose a framework formalizing four security properties for LLM agents (task alignment, action alignment, source authorization, data isolation); our work provides the architectural implementation and quantitative guarantees for properties analogous to their source authorization and data isolation.

### 6.4 Alignment and structural security

Behavioral safety (RLHF²³, constitutional AI²⁴) addresses *what the agent wants to do*. Structural security (our contribution) addresses *what the agent is allowed to do*. These are complementary: alignment reduces p(injection succeeds); structural security bounds the consequence of successful injection. The mature security posture requires both.

---

## 7. Limitations

### 7.1 Semantic attacks

The CapabilityGate constrains *which tools* are called with *what data*, but cannot evaluate the *semantic appropriateness* of content generated within those constraints.

### 7.2 Taint tracking trade-offs

**False negatives.** LLM reasoning chains may synthesize external data undetectably by content-level fingerprinting. CausalTaintTracker mitigates this but resets per batch, leaving a cross-batch gap.

**False positives.** CausalTaintTracker blocks all external actions in a batch once any external data is observed — including legitimate operations. This tension is fundamental: precise tracking through the LLM's opaque reasoning would require interpretability capabilities that do not exist.

### 7.3 Cross-batch context contamination

External data from prior batches may persist in session history and influence LLM behavior in subsequent locally-trusted batches. CausalTaintTracker could persist across batches at the cost of severe usability penalties.

### 7.4 Policy completeness

Tool classification ($\mathcal{T}_R$, $\mathcal{T}_X$, $\mathcal{T}_U$) and EVENT_TOOL_GRANTS are manually curated. Misclassification weakens the real-world guarantee without affecting formal correctness.

### 7.5 Formal verification

Supplementary Notes provide hand proofs. Machine-verified proofs (TLA+, Coq, Lean) remain future work. The CapabilityGate's deterministic nature makes it amenable to such verification.
