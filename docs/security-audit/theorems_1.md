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
- L: 𝒮 → Δ(𝒜 ∪ {⊥}) is the LLM planning function, mapping the current state to a probability distribution over tool invocations or ⊥ (no action)
- E: 𝒜 × 𝒮 → {ALLOW, DENY} is the safety enforcement function

**Definition 4 (Permission Model).** A permission model is a function P: 𝒮 → 𝒫(𝒯) assigning a set of permitted tools to each state. Under **ambient authority**: P(s) = P₀ for all reachable s ∈ 𝒮.

**Definition 5 (Data-Instruction Conflation).** ℳ exhibits data-instruction conflation if:

∃ s, s' ∈ 𝒮, ∃ a ∈ 𝒜: s \ data(s) = s' \ data(s') ∧ Pr[L(s) = a] ≠ Pr[L(s') = a]

where data(s) ⊂ s denotes the untrusted external data present in state s.

**Definition 6 (Safety).** An agent system ℳ with enforcement function E is safe if no adversary-influenced tool invocation of a sensitive tool is permitted:

Safe(ℳ) ≡ ∀ s ∈ Reach(ℳ), ∀ (t, a) ∈ 𝒜:
  [t ∈ 𝒯_s ∧ InfluencedByAdversary(s, t, a)] ⟹ E((t, a), s) = DENY

where InfluencedByAdversary(s, t, a) holds when ∃ d ∈ data(s) with source(d) ∉ Trusted such that removing d from s would change L's probability of outputting (t, a):

InfluencedByAdversary(s, t, a) ≡ ∃ d ∈ data(s): source(d) ∉ Trusted ∧ Pr[L(s) = (t,a)] ≠ Pr[L(s \ {d}) = (t,a)]

**Definition 7 (Utility).** An agent system ℳ with enforcement function E has utility if the operator can effectively use sensitive tools for legitimate purposes:

Useful(ℳ) ≡ ∃ s_b ∈ Reach(ℳ), ∃ t_s ∈ 𝒯_s, ∃ a_b ∈ Vals(Params(t_s)):
  E((t_s, a_b), s_b) = ALLOW

**Definition 8 (Conditions C1, C2, C3).**

**(C1)** ∃ s ∈ Reach(ℳ): data(s) ≠ ∅ ∧ ∀ d ∈ data(s): source(d) ∉ Trusted ∧ approved(d) = false

**(C2)** Useful(ℳ) ∧ [∀ s ∈ Reach(ℳ): P(s) ∩ 𝒯_s ≠ ∅]

**(C3)** ∀ a ∈ 𝒜, ∀ s₁, s₂ ∈ 𝒮: P(s₁) = P(s₂) ⟹ E(a, s₁) = E(a, s₂)

## Assumptions

**Assumption 1 (LLM Manipulability).** For any t ∈ P₀, there exists adversarial d* and a* ∈ Vals(Params(t)) such that incorporating d* into a reachable state s yields s* with:

Pr[L(s*) = (t, a*)] > 0

Empirical support: prompt injection benchmarks report 20-80% success rates across major LLM families¹⁵˒¹⁶. Assumption 1 requires only Pr > 0, strictly weaker than observed rates.

**Assumption 2 (Argument Mimicry).** For any t ∈ 𝒯_s, for any legitimate a_b ∈ Vals(Params(t)) satisfying E((t, a_b), s_b) = ALLOW at some benign state s_b, there exists adversarial a* ∈ Vals(Params(t)) such that:

∀ s ∈ 𝒮: E((t, a_b), s) = E((t, a*), s)          ... (*)

## Theorem

**Theorem 1 (Agent Authority Impossibility).** Let ℳ be an agent system with ambient authority P(s) = P₀ exhibiting data-instruction conflation. Under Assumptions 1 and 2, Safe(ℳ) and C1 ∧ C2 ∧ C3 cannot simultaneously hold.

## Proof

Assume for contradiction that Safe(ℳ) holds and C1 ∧ C2 ∧ C3 hold simultaneously.

**Step 1 (Existence of sensitive tool access).**

By C2 (Definition 8), Useful(ℳ) holds. By Definition 7:

∃ s_b ∈ Reach(ℳ), ∃ t_s ∈ 𝒯_s, ∃ a_b ∈ Vals(Params(t_s)): E((t_s, a_b), s_b) = ALLOW     ... (1)

Fix such s_b, t_s, a_b. Note that t_s ∈ 𝒯_s and t_s ∈ P₀ (by C2, P(s_b) ∩ 𝒯_s ≠ ∅, and P(s_b) = P₀ under ambient authority).

**Step 2 (Existence of adversarial invocation).**

By C1 (Definition 8), ∃ s_u ∈ Reach(ℳ) with data(s_u) ≠ ∅ and untrusted data present. Let s₀ be a reachable state before adversarial data is introduced. Since t_s ∈ P₀, by Assumption 1:

∃ d*, a*₁: Pr[L(s*) = (t_s, a*₁)] > 0     ... (2a)

where s* = s₀ ∪ {d*}. Moreover, since d* was specifically chosen to induce this output:

Pr[L(s₀) = (t_s, a*₁)] ≠ Pr[L(s*) = (t_s, a*₁)]     ... (2b)

(If the probabilities were equal, d* would have no causal influence and could be replaced by a d* that does.)

**Step 3 (Combining Assumptions 1 and 2).**

By Assumption 2 applied to t_s and a_b (from Step 1), there exists a*₂ in the same E-equivalence class as a_b:

∀ s ∈ 𝒮: E((t_s, a_b), s) = E((t_s, a*₂), s)     ... (3a)

We now need a* that satisfies both (2a) and (3a) — i.e., both producible by L under injection and indistinguishable from a_b by E. By Assumption 2, the E-equivalence class [a_b] = {a ∈ Vals(Params(t_s)) | ∀ s: E((t_s, a), s) = E((t_s, a_b), s)} contains all syntactically valid, type-correct argument vectors for t_s. By Assumption 1, the adversary can induce L to produce any such vector. Therefore, the adversary can choose d* such that a*₁ ∈ [a_b], giving us a single a* = a*₁ = a*₂ satisfying both:

Pr[L(s*) = (t_s, a*)] > 0     ... (2)     [from (2a)]
∀ s: E((t_s, a_b), s) = E((t_s, a*), s)     ... (3)     [from (3a)]

Instantiating (3) at s = s_b:

E((t_s, a*), s_b) = E((t_s, a_b), s_b) = ALLOW     (by (1) and (3))     ... (4)

Instantiating (3) at s = s*:

E((t_s, a*), s*) = E((t_s, a_b), s*)     ... (5)

**Step 4 (Applying C3).**

By C3 (Definition 8), E depends only on the invocation and the permission set. Under ambient authority, P(s_b) = P(s*) = P₀. Therefore:

E((t_s, a_b), s_b) = E((t_s, a_b), s*)     (by C3, since P(s_b) = P(s*) = P₀)     ... (6)

Combining (5) and (6):

E((t_s, a*), s*) = E((t_s, a_b), s*) = E((t_s, a_b), s_b) = ALLOW     (by (6) and (1))     ... (7)

**Step 5 (Contradiction with Safety).**

From (2): Pr[L(s*) = (t_s, a*)] > 0, so (t_s, a*) can be invoked at s*.
From (7): E((t_s, a*), s*) = ALLOW, so E permits this invocation.

We verify InfluencedByAdversary(s*, t_s, a*) (Definition 6). We need:
∃ d ∈ data(s*): source(d) ∉ Trusted ∧ Pr[L(s*) = (t_s, a*)] ≠ Pr[L(s* \ {d}) = (t_s, a*)]

Take d = d*. By construction, source(d*) ∉ Trusted. By (2b), Pr[L(s₀) = (t_s, a*)] ≠ Pr[L(s*) = (t_s, a*)], and s₀ = s* \ {d*}. Therefore the condition holds.

Hence InfluencedByAdversary(s*, t_s, a*) = true.     ... (8)

We now have:
- t_s ∈ 𝒯_s     (from Step 1)
- InfluencedByAdversary(s*, t_s, a*) = true     (equation (8))
- E((t_s, a*), s*) = ALLOW     (equation (7))

By Definition 6 (Safety), Safe(ℳ) requires:
[t_s ∈ 𝒯_s ∧ InfluencedByAdversary(s*, t_s, a*)] ⟹ E((t_s, a*), s*) = DENY

The antecedent is satisfied (by Step 1 and (8)), so the consequent must hold: E((t_s, a*), s*) = DENY. But (7) gives E((t_s, a*), s*) = ALLOW.

DENY = ALLOW is a contradiction. ∎

## Remark on Scope

Theorem 1 is conditional on (i) ambient authority, (ii) data-instruction conflation, and (iii) Assumptions 1-2. Our architecture resolves the impossibility by replacing ambient authority with event-scoped authority: when s contains untrusted data, P(s) is restricted to exclude 𝒯_s, so Step 1 cannot produce a state where both t_s ∈ P(s) and adversarial data coexist. Equation (6) no longer holds because P(s_b) ≠ P(s*), breaking the chain at Step 4.
