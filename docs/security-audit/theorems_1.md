# Formal Theorems and Proofs — Supplementary Note

---

## Theorem 1: Agent Authority Impossibility

### 1.1 Formal Model

**Definition 1 (Agent System).** An agent system is a tuple A = (T, P, C, L, E) where:
- T = {t₁, t₂, ..., tₙ} is a finite set of **tools** (executable actions)
- P ⊆ T is the **permission set** (tools the agent is authorized to invoke)
- C is the **processing channel**: an ordered sequence of tokens consumed by the LLM, containing both system instructions and external data
- L: C → T × Args is the **LLM function** mapping channel content to tool invocations
- E: T × Args × P → {ALLOW, DENY} is the **safety enforcement function**

**Definition 2 (Ambient Authority).** An agent system operates under ambient authority if P(t) = P₀ for all time steps t during a session, where P₀ is determined at authentication time and does not vary with the content of C.

**Definition 3 (Sensitive Tools).** T_s ⊆ T is the subset of tools that access sensitive resources (filesystem credentials, communication channels, financial endpoints).

**Definition 4 (Data-Instruction Conflation).** A processing channel C exhibits data-instruction conflation if external data d and system instructions i occupy the same token sequence, i.e., C = (..., i₁, ..., d₁, ..., i₂, ..., d₂, ...) with no type-level separation between d-tokens and i-tokens.

**Definition 5 (Conditions C1, C2, C3).**
- **(C1)** Autonomous data processing: ∃ d_u ∈ C such that d_u originates from an untrusted external source, and d_u enters C without per-item human approval.
- **(C2)** Privileged resource access: P₀ ∩ T_s ≠ ∅.
- **(C3)** LLM-independent safety: E does not depend on the correctness of L's output. Formally: E(t, args, P) is a deterministic function of (t, args, P) only, and does not use L, C, or any LLM internal state as input.

### 1.2 Theorem Statement

**Theorem 1 (Agent Authority Impossibility).** For any agent system A = (T, P₀, C, L, E) operating under ambient authority with data-instruction conflation in C, the conditions C1, C2, and C3 cannot be simultaneously satisfied.

### 1.3 Proof

Assume for contradiction that C1, C2, and C3 all hold under ambient authority.

**Step 1.** By C2, P₀ ∩ T_s ≠ ∅. Let t_s ∈ P₀ ∩ T_s be a sensitive tool that the agent is authorized to invoke.

**Step 2.** By C1, there exists untrusted data d_u that enters C without human approval. By data-instruction conflation, d_u occupies the same token sequence as system instructions. Therefore, an adversary can construct d* (adversarial input) such that d* ∈ C and d* is designed to cause L to output (t_s, args*) where args* achieves the adversary's goal (e.g., exfiltrating sensitive data).

**Step 3.** Consider two scenarios:
- **Scenario A (Benign):** L(C_benign) = (t_s, args_benign) — the user legitimately requests invocation of t_s.
- **Scenario B (Adversarial):** L(C_adversarial) = (t_s, args*) — adversarial data d* in C causes L to invoke t_s with malicious arguments.

In both scenarios, the tool invocation presented to E is of the form (t_s, args, P₀).

**Step 4.** By C3, E is a deterministic function of (t, args, P₀) only. E cannot inspect C to determine whether the invocation originated from user intent (Scenario A) or adversarial injection (Scenario B). Since t_s ∈ P₀ in both scenarios, E must make its decision based solely on (t_s, args, P₀).

**Step 5.** E faces a dilemma:
- If E(t_s, args, P₀) = ALLOW for valid invocations → E must also ALLOW adversarial invocations with the same (t_s, args_format, P₀) structure, because E cannot distinguish them without inspecting C (which would violate C3 by introducing dependence on LLM context).
- If E(t_s, ·, P₀) = DENY for all invocations of t_s → this eliminates the agent's ability to use t_s, contradicting C2 (the agent no longer has effective access to t_s).

**Step 6.** Therefore, under ambient authority with data-instruction conflation:
- Allowing t_s (preserving C2) means adversarial invocations cannot be blocked by E without inspecting L's reasoning (violating C3).
- Blocking t_s (preserving C3) means the agent loses access to sensitive tools (violating C2).

C1, C2, and C3 cannot all hold simultaneously. ∎

### 1.4 Scope Condition

The theorem requires data-instruction conflation (Definition 4). This condition holds for all current LLM architectures, where inputs are concatenated into a single token stream. A hypothetical architecture achieving perfect channel separation — where L never processes d-tokens and i-tokens in the same computational context — would escape this theorem, but would itself constitute a form of the architectural separation proposed in this work.
