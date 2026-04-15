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

**Assumption 1 (LLM Manipulability).** For any t ∈ P₀ and any target argument vector a ∈ Vals(Params(t)), there exists adversarial data d* such that incorporating d* into a reachable state s₀ yields s* = s₀ ∪ {d*} with:

(A1a)  Pr[L(s*) = (t, a)] > 0
(A1b)  Pr[L(s₀) = (t, a)] ≠ Pr[L(s*) = (t, a)]

That is, the adversary can craft d* to induce L to produce any specific tool invocation with non-zero probability, and d*'s presence causally affects L's output distribution. Empirical support: prompt injection benchmarks report 20-80% success rates across major LLM families for diverse target actions¹⁵˒¹⁶. (A1a) requires only Pr > 0, strictly weaker than observed rates. (A1b) follows from the adversarial construction: if d* had no causal effect, it would not serve as an injection vector.

**Assumption 2 (Argument Mimicry).** For any t ∈ 𝒯_s, for any legitimate a_b ∈ Vals(Params(t)) satisfying E((t, a_b), s_b) = ALLOW at some benign state s_b, there exists adversarial a* ∈ Vals(Params(t)) such that:

∀ s ∈ 𝒮: E((t, a_b), s) = E((t, a*), s)          ... (*)

**Definition 9 (Safety Violation Probability).** For a state s ∈ 𝒮, define:

V(s) = Pr_{(t,a) ~ L(s)} [t ∈ 𝒯_s ∧ InfluencedByAdversary(s, t, a) ∧ E((t, a), s) = ALLOW]

V(s) is the probability that L proposes a sensitive, adversary-influenced tool invocation that E permits. Note that Safe(ℳ) (Definition 6) holds if and only if V(s) = 0 for all s ∈ Reach(ℳ).

## Theorem

**Theorem 1 (Agent Authority Impossibility).** Let ℳ be an agent system with ambient authority P(s) = P₀ exhibiting data-instruction conflation. Under Assumptions 1 and 2, if C1 ∧ C2 ∧ C3 hold, then ∃ s* ∈ Reach(ℳ): V(s*) > 0, and therefore ¬Safe(ℳ).

## Proof

Assume for contradiction that Safe(ℳ) holds and C1 ∧ C2 ∧ C3 hold simultaneously.

**Step 1.** By C2 and Definition 7 (Useful):

∃ s_b ∈ Reach(ℳ), ∃ t_s ∈ 𝒯_s, ∃ a_b ∈ Vals(Params(t_s)):
    E((t_s, a_b), s_b) = ALLOW                                          ... (1)

Fix such s_b, t_s, a_b. By C2 and ambient authority:

t_s ∈ P(s_b) ∩ 𝒯_s = P₀ ∩ 𝒯_s                                         ... (1')

**Step 2.** By Assumption 2 applied to (t_s, a_b) from (1), ∃ a* ∈ Vals(Params(t_s)):

∀ s ∈ 𝒮: E((t_s, a*), s) = E((t_s, a_b), s)                            ... (2)

**Step 3.** By C1, ∃ s₀ ∈ Reach(ℳ) with data(s₀) ≠ ∅. Since t_s ∈ P₀ (by (1')), apply Assumption 1 with target (t_s, a*) from (2):

∃ d*: Pr[L(s₀ ∪ {d*}) = (t_s, a*)] > 0                                 (A1a)
      Pr[L(s₀) = (t_s, a*)] ≠ Pr[L(s₀ ∪ {d*}) = (t_s, a*)]            (A1b)

Let s* = s₀ ∪ {d*}. Then:

Pr[L(s*) = (t_s, a*)] > 0                                               ... (3)
Pr[L(s₀) = (t_s, a*)] ≠ Pr[L(s*) = (t_s, a*)]                          ... (4)

**Step 4.** We derive E's decision at s*. By C3 and ambient authority:

P(s_b) = P(s*) = P₀                                                     ... (5)

Applying C3 to (t_s, a_b):

E((t_s, a_b), s_b) = E((t_s, a_b), s*)     [by (5) and C3]             ... (6)

Applying (2) at s = s*:

E((t_s, a*), s*) = E((t_s, a_b), s*)                                    ... (7)

Chaining (1), (6), (7):

E((t_s, a*), s*) = E((t_s, a_b), s*)    [by (7)]
                 = E((t_s, a_b), s_b)    [by (6)]
                 = ALLOW                  [by (1)]                       ... (8)

**Step 5.** We verify InfluencedByAdversary(s*, t_s, a*) (Definition 6). Take d = d*:

source(d*) ∉ Trusted                                  [by construction]
s* \ {d*} = s₀                                        [by definition of s*]
Pr[L(s*) = (t_s, a*)] ≠ Pr[L(s₀) = (t_s, a*)]       [by (4)]

All conditions of Definition 6 are satisfied:

InfluencedByAdversary(s*, t_s, a*) = true                                ... (9)

**Step 6.** We compute V(s*) (Definition 9). Since (t_s, a*) satisfies all three conditions in V's definition:

V(s*) = Pr_{(t,a) ~ L(s*)} [t ∈ 𝒯_s ∧ Influenced ∧ E = ALLOW]
      ≥ Pr[L(s*) = (t_s, a*)]
         · 𝟙[t_s ∈ 𝒯_s] · 𝟙[InfluencedByAdversary(s*, t_s, a*)] · 𝟙[E((t_s, a*), s*) = ALLOW]
      = Pr[L(s*) = (t_s, a*)] · 1 · 1 · 1                              [by (1'), (9), (8)]
      = Pr[L(s*) = (t_s, a*)]
      > 0                                                                [by (3)]    ... (10)

Therefore V(s*) > 0, so ¬Safe(ℳ) (by Definition 9). ∎

## Remark on Scope

Theorem 1 is conditional on (i) ambient authority, (ii) data-instruction conflation, and (iii) Assumptions 1-2. Our architecture resolves the impossibility by replacing ambient authority with event-scoped authority: when s contains untrusted data, P(s) is restricted to exclude 𝒯_s. Equation (5) no longer holds (P(s_b) ≠ P(s*)), breaking the equality chain at Step 4. As a result, E((t_s, a*), s*) is no longer constrained to equal ALLOW, and V(s*) = 0 becomes achievable.
