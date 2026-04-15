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

**Step 1.** By C2 and Definition 7 (Useful):

∃ s_b ∈ Reach(ℳ), ∃ t_s ∈ 𝒯_s, ∃ a_b ∈ Vals(Params(t_s)):
    E((t_s, a_b), s_b) = ALLOW                                          ... (1)

Fix such s_b, t_s, a_b. By C2 and ambient authority:

t_s ∈ P(s_b) ∩ 𝒯_s = P₀ ∩ 𝒯_s                                         ... (1')

**Step 2.** By C1, ∃ s₀ ∈ Reach(ℳ) with data(s₀) ≠ ∅. Since t_s ∈ P₀ (by (1')), Assumption 1 gives:

∃ d*, a*₁ ∈ Vals(Params(t_s)):
    Pr[L(s₀ ∪ {d*}) = (t_s, a*₁)] > 0                                  ... (2a)

Let s* = s₀ ∪ {d*}. By construction of d* (chosen to induce the output):

Pr[L(s₀) = (t_s, a*₁)] ≠ Pr[L(s*) = (t_s, a*₁)]                       ... (2b)

**Step 3.** By Assumption 2 applied to (t_s, a_b) from (1), define the equivalence class:

[a_b]_E = {a ∈ Vals(Params(t_s)) | ∀ s ∈ 𝒮: E((t_s, a), s) = E((t_s, a_b), s)}

By Assumption 2, [a_b]_E contains all syntactically valid arguments for t_s. By Assumption 1, the adversary can induce L to produce any (t_s, a) with a ∈ Vals(Params(t_s)). Take a* ∈ [a_b]_E with Pr[L(s*) = (t_s, a*)] > 0. Then:

∀ s ∈ 𝒮: E((t_s, a*), s) = E((t_s, a_b), s)                            ... (3)
Pr[L(s*) = (t_s, a*)] > 0                                               ... (4)
Pr[L(s₀) = (t_s, a*)] ≠ Pr[L(s*) = (t_s, a*)]                          ... (5)

where (3) holds by a* ∈ [a_b]_E, (4) by Assumption 1, and (5) by the same argument as (2b).

**Step 4.** We now derive E's decision at s*. By C3 (Definition 8) and ambient authority:

P(s_b) = P(s*) = P₀                                                     ... (6)

Applying C3 to (t_s, a_b):

E((t_s, a_b), s_b) = E((t_s, a_b), s*)     [by (6) and C3]             ... (7)

Applying (3) at s = s*:

E((t_s, a*), s*) = E((t_s, a_b), s*)                                    ... (8)

Chaining (1), (7), (8):

E((t_s, a*), s*) = E((t_s, a_b), s*)    [by (8)]
                 = E((t_s, a_b), s_b)    [by (7)]
                 = ALLOW                  [by (1)]                       ... (9)

**Step 5.** We verify InfluencedByAdversary(s*, t_s, a*) (Definition 6). Need:

∃ d ∈ data(s*): source(d) ∉ Trusted ∧ Pr[L(s*) = (t_s, a*)] ≠ Pr[L(s* \ {d}) = (t_s, a*)]

Take d = d*:
- source(d*) ∉ Trusted   [by construction in Step 2]
- s* \ {d*} = s₀          [by definition of s*]
- Pr[L(s*) = (t_s, a*)] ≠ Pr[L(s₀) = (t_s, a*)]   [by (5)]

Therefore:

InfluencedByAdversary(s*, t_s, a*) = true                                ... (10)

**Step 6 (Contradiction).** Collecting results:

t_s ∈ 𝒯_s                                      [by (1')]
InfluencedByAdversary(s*, t_s, a*) = true       [by (10)]
E((t_s, a*), s*) = ALLOW                        [by (9)]

By Definition 6 (Safety):

[t_s ∈ 𝒯_s ∧ InfluencedByAdversary(s*, t_s, a*)] ⟹ E((t_s, a*), s*) = DENY

The antecedent holds by (1') and (10). Therefore E((t_s, a*), s*) = DENY. But (9) gives E((t_s, a*), s*) = ALLOW.

DENY ≠ ALLOW. Contradiction. ∎

## Remark on Scope

Theorem 1 is conditional on (i) ambient authority, (ii) data-instruction conflation, and (iii) Assumptions 1-2. Our architecture resolves the impossibility by replacing ambient authority with event-scoped authority: when s contains untrusted data, P(s) is restricted to exclude 𝒯_s, so Step 1 cannot produce a state where both t_s ∈ P(s) and adversarial data coexist. Equation (6) no longer holds because P(s_b) ≠ P(s*), breaking the chain at Step 4.
