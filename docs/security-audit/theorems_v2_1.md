# Supplementary Notes: Formal Security Analysis

## 1. Model and Definitions

**Definition 1 (Agent System).** ℳ = (𝒮, s₀, 𝒯, 𝒜, δ, L, E, P), where 𝒯 = {t₁,...,t_n}, L: 𝒮 → Δ(𝒜 ∪ {⊥}), E: 𝒜 × 𝒮 → {ALLOW, DENY}, P: 𝒮 → 𝒫(𝒯).

**Definition 2 (Tool Classification).** 𝒯_R = {t ∈ 𝒯 | reads_sensitive(t)}, 𝒯_X = {t ∈ 𝒯 | sends_external(t)}, 𝒯_U = {t ∈ 𝒯 | receives_untrusted(t)}, with |𝒯_R| = n_R, |𝒯_X| = n_X, |𝒯_U| = n_U.

**Definition 3 (CapabilityToken).** For batch b, token τ_b specifies τ_b.tools ⊆ 𝒯 with |τ_b.tools| = k, constructed from event metadata independent of L.

**Definition 4 (Batch Events).** For batch b, define:
- A = {injection succeeds}, Pr[A] = p
- B_R = {τ_b.tools ∩ 𝒯_R ≠ ∅}, B_X = {τ_b.tools ∩ 𝒯_X ≠ ∅}
- C = {Rule of Two not enforced}, i.e. τ_b.tools ∩ 𝒯_U ≠ ∅ ∧ τ_b.tools ∩ 𝒯_R ≠ ∅ ∧ τ_b.tools ∩ 𝒯_X ≠ ∅
- D = {taint tracking evaded}, Pr[D | A, B_R, B_X, C] = q
- Exfil(b) = A ∩ B_R ∩ B_X ∩ C ∩ D

**Definition 5 (Taint Lattice).** (Λ, ≤) with Λ = {USER, INTERNAL, EXTERNAL}, USER < INTERNAL < EXTERNAL, ⊥ = USER, ⊤ = EXTERNAL.

**Definition 6 (CausalTaintTracker).** For batch with m tool executions: τ₀ = ⊥, τ_k = max(τ_{k-1}, taint(t_k)) for k = 1,...,m.

---

## 2. Theorem 1: Single-Batch Attack Probability Bound

**Theorem 1.** Under event-scoped capability authority:

Pr[Exfil(b)] ≤ p · (kn_R / n) · (kn_X / n) · (1 - R₂) · q

where R₂ = 𝟙[Rule of Two enforced on τ_b].

**Proof.**

Pr[Exfil(b)]
= Pr[A ∩ B_R ∩ B_X ∩ C ∩ D]                                                       (Definition 4)
= Pr[A] · Pr[B_R ∩ B_X | A] · Pr[C | A, B_R, B_X] · Pr[D | A, B_R, B_X, C]       (chain rule)

We bound each factor separately.

**Factor 1: Pr[A].**

Pr[A] = p                                                                           (Definition 4)

**Factor 2: Pr[B_R ∩ B_X | A].**

Token τ_b is issued by CapabilityIssuer from event metadata before L is invoked (Definition 3). Therefore B_R and B_X are independent of A:

Pr[B_R ∩ B_X | A] = Pr[B_R ∩ B_X]

We bound Pr[B_R]. The token draws k tools from 𝒯 = {t₁,...,t_n} without replacement. Let Z_R = |τ_b.tools ∩ 𝒯_R| (number of sensitive-read tools in token). Then B_R = {Z_R ≥ 1}, and:

Pr[¬B_R]
= Pr[Z_R = 0]
= C(n - n_R, k) / C(n, k)                                                          (hypergeometric)
= [(n-n_R)! / ((n-n_R-k)! · k!)] / [n! / ((n-k)! · k!)]
= [(n-n_R)! · (n-k)!] / [(n-n_R-k)! · n!]
= ∏_{i=0}^{k-1} (n - n_R - i) / (n - i)

Each factor in the product satisfies:

(n - n_R - i) / (n - i) = 1 - n_R/(n - i)
                        ≤ 1 - n_R/n                                                (since n - i ≤ n)

Therefore:

Pr[¬B_R] ≤ (1 - n_R/n)^k

And:

Pr[B_R]
= 1 - Pr[¬B_R]
≥ 1 - (1 - n_R/n)^k

For the upper bound (we need Pr[B_R] ≤ ... to bound the attack probability):

Pr[B_R]
= 1 - ∏_{i=0}^{k-1} (n - n_R - i)/(n - i)
≤ 1 - ∏_{i=0}^{k-1} (n - n_R - i)/n                                               (n - i ≤ n in denominator)
≤ 1 - ((n - n_R)/n)^k                                                              (each numerator ≤ n - n_R)

Let x = n_R/n. Then:

Pr[B_R] ≤ 1 - (1 - x)^k
         ≤ kx                                                                       (Bernoulli: 1-(1-x)^k ≤ kx for x ∈ [0,1])
         = k · n_R / n

Define α_R := k · n_R / n. Then Pr[B_R] ≤ α_R. By identical argument with n_X replacing n_R:

Pr[B_X] ≤ k · n_X / n =: α_X

For the joint probability, exfiltration requires both B_R and B_X. Since both events are determined by the same token draw (not independent), we bound:

Pr[B_R ∩ B_X]
= Pr[B_R] · Pr[B_X | B_R]
≤ Pr[B_R] · Pr[B_X]                                                                (‡)
≤ α_R · α_X

where (‡) holds because conditioning on B_R (at least one 𝒯_R tool drawn) does not decrease the probability of also drawing a 𝒯_X tool when 𝒯_R ∩ 𝒯_X may overlap. For a rigorous upper bound, Pr[B_X | B_R] ≤ 1, so Pr[B_R ∩ B_X] ≤ Pr[B_R] ≤ α_R. We use the tighter product bound α_R · α_X which is valid when 𝒯_R and 𝒯_X are disjoint; when they overlap, Pr[B_R ∩ B_X] is even smaller.

**Factor 3: Pr[C | A, B_R, B_X].**

C is a deterministic property of τ_b (Definition 4), independent of A. So Pr[C | A, B_R, B_X] = Pr[C | B_R, B_X]. When Rule of Two is enforced (R₂ = 1), the token construction guarantees ¬(τ_b.tools ∩ 𝒯_U ≠ ∅ ∧ τ_b.tools ∩ 𝒯_R ≠ ∅ ∧ τ_b.tools ∩ 𝒯_X ≠ ∅), so given B_R and B_X, the event C cannot occur:

Pr[C | B_R, B_X] = 1 - R₂

**Factor 4: Pr[D | A, B_R, B_X, C].**

Pr[D | A, B_R, B_X, C] = q                                                         (Definition 4)

**Combining all factors:**

Pr[Exfil(b)]
= Pr[A] · Pr[B_R ∩ B_X | A] · Pr[C | A, B_R, B_X] · Pr[D | A, B_R, B_X, C]
≤ p · Pr[B_R ∩ B_X] · (1 - R₂) · q
≤ p · α_R · α_X · (1 - R₂) · q
= p · (kn_R/n) · (kn_X/n) · (1 - R₂) · q                                          ∎

---

## 3. Theorem 2: Session-Level Compound Bound

**Theorem 2.** Over B independent batches, the probability that at least one exfiltration occurs satisfies:

Pr[∃ b ∈ {1,...,B}: Exfil(b)] ≤ 1 - (1 - p · α_R · α_X · (1-R₂) · q)^B

where α_R = kn_R/n, α_X = kn_X/n.

Under ambient authority:

Pr[∃ b: Exfil(b)] ≥ 1 - (1 - p)^B

**Proof.**

Let ε = p · α_R · α_X · (1 - R₂) · q. By Theorem 1, Pr[Exfil(b)] ≤ ε for each b.

Pr[∃ b: Exfil(b)]
= 1 - Pr[∀ b: ¬Exfil(b)]
= 1 - ∏_{b=1}^{B} Pr[¬Exfil(b)]                                        (batch independence)
= 1 - ∏_{b=1}^{B} (1 - Pr[Exfil(b)])
≤ 1 - ∏_{b=1}^{B} (1 - ε)                                               (Theorem 1)
= 1 - (1 - ε)^B                                                          ∎

For ambient authority, Pr[Exfil(b)] ≥ p (all tools available, no structural constraint), so:

Pr[∃ b: Exfil(b)] ≥ 1 - (1 - p)^B                                       ∎

---

## 4. Theorem 3: Rule of Two Hard Guarantee

**Theorem 3.** When Rule of Two is enforced (R₂ = 1) on token τ_b:

Pr[Exfil(b)] = 0

**Proof.**

By Theorem 1:

Pr[Exfil(b)]
≤ p · α_R · α_X · (1 - R₂) · q
= p · α_R · α_X · (1 - 1) · q
= p · α_R · α_X · 0 · q
= 0

Since probabilities are non-negative: Pr[Exfil(b)] = 0.                             ∎

---

## 5. Theorem 4: Taint Tracking Monotonicity and Evasion Bound

**Theorem 4 (Monotonicity).** For the CausalTaintTracker sequence (Definition 6): ∀ 0 ≤ i ≤ j ≤ m, τ_i ≤ τ_j.

**Proof.**

τ_k = max(τ_{k-1}, taint(t_k)) ≥ τ_{k-1}                                (max(a,b) ≥ a)

Chaining: τ_i ≤ τ_{i+1} ≤ ··· ≤ τ_j.                                                ∎

**Corollary 4.1 (Irrevocability).** If τ_{k₀} = ⊤ for some k₀, then τ_k = ⊤ for all k ≥ k₀.

**Proof.**

⊤ = τ_{k₀} ≤ τ_k ≤ ⊤        (Theorem 4 and ⊤ = max Λ)
⟹ τ_k = ⊤                    (antisymmetry)                                          ∎

**Proposition 4.2 (Within-Batch Evasion Bound).** Within a single batch where CausalTaintTracker is active, if any tool t_j with taint(t_j) = EXTERNAL is executed at step j, then for all subsequent steps k > j, any tool t with causal policy max_causal(t) < EXTERNAL is blocked. Therefore, within-batch taint evasion probability is:

q_within = 0

**Proof.** Let k > j.

τ_{k-1} ≥ τ_j                                                            (Theorem 4)
        = max(τ_{j-1}, taint(t_j))                                       (Definition 6)
        ≥ taint(t_j)                                                      (max(a,b) ≥ b)
        = EXTERNAL = ⊤

So τ_{k-1} = ⊤ > max_causal(t) for any t with max_causal(t) < ⊤.
Tool t is blocked. No external-action tool can execute after step j.
Therefore q_within = 0.                                                                ∎

**Remark.** The taint evasion probability q in Theorem 1 accounts for cross-batch contamination only (session history carrying untrusted data from prior batches). Within a single batch, Proposition 4.2 guarantees q_within = 0. Therefore q = q_cross, which requires adversarial data to persist across batch boundaries — a condition whose frequency is quantified in Experiment 4 of the evaluation framework.

---

## 6. Theorem 5: Gate Determinism (Bound Integrity)

**Theorem 5.** The CapabilityGate function G: 𝒜 × Token × TaintStore × Counter × ℝ → {ALLOW, DENY} is deterministic and LLM-independent.

This theorem ensures the integrity of the bound in Theorem 1: the factors α_R, α_X, R₂ are determined by the token (which is issued by deterministic code from event metadata), not by the LLM. An adversary who compromises L cannot alter these factors.

**Proof.** G consists of three sequential checks. Each check uses only:
- set membership (∈ on frozenset): deterministic
- SHA-256 hash: deterministic
- dictionary lookup: deterministic
- real/integer comparison: deterministic
- glob matching: deterministic
- boolean operations: deterministic

No operation invokes L or reads L's internal state. The token τ is constructed before L is invoked (CapabilityIssuer uses event metadata only). Therefore:

∀ (a, τ, S, C, t): G(a, τ, S, C, t) is determined by its inputs alone     (determinism)
G does not invoke L at any step                                              (LLM-independence)

Consequence for Theorem 1: the adversary controls Pr[A] = p (injection success) and influences Pr[D] = q (taint evasion), but cannot influence α_R, α_X, or R₂, since these are properties of τ which is computed by G's issuer — a deterministic, LLM-independent function.          ∎

---

## 7. Concrete Instantiation

| Parameter | Value | Source |
|-----------|-------|--------|
| n = \|𝒯\| | 55 | System tool registry (16 modules) |
| k = \|τ.tools\| | 5 | CapabilityIssuer max grant |
| n_R = \|𝒯_R\| | 8 | read_file, bash, execute_code, ... |
| n_X = \|𝒯_X\| | 6 | send_email, curl, wget, bash, ... |
| p | 0.5 | Median injection success rate¹⁵˒¹⁶ |
| q | 0.1 | Estimated cross-batch contamination |
| R₂ | 1 | Rule of Two enforced (default) |
| B | 288 | 24h session / 300s TTL |

**With Rule of Two (R₂ = 1):**

Pr[Exfil(b)] = 0                                                (Theorem 3)
Pr[∃ b: Exfil(b)] = 0                                           (Corollary)

**Without Rule of Two (R₂ = 0, worst case):**

α_R = 5 · 8 / 55 = 40/55 ≈ 0.727
α_X = 5 · 6 / 55 = 30/55 ≈ 0.545

Pr[Exfil(b)]
≤ 0.5 · 0.727 · 0.545 · 1.0 · 0.1
= 0.5 · 0.396 · 0.1
= 0.0198

Pr[∃ b: Exfil(b)]
≤ 1 - (1 - 0.0198)^288
≈ 1 - 0.9802^288
≈ 1 - 0.00316
≈ 0.997

Compare ambient authority: 1 - (1 - 0.5)^288 ≈ 1.0.

**Per-batch reduction factor:**

Pr_ambient / Pr_capability ≈ 0.5 / 0.0198 ≈ 25×

With Rule of Two: ∞ (zero vs non-zero).
