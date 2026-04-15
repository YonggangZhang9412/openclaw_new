# Supplementary Note 1: Agent Authority Impossibility

## Formal Model

**Definition 1 (Security Domain).** A security domain is a finite set D = {d₁, d₂, ..., dₘ} of principals. In the agent setting, D includes at minimum the operator (user), the agent, and external entities (untrusted data sources).

**Definition 2 (Tool Signature).** A tool signature is a tuple t = (name, params, returns) where name ∈ Names, params: ParamName → Type, and returns: Type. We write T = {t₁, ..., tₙ} for the finite set of all tool signatures in the system.

**Definition 3 (Sensitive Tools).** T_s ⊆ T is the subset of tools that access sensitive resources. Formally, t ∈ T_s if and only if invoking t can read from or write to resources classified above a minimum security threshold.

**Definition 4 (Agent System).** An agent system is a tuple M = (S, s₀, T, A, δ, L, E) where:
- S is a set of states (including LLM context, session history, and tool results)
- s₀ ∈ S is the initial state
- T is a finite set of tool signatures
- A = {(t, a) | t ∈ T, a ∈ Args(t)} is the set of tool invocations (a tool paired with concrete arguments)
- δ: S × A → S is the deterministic state transition function (executing a tool invocation updates system state)
- L: S → Dist(A ∪ {⊥}) is the LLM planning function, mapping the current state to a probability distribution over tool invocations (or ⊥ for no action). L is probabilistic and opaque.
- E: A × S → {ALLOW, DENY} is the safety enforcement function

**Definition 5 (Permission Model).** A permission model is a function P: S → 2^T assigning a set of permitted tools to each state. We say P is:
- **Ambient** if P(s) = P₀ for all reachable states s (permissions do not depend on state)
- **Event-scoped** if P(s) may vary with s (permissions depend on the current processing context)

**Definition 6 (Data-Instruction Conflation).** An agent system M exhibits data-instruction conflation if there exists no function partition: S → (S_instr × S_data) that separates the instruction-relevant and data-relevant components of state such that L depends only on S_instr. That is, the LLM's output distribution is influenced by the full state including untrusted data:

∃ s, s' ∈ S: s|_instr = s'|_instr ∧ s|_data ≠ s'|_data ∧ L(s) ≠ L(s')

where s|_instr and s|_data denote the instruction and data components respectively.

**Definition 7 (Conditions C1, C2, C3).** Let M = (S, s₀, T, A, δ, L, E) be an agent system with permission model P.

- **(C1) Autonomous data processing.** There exists a reachable state s ∈ S such that s contains data d originating from an untrusted external source (source(d) ∉ Trusted), and d was incorporated into s without per-item human approval.

- **(C2) Privileged resource access.** P(s) ∩ T_s ≠ ∅ for all reachable states s. That is, the agent always has access to at least one sensitive tool.

- **(C3) LLM-independent safety.** E(·, ·) does not depend on L. Formally, E is a deterministic function of (a, P(s)) only:

∀ a ∈ A, ∀ s₁, s₂ ∈ S: P(s₁) = P(s₂) ⟹ E(a, s₁) = E(a, s₂)

That is, E's decision depends only on the tool invocation and the permission set, not on the LLM's internal state or the content of the processing context.

## Theorem Statement

**Theorem 1 (Agent Authority Impossibility).** Let M be an agent system with ambient permission model P(s) = P₀ that exhibits data-instruction conflation (Definition 6). Then C1, C2, and C3 cannot be simultaneously satisfied.

## Proof

Assume for contradiction that all three conditions hold.

**Step 1 (Adversarial input existence).** By C1, there exists a reachable state s_u containing untrusted data d_u. By data-instruction conflation (Definition 6), L's output distribution is influenced by d_u. Therefore, there exists adversarial data d* that can be substituted for d_u such that:

∃ t_s ∈ T_s, ∃ a* ∈ Args(t_s): Pr[L(s*) = (t_s, a*)] > 0

where s* is the state with d* incorporated. This step relies on the following assumption, which we state explicitly:

**Assumption A1 (LLM Manipulability).** For any target tool invocation (t, a) with t ∈ P₀, there exists adversarial data d* such that when d* is incorporated into the agent's state, Pr[L(s*) = (t, a)] > 0.

This assumption is empirically validated: prompt injection attacks have been demonstrated against every major LLM family (GPT-4, Claude, Gemini, Llama) with non-zero success rates across diverse target actions¹³'¹⁵'¹⁶.

**Step 2 (Enforcement dilemma).** By C2, P₀ ∩ T_s ≠ ∅. Let t_s ∈ P₀ ∩ T_s. Consider two states:
- s_benign: operator legitimately requests invocation of t_s with arguments a_benign
- s_adv: adversarial data d* causes L to propose (t_s, a*) with malicious arguments

By C3 (Definition 7), E depends only on the invocation and the permission set:

E((t_s, a_benign), s_benign) = f(t_s, a_benign, P₀)
E((t_s, a*), s_adv) = f(t_s, a*, P₀)

**Step 3 (Argument indistinguishability).** One might object that E could distinguish a_benign from a* based on argument structure. However, under Assumption A1, the adversary can choose d* such that a* is structurally indistinguishable from a_benign — for example, a* could be a well-formed email address, a valid file path, or a syntactically correct shell command. Formally:

**Assumption A2 (Argument Mimicry).** For any legitimate argument value a_benign ∈ Args(t_s), there exists adversarial a* ∈ Args(t_s) such that a* is syntactically valid, type-correct, and indistinguishable from a_benign by any function that does not have access to the LLM's reasoning trace or the data provenance of a*.

Under A2, any enforcement function E that allows (t_s, a_benign) must also allow (t_s, a*), since E cannot distinguish them without accessing provenance information — which would require E to depend on the LLM's context (the origin of the argument), violating C3.

**Step 4 (Contradiction).** Therefore:
- If E allows invocations of t_s with well-formed arguments (to preserve C2, enabling legitimate use), then E also allows adversarial invocations with mimicked arguments → system is unsafe.
- If E blocks all invocations of t_s → agent cannot use sensitive tools → C2 is violated.
- If E inspects argument provenance (where the argument value originated in the LLM's context) → E depends on L's internal state → C3 is violated.

All three alternatives violate at least one condition. Therefore C1 ∧ C2 ∧ C3 is unsatisfiable under ambient authority with data-instruction conflation. ∎

## Remarks

**On Assumption A1.** This is an empirical assumption, not a mathematical axiom. If a future LLM achieves perfect resistance to all forms of prompt injection (Pr[L produces adversarial output | adversarial input] = 0 for all adversarial inputs), Theorem 1 would not apply. However, as argued in Section 2.3 of the main text, data-instruction conflation makes this asymptotically unlikely for any system where the LLM must reason about external data.

**On Assumption A2.** This assumption holds for all current tool interfaces where arguments are typed values (strings, paths, URLs, numbers). It would not hold for a system where every argument carries a cryptographic provenance certificate — but such a system would implement a form of taint tracking, which is part of our proposed solution.

**Relationship to our architecture.** Our event-scoped capability architecture circumvents Theorem 1 by replacing ambient authority (P(s) = P₀) with event-scoped authority (P(s) varies with state). Under event-scoped authority, when s contains untrusted data (C1), P(s) is automatically restricted to exclude sensitive tools, breaking the precondition for Step 2.
