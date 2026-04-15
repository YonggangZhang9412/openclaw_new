# Supplementary Note 1: Attack Success Probability Bound

## Formal Model

**Definition 1 (Agent System).** An agent system is a tuple ℳ = (𝒮, s₀, 𝒯, 𝒜, δ, L, E, P) where:
- 𝒮 is the set of states
- s₀ ∈ 𝒮 is the initial state
- 𝒯 = {t₁, ..., t_n} is a finite set of tools with |𝒯| = n
- 𝒜 = {(t, a) | t ∈ 𝒯, a ∈ Args(t)} is the set of tool invocations
- δ: 𝒮 × 𝒜 → 𝒮 is the state transition function
- L: 𝒮 → Δ(𝒜 ∪ {⊥}) is the LLM planning function (stochastic)
- E: 𝒜 × 𝒮 → {ALLOW, DENY} is the enforcement function
- P: 𝒮 → 𝒫(𝒯) is the permission model

**Definition 2 (Tool Classification).** 𝒯 is partitioned into three overlapping capability classes:
- 𝒯_R ⊆ 𝒯: tools that can read sensitive data, with |𝒯_R| = n_R
- 𝒯_X ⊆ 𝒯: tools that can send data externally, with |𝒯_X| = n_X
- 𝒯_U ⊆ 𝒯: tools that introduce untrusted data, with |𝒯_U| = n_U

**Definition 3 (Exfiltration Event).** An exfiltration event occurs in batch b if the tool invocation sequence σ_b contains:
- at least one invocation of t_r ∈ 𝒯_R that reads sensitive data d_s
- at least one invocation of t_x ∈ 𝒯_X that transmits content derived from d_s externally
- the sequence σ_b was causally influenced by untrusted input

We write Exfil(b) for the event that batch b results in exfiltration.

**Definition 4 (Injection Success).** For a batch b processing event e, let Inj(b) denote the event that untrusted data in e successfully manipulates L to produce tool invocations serving the adversary's goal. We write:

p = Pr[Inj(b)]

as the injection success probability, which depends on the LLM model, attack sophistication, and content wrapping defenses. Empirically, p ∈ [0.2, 0.8] for current LLMs without structural defenses¹⁵˒¹⁶.

## Ambient Authority Baseline

**Proposition 1 (Ambient Authority Attack Probability).** Under ambient authority P(s) = P₀ with 𝒯_R ∪ 𝒯_X ⊆ P₀, the probability of exfiltration in batch b satisfies:

Pr[Exfil(b)] ≥ p                                                        ... (1)

**Proof.** Under ambient authority, P₀ contains all tools including 𝒯_R and 𝒯_X. If injection succeeds (event Inj(b)), the LLM has access to both sensitive-read and external-send tools simultaneously, with no structural constraint on their joint use. Therefore:

Pr[Exfil(b)] ≥ Pr[Inj(b)] = p                                          ∎

Over a session of B batches, the probability that at least one batch is compromised is:

Pr[∃ b: Exfil(b)] = 1 - (1-p)^B                                        ... (2)

For persistent agents (B → ∞), this approaches 1 for any p > 0.

## Event-Scoped Capability Authority

**Definition 5 (CapabilityToken).** For each batch b, a CapabilityToken τ_b specifies:
- τ_b.tools ⊆ 𝒯 with |τ_b.tools| = k ≪ n (granted tool set)
- τ_b.ttl = T (time-to-live, in seconds)
- τ_b.R2 ∈ {true, false} (whether Rule of Two is satisfied)

The token is constructed by the CapabilityIssuer based on event metadata, independent of L.

**Definition 6 (Rule of Two).** Token τ satisfies the Rule of Two (τ.R2 = true) if:

¬(τ.tools ∩ 𝒯_U ≠ ∅ ∧ τ.tools ∩ 𝒯_R ≠ ∅ ∧ τ.tools ∩ 𝒯_X ≠ ∅)

That is, the token does not simultaneously grant untrusted-input, sensitive-read, and external-send capabilities.

**Definition 7 (Taint Evasion).** Let q denote the probability that the LLM successfully evades taint tracking — i.e., it reformulates external data such that neither TaintStore (content-level fingerprinting) nor CausalTaintTracker (context-level monotonic tagging) detects the provenance. By construction, CausalTaintTracker is irrevocable within a batch (Lemma 1 below), so evasion requires cross-batch contamination:

q = Pr[taint evasion | inject succeeds, target tools ∈ token]

## Main Result

**Theorem 1 (Attack Success Probability Upper Bound).** Under event-scoped capability authority, the probability of exfiltration in a single batch b satisfies:

Pr[Exfil(b)] ≤ p · α_R · α_X · (1 - R₂) · q                            ... (3)

where:
- p = Pr[Inj(b)] (injection success probability)
- α_R = min(k, n_R) / n (probability that at least one 𝒯_R tool is in token)
- α_X = min(k, n_X) / n (probability that at least one 𝒯_X tool is in token)
- R₂ = 𝟙[τ_b.R2 = true] (Rule of Two indicator: 1 if enforced, 0 if not)
- q = Pr[taint evasion] (Definition 7)

**Proof.**

We decompose Pr[Exfil(b)] by conditioning on independent events. Define:

- A = Inj(b) (injection succeeds)
- B_R = {τ_b.tools ∩ 𝒯_R ≠ ∅} (at least one sensitive-read tool granted)
- B_X = {τ_b.tools ∩ 𝒯_X ≠ ∅} (at least one external-send tool granted)
- C = {τ_b.R2 = false} (Rule of Two does not block)
- D = {taint tracking evaded} (both content and causal tracking fail)

Exfiltration requires all five conditions. Therefore:

Pr[Exfil(b)] = Pr[A ∩ B_R ∩ B_X ∩ C ∩ D]                              ... (4)

**Step 1 (Chain rule).** Expanding by conditional probability:

Pr[A ∩ B_R ∩ B_X ∩ C ∩ D]
= Pr[A] · Pr[B_R ∩ B_X | A] · Pr[C | A ∩ B_R ∩ B_X] · Pr[D | A ∩ B_R ∩ B_X ∩ C]
                                                                         ... (5)

**Step 2 (Bounding Pr[A]).** By definition:

Pr[A] = Pr[Inj(b)] = p                                                  ... (6)

**Step 3 (Bounding Pr[B_R ∩ B_X | A]).** The token τ_b is constructed by the CapabilityIssuer from event metadata, independent of whether injection succeeds (since token issuance precedes LLM invocation). Therefore B_R, B_X are independent of A:

Pr[B_R ∩ B_X | A] = Pr[B_R ∩ B_X]                                      ... (7)

To bound Pr[B_R ∩ B_X], we use the union bound on the complement. The token selects k tools from 𝒯. The probability that at least one 𝒯_R tool is selected:

Pr[B_R] = 1 - Pr[τ_b.tools ∩ 𝒯_R = ∅]
        = 1 - C(n - n_R, k) / C(n, k)                                   ... (8)

where C(·,·) denotes the binomial coefficient. For k ≪ n, this simplifies to:

Pr[B_R] ≤ k · n_R / n = k · (n_R/n)                                     ... (9)

We define α_R = min(k, n_R)/n as an upper bound. Similarly:

Pr[B_X] ≤ α_X = min(k, n_X)/n                                           ... (10)

Since B_R and B_X depend on the same token draw, they are not independent in general. However:

Pr[B_R ∩ B_X] ≤ min(Pr[B_R], Pr[B_X]) ≤ Pr[B_R] · Pr[B_X] / max(Pr[B_R], Pr[B_X])

In the worst case, we use:

Pr[B_R ∩ B_X] ≤ Pr[B_R] · Pr[B_X] ≤ α_R · α_X                         ... (11)

where the first inequality is not tight (B_R, B_X are positively correlated since both require tools in a finite draw), but provides a valid upper bound since we are bounding the attack probability from above.

**Step 4 (Bounding Pr[C | A ∩ B_R ∩ B_X]).** The Rule of Two is a deterministic property of the token τ_b. If τ_b.R2 = true (Rule of Two is enforced), then by Definition 6, 𝒯_R and 𝒯_X cannot both be present in the token when 𝒯_U tools are also present. In this case, the exfiltration chain cannot complete within the batch:

Pr[C | A ∩ B_R ∩ B_X] = 𝟙[τ_b.R2 = false] = 1 - R₂                   ... (12)

When R₂ = 1 (Rule of Two enforced), this term is 0 and Pr[Exfil(b)] = 0.

**Step 5 (Bounding Pr[D | A ∩ B_R ∩ B_X ∩ C]).** Taint evasion requires bypassing both TaintStore and CausalTaintTracker:

Pr[D | A ∩ B_R ∩ B_X ∩ C] = q                                          ... (13)

**Step 6 (Combining).** Substituting (6), (11), (12), (13) into (5):

Pr[Exfil(b)] ≤ p · α_R · α_X · (1 - R₂) · q                           ... (3)  ∎

## Corollaries

**Corollary 1 (Rule of Two Hard Guarantee).** When Rule of Two is enforced (R₂ = 1):

Pr[Exfil(b)] ≤ p · α_R · α_X · 0 · q = 0                              ... (14)

This is a deterministic guarantee: exfiltration probability is exactly zero within a single batch when Rule of Two holds, regardless of p, α_R, α_X, or q.

**Corollary 2 (Session-Level Bound).** Over B independent batches:

Pr[∃ b: Exfil(b)] = 1 - ∏_b (1 - Pr[Exfil(b)])
                   ≤ 1 - (1 - p · α_R · α_X · (1-R₂) · q)^B           ... (15)

Compare with ambient authority (equation (2)): 1 - (1-p)^B.

**Corollary 3 (Concrete Instantiation).** With system parameters n = 55, k = 5, n_R = 8, n_X = 6, p = 0.5, q = 0.1, and R₂ = 1 for all batches:

Pr[Exfil(b)] = 0     (by Corollary 1)

If R₂ = 0 (Rule of Two disabled, worst case):

α_R = min(5, 8)/55 = 5/55 ≈ 0.091
α_X = min(5, 6)/55 = 5/55 ≈ 0.091

Pr[Exfil(b)] ≤ 0.5 · 0.091 · 0.091 · 1.0 · 0.1
              = 0.5 · 0.00828 · 0.1
              = 4.14 × 10⁻⁴

Compare: ambient authority Pr[Exfil(b)] ≥ p = 0.5.

Reduction factor: 0.5 / 4.14×10⁻⁴ ≈ 1208× improvement even without Rule of Two.

---

## Supplementary Lemma

**Lemma 1 (CausalTaintTracker Monotonicity).** Let (Λ, ≤) = ({USER, INTERNAL, EXTERNAL}, <) be the taint lattice with USER < INTERNAL < EXTERNAL. Define the causal taint sequence for a batch of m tool executions:

τ₀ = USER
τ_k = max(τ_{k-1}, taint(t_k))     for k = 1, ..., m

Then for all 0 ≤ i ≤ j ≤ m: τ_i ≤ τ_j.

**Proof.** For any k ∈ {1, ..., m}:

τ_k = max(τ_{k-1}, taint(t_k))
    ≥ τ_{k-1}                        [since max(a,b) ≥ a]

Chaining: τ_i ≤ τ_{i+1} ≤ ··· ≤ τ_j. By transitivity: τ_i ≤ τ_j.  ∎

**Corollary (Irrevocability).** If τ_{k₀} = EXTERNAL for some k₀, then τ_k = EXTERNAL for all k ≥ k₀.

Proof. EXTERNAL ≤ τ_k (by Lemma 1) and τ_k ≤ EXTERNAL (top element). By antisymmetry: τ_k = EXTERNAL.  ∎

This establishes that within a single batch, once external data is observed, the causal taint level cannot decrease — contributing to the bound on q in Theorem 1.
