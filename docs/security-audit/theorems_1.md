# Supplementary Note 1: Agent Authority Impossibility

## Formal Model

**Definition 1 (Tool Signature).** A tool signature is a triple t = (name, Params, Returns) where name ∈ 𝒩 is an identifier, Params is a finite set of typed parameter slots, and Returns is a return type. We write 𝒯 = {t₁, ..., tₙ} for the finite set of all tool signatures.

**Definition 2 (Sensitive Tools).** A subset 𝒯_s ⊆ 𝒯 denotes tools that access sensitive resources. Membership is determined by a labeling function sens: 𝒯 → {0, 1}; we define 𝒯_s = {t ∈ 𝒯 | sens(t) = 1}.

**Definition 3 (Agent System).** An agent system is a tuple ℳ = (𝒮, s₀, 𝒯, 𝒜, δ, L, E) where:
- 𝒮 is a (possibly infinite) set of states encoding LLM context, session history, and tool results
- s₀ ∈ 𝒮 is the initial state
- 𝒯 is a finite set of tool signatures (Definition 1)
- 𝒜 = {(t, a) | t ∈ 𝒯, a ∈ Vals(Params(t))} is the set of concrete tool invocations
- δ: 𝒮 × 𝒜 → 𝒮 is the state transition function
- L: 𝒮 → Δ(𝒜 ∪ {⊥}) is the LLM planning function, mapping the current state to a probability distribution over tool invocations or ⊥ (no action). L is stochastic and opaque.
- E: 𝒜 × 𝒮 → {ALLOW, DENY} is the safety enforcement function

**Definition 4 (Permission Model).** A permission model is a function P: 𝒮 → 𝒫(𝒯) assigning a set of permitted tools to each state. We distinguish:
- **Ambient authority**: P(s) = P₀ for all reachable s ∈ 𝒮 (constant)
- **Event-scoped authority**: P may vary with s

**Definition 5 (Data-Instruction Conflation).** An agent system ℳ exhibits data-instruction conflation if the LLM's output distribution is sensitive to untrusted data in the state. Formally: there exist states s, s' ∈ 𝒮 differing only in their untrusted data component such that L(s) ≠ L(s'), i.e.,

∃ s, s' ∈ 𝒮, ∃ a ∈ 𝒜: s \ data(s) = s' \ data(s') ∧ Pr[L(s) = a] ≠ Pr[L(s') = a]

where data(s) ⊂ s denotes the untrusted external data present in state s. Intuitively, the LLM cannot be isolated from the data it processes.

**Definition 6 (Conditions C1, C2, C3).**

**(C1)** ∃ s ∈ Reach(ℳ): data(s) ≠ ∅ ∧ ∀ d ∈ data(s): source(d) ∉ Trusted ∧ approved(d) = false

(The agent reaches a state containing untrusted, unapproved external data.)

**(C2)** ∀ s ∈ Reach(ℳ): P(s) ∩ 𝒯_s ≠ ∅

(In every reachable state, the agent has access to at least one sensitive tool.)

**(C3)** ∀ a ∈ 𝒜, ∀ s₁, s₂ ∈ 𝒮: P(s₁) = P(s₂) ⟹ E(a, s₁) = E(a, s₂)

(The enforcement function's decision depends only on the invocation and the permission set — it does not inspect LLM context, reasoning trace, or data provenance.)

## Assumptions

**Assumption 1 (LLM Manipulability).** For any tool invocation (t, a) with t ∈ P₀, there exists adversarial data d* such that, when d* is incorporated into state s yielding s*:

Pr[L(s*) = (t, a)] > 0

Empirical support: indirect prompt injection benchmarks report attack success rates of 20-80% across GPT-4, Claude, Gemini, and Llama model families on tool-integrated agent tasks¹⁵˒¹⁶. CaMeL¹² reports that without defenses, 100% of benchmark attacks succeed on some models. Assumption 1 requires only that this probability is non-zero, which is strictly weaker than observed rates.

**Assumption 2 (Argument Mimicry).** For any t ∈ 𝒯_s and legitimate argument vector a_b ∈ Vals(Params(t)), there exists adversarial a* ∈ Vals(Params(t)) such that a_b and a* lie in the same equivalence class under E's observable inputs:

∀ s ∈ 𝒮: E((t, a_b), s) = E((t, a*), s)

That is, E cannot distinguish a_b from a* without access to information beyond (t, a, P(s)) — specifically, without access to the provenance of a's values within L's reasoning context.

## Theorem

**Theorem 1 (Agent Authority Impossibility).** Let ℳ be an agent system (Definition 3) with ambient authority P(s) = P₀ exhibiting data-instruction conflation (Definition 5). Under Assumptions 1 and 2, conditions C1, C2, and C3 (Definition 6) cannot be simultaneously satisfied.

## Proof

Assume for contradiction that C1 ∧ C2 ∧ C3 hold.

**Step 1.** By C2, ∃ t_s ∈ P₀ ∩ 𝒯_s. Fix such a t_s.

**Step 2.** By C1, ∃ s_u ∈ Reach(ℳ) with data(s_u) ≠ ∅ and source(d) ∉ Trusted for some d ∈ data(s_u). By Definition 5 (conflation), L's output distribution is sensitive to data(s_u). By Assumption 1, ∃ d* such that the modified state s* satisfies Pr[L(s*) = (t_s, a*)] > 0 for some a* ∈ Vals(Params(t_s)).

**Step 3.** Consider two scenarios producing tool invocations evaluated by E:
- *Benign*: L(s_b) = (t_s, a_b) with probability p_b > 0 (legitimate user intent)
- *Adversarial*: L(s*) = (t_s, a*) with probability p* > 0 (injection-induced, Step 2)

By Assumption 2, E((t_s, a_b), s_b) = E((t_s, a*), s*) since a_b and a* are in the same E-equivalence class. Denote this common decision as D.

**Step 4.** Case analysis on D:
- **D = ALLOW**: E permits both the legitimate and adversarial invocations of t_s. Since t_s ∈ 𝒯_s, the adversarial invocation accesses sensitive resources, violating safety. But E has allowed it — so the system is unsafe despite E being present.
- **D = DENY**: E blocks all invocations of t_s with arguments in this equivalence class. Since a_b is a legitimate argument, the operator loses the ability to use t_s for its intended purpose. This means P₀ ∩ 𝒯_s is not effectively accessible — contradicting C2 in practice (the tool is permitted but unusable).
- **E distinguishes based on provenance**: E inspects which state component generated a (was it user intent or injected data?). This requires E(a, s₁) ≠ E(a, s₂) for some s₁, s₂ with P(s₁) = P(s₂) — violating C3 (Definition 6).

All cases lead to contradiction. Therefore C1 ∧ C2 ∧ C3 is unsatisfiable under ambient authority with data-instruction conflation. ∎

## Remark on Scope

Theorem 1 is conditional on (i) ambient authority, (ii) data-instruction conflation, and (iii) Assumptions 1-2. A system achieving perfect channel separation (violating Definition 5) or perfect argument authentication (violating Assumption 2) would escape the theorem. As discussed in Section 2.3 of the main text, both conditions hold for all deployed agent systems as of 2026.

Our architecture resolves Theorem 1 by replacing ambient authority with event-scoped authority: when s contains untrusted data (C1 holds), P(s) is restricted to exclude 𝒯_s, so C2 is relaxed to "the agent has access to sensitive tools only when not processing untrusted data." The enforcement dilemma of Step 4 does not arise because E no longer faces the benign/adversarial ambiguity — the tools are simply not in the permission set.
