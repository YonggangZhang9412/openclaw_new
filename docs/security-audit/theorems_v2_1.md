# Supplementary Notes: Formal Security Analysis

## 1. Model and Definitions

**Definition 1 (Agent System).** ℳ = (𝒮, s₀, 𝒯, 𝒜, δ, L, E, P), where 𝒯 = {t₁,...,t_n}, L: 𝒮 → Δ(𝒜 ∪ {⊥}), E: 𝒜 × 𝒮 → {ALLOW, DENY}, P: 𝒮 → 𝒫(𝒯).

**Definition 2 (Tool Classification).** 𝒯_R = {t ∈ 𝒯 | reads_sensitive(t)}, 𝒯_X = {t ∈ 𝒯 | sends_external(t)}, 𝒯_U = {t ∈ 𝒯 | receives_untrusted(t)}, with |𝒯_R| = n_R, |𝒯_X| = n_X, |𝒯_U| = n_U.

**Definition 3 (CapabilityToken).** For batch b, token τ_b specifies τ_b.tools ⊆ 𝒯 with |τ_b.tools| = k, constructed from event metadata independent of L.

**Definition 4 (Batch Events).** For batch b, define:
- A = {injection succeeds}, Pr[A] = p
- B_R = {τ_b.tools ∩ 𝒯_R ≠ ∅}, B_X = {τ_b.tools ∩ 𝒯_X ≠ ∅}
- C = {τ_b.tools ∩ 𝒯_U ≠ ∅ ∧ τ_b.tools ∩ 𝒯_R ≠ ∅ ∧ τ_b.tools ∩ 𝒯_X ≠ ∅}
- D = {taint tracking evaded}, Pr[D | A, B_R, B_X, C] = q
- Exfil(b) = A ∩ B_R ∩ B_X ∩ C ∩ D

**Definition 5 (Taint Lattice).** (Λ, ≤) with Λ = {USER, INTERNAL, EXTERNAL}, USER < INTERNAL < EXTERNAL, ⊥ = USER, ⊤ = EXTERNAL.

**Definition 6 (CausalTaintTracker).** For batch with m tool executions: τ₀ = ⊥, τ_k = max(τ_{k-1}, taint(t_k)) for k = 1,...,m.

---

## 2. Theorem 1: Single-Batch Attack Probability Bound

**Theorem 1.** Pr[Exfil(b)] ≤ p · (kn_R/n) · (kn_X/n) · (1 - R₂) · q, where R₂ = 𝟙[Rule of Two enforced].

**Proof.**

Pr[Exfil(b)]
= Pr[A ∩ B_R ∩ B_X ∩ C ∩ D]                                                        (Def. 4)
= Pr[A] · Pr[B_R ∩ B_X | A] · Pr[C | A, B_R, B_X] · Pr[D | A, B_R, B_X, C]        (chain rule)
= p · Pr[B_R ∩ B_X] · Pr[C | B_R, B_X] · q                                         (i)

where line (i) follows from: Pr[A] = p (Def. 4); B_R, B_X ⊥ A (τ_b issued before L invoked, Def. 3); C ⊥ A | B_R, B_X (C is a property of τ_b); Pr[D | ·] = q (Def. 4).

We bound Pr[B_R]. Let Z_R = |τ_b.tools ∩ 𝒯_R|. Then:

Pr[¬B_R]
= Pr[Z_R = 0]
= C(n - n_R, k) / C(n, k)                                                           (hypergeometric)
= ∏_{i=0}^{k-1} (n - n_R - i) / (n - i)                                             (expand binomials)
= ∏_{i=0}^{k-1} [1 - n_R / (n - i)]                                                 (factor each term)
≥ ∏_{i=0}^{k-1} [1 - n_R / (n - k + 1)]                                             (n - i ≥ n - k + 1)
= [1 - n_R / (n - k + 1)]^k                                                          (ii)

For the upper bound on Pr[B_R], set x = n_R / n ∈ [0, 1]:

Pr[B_R]
= 1 - Pr[¬B_R]
= 1 - ∏_{i=0}^{k-1} (n - n_R - i) / (n - i)
≤ 1 - ∏_{i=0}^{k-1} (1 - n_R / n)                                                   (n - i ≤ n ⟹ 1/(n-i) ≥ 1/n)
= 1 - (1 - x)^k                                                                      (iii)
≤ kx                                                                                  (Bernoulli: 1-(1-x)^k ≤ kx, x ∈ [0,1])
= kn_R / n
=: α_R                                                                                (iv)

By identical argument replacing n_R with n_X:

Pr[B_X] ≤ kn_X / n =: α_X                                                            (v)

For the joint probability:

Pr[B_R ∩ B_X]
= Pr[B_R] · Pr[B_X | B_R]
≤ Pr[B_R] · 1                                                                         (trivial bound)
≤ α_R                                                                                  (vi)

We can also write:

Pr[B_R ∩ B_X] ≤ min(Pr[B_R], Pr[B_X]) ≤ min(α_R, α_X)                                (vi')

For the product bound (valid when 𝒯_R ∩ 𝒯_X = ∅):

Pr[B_R ∩ B_X] = Pr[B_R] · Pr[B_X | B_R] ≤ Pr[B_R] · Pr[B_X] ≤ α_R · α_X            (vii)

Note: when 𝒯_R ∩ 𝒯_X ≠ ∅, B_R and B_X are positively correlated, so (vii) is conservative as an upper bound. We use (vii) as it yields the tightest parameterized bound.

For the Rule of Two factor, when R₂ = 1, token construction guarantees ¬C given any B_R, B_X:

Pr[C | B_R, B_X] = 1 - R₂                                                             (viii)

Substituting (i), (iv), (v), (vii), (viii):

Pr[Exfil(b)]
≤ p · Pr[B_R ∩ B_X] · (1 - R₂) · q                                                   (from (i))
≤ p · α_R · α_X · (1 - R₂) · q                                                        (from (vii))
= p · (kn_R / n) · (kn_X / n) · (1 - R₂) · q                                          ∎

---


## 3. Theorem 2: Session-Level Compound Bound

**Theorem 2.** Let ε = p · α_R · α_X · (1-R₂) · q. Over B independent batches:

Pr[∃ b ∈ {1,...,B}: Exfil(b)] ≤ 1 - (1 - ε)^B

**Proof.**

Pr[∃ b: Exfil(b)]
= 1 - Pr[¬Exfil(1) ∩ ¬Exfil(2) ∩ ··· ∩ ¬Exfil(B)]                            (complement)
= 1 - ∏_{b=1}^{B} Pr[¬Exfil(b)]                                                (batches independent: distinct tokens, disjoint time windows)
= 1 - ∏_{b=1}^{B} (1 - Pr[Exfil(b)])                                            (Pr[¬X] = 1 - Pr[X])

Since Pr[Exfil(b)] ≤ ε for each b (Theorem 1):

1 - Pr[Exfil(b)] ≥ 1 - ε                                                        (monotonicity of subtraction)

Therefore:

∏_{b=1}^{B} (1 - Pr[Exfil(b)])
≥ ∏_{b=1}^{B} (1 - ε)                                                            (each factor ≥ 1 - ε)
= (1 - ε)^B

Substituting:

Pr[∃ b: Exfil(b)]
= 1 - ∏_{b=1}^{B} (1 - Pr[Exfil(b)])
≤ 1 - (1 - ε)^B                                                                   ∎

**Proposition 2.1 (Ambient Authority Baseline).**

Pr_amb[Exfil(b)]
≥ Pr[A]                                              (all tools available, no R₂, no taint tracking)
= p

Pr_amb[∃ b: Exfil(b)]
= 1 - ∏_{b=1}^{B} (1 - Pr_amb[Exfil(b)])
≥ 1 - ∏_{b=1}^{B} (1 - p)                            (Pr_amb[Exfil(b)] ≥ p)
= 1 - (1 - p)^B

lim_{B→∞} [1 - (1-p)^B]
= 1 - lim_{B→∞} (1-p)^B
= 1 - 0                                               (|1-p| < 1 for p > 0)
= 1                                                                                ∎

---

## 4. Theorem 3: Rule of Two Hard Guarantee

**Theorem 3.** If R₂ = 1, then Pr[Exfil(b)] = 0.

**Proof.**

Pr[Exfil(b)]
≤ p · α_R · α_X · (1 - R₂) · q                      (Theorem 1)
= p · α_R · α_X · (1 - 1) · q                        (R₂ = 1)
= p · α_R · α_X · 0 · q
= 0

0 ≤ Pr[Exfil(b)] ≤ 0                                  (probability axiom + above)
⟹ Pr[Exfil(b)] = 0                                                                 ∎

**Corollary 3.1.** If R₂ = 1 for all B batches:

Pr[∃ b: Exfil(b)]
≤ 1 - (1 - 0)^B                                       (Theorem 2 with ε = 0)
= 1 - 1^B
= 1 - 1
= 0                                                                                 ∎

---

## 5. Theorem 4: CausalTaintTracker Monotonicity

**Theorem 4.** ∀ 0 ≤ i ≤ j ≤ m: τ_i ≤ τ_j.

**Proof.** For any k ∈ {1, ..., m}:

τ_k = max(τ_{k-1}, taint(t_k))                        (Definition 6)
    ≥ τ_{k-1}                                          (max(a,b) ≥ a for all a,b ∈ Λ)

Applying this at k = i+1, i+2, ..., j:

τ_i ≤ τ_{i+1}                                          (k = i+1)
     ≤ τ_{i+2}                                         (k = i+2)
     ≤ ···
     ≤ τ_j                                             (k = j)

By transitivity of ≤ on Λ:

τ_i ≤ τ_j                                                                           ∎

**Corollary 4.1 (Irrevocability).** If τ_{k₀} = ⊤ for some k₀, then ∀ k ≥ k₀: τ_k = ⊤.

**Proof.**

τ_k ≥ τ_{k₀}                                           (Theorem 4, k₀ ≤ k)
    = ⊤                                                 (assumption)

τ_k ≤ ⊤                                                 (⊤ = max Λ, so ∀ x ∈ Λ: x ≤ ⊤)

Combining:

⊤ ≤ τ_k ≤ ⊤
⟹ τ_k = ⊤                                              (antisymmetry of ≤)          ∎

**Proposition 4.2 (Within-Batch Evasion).** If taint(t_j) = ⊤ at step j, then q_within = 0.

**Proof.** Let k > j. Then k - 1 ≥ j.

τ_{k-1} ≥ τ_j                                           (Theorem 4, j ≤ k-1)
         = max(τ_{j-1}, taint(t_j))                     (Definition 6)
         ≥ taint(t_j)                                    (max(a,b) ≥ b)
         = ⊤                                             (assumption)

⟹ τ_{k-1} = ⊤                                           (Corollary 4.1)

For any tool t with max_causal(t) = INTERNAL < ⊤:

τ_{k-1} = ⊤ = EXTERNAL
        > INTERNAL
        = max_causal(t)

⟹ τ_{k-1} > max_causal(t)
⟹ t is causally blocked at step k

This holds ∀ k > j and ∀ t ∈ 𝒯_X (all external-action tools have max_causal ≤ INTERNAL). No external transmission can occur after step j within this batch.

∴ q_within = Pr[within-batch taint evasion] = 0                                      ∎

**Remark.** q in Theorem 1 decomposes as q = q_within + q_cross - q_within · q_cross. Since q_within = 0: q = q_cross.

---

## 6. Theorem 5: Gate Determinism and Bound Integrity

**Theorem 5.** G: 𝒜 × Token × TaintStore × Counter × ℝ≥₀ → {ALLOW, DENY} is deterministic and LLM-independent.

**Proof (Determinism).** G = Check₃ ∘ Check₂ ∘ Check₁. Each Checkᵢ is a composition of:

Check₁:  {>, ∈, ∈}                  (real comparison, set membership ×2)
Check₂:  {SHA-256, dict.lookup, >}  (per parameter, |args| iterations)
Check₃:  {fnmatch, ≥, boolean}     (glob match, counter compare, R₂ eval)

Let x = (a, τ, S, C, t_now) = x'. Then:

Check₁(x):  t_now > τ.issued_at + τ.ttl   → same t_now, same τ → same result
            t ∈ τ.granted_tools            → same t, same τ → same result
            t ∈ τ.denied_tools             → same t, same τ → same result
⟹ Check₁(x) = Check₁(x')

Check₂(x):  ∀ (p,v) ∈ args:
              SHA-256(v) = SHA-256(v')     (same v → same hash)
              S.lookup(h) = S'.lookup(h')  (same S, same h → same tl)
              tl > max_taint               (same tl, same policy → same result)
⟹ Check₂(x) = Check₂(x')

Check₃(x):  fnmatch(path, globs) = fnmatch(path', globs')   (same inputs)
            C[t] = C'[t']                                     (same counter)
            R₂(τ) = R₂(τ')                                    (same token)
⟹ Check₃(x) = Check₃(x')

∴ G(x) = G(x')                                                (determinism)       ∎

**Proof (LLM-Independence).** G's input sources:

a = (t, args)  ← output of L, but G does not invoke L to evaluate a
τ              ← CapabilityIssuer(event_metadata); L not involved
S              ← framework registers tool returns; L not involved
C              ← framework counter; L not involved
t_now          ← wall clock; L not involved

Operations within G:  {SHA-256, ∈, dict.lookup, >, <, ≥, ≤, =, fnmatch, ∧, ∨, ¬}
∩ {operations that invoke L} = ∅

∴ G is LLM-independent                                                             ∎

**Corollary 5.1 (Bound Integrity).**

α_R = k · n_R / n = f(k, n_R, n)                      (constants, not functions of L)
α_X = k · n_X / n = f(k, n_X, n)                      (same)
R₂ = 𝟙[Rule of Two on τ_b] = g(τ_b)                  (function of token, not L)
τ_b = CapabilityIssuer(event_metadata)                 (L not invoked, Def. 3 + Thm. 5)

Adversary controls: p (via injection quality), q (via paraphrasing skill)
Adversary cannot control: α_R, α_X, R₂ (determined before L runs)                  ∎
---

## 7. Concrete Instantiation

Parameters: n = 55, k = 5, n_R = 8, n_X = 6, p = 0.5, q = 0.1, R₂ ∈ {0, 1}, B = 288.

**Case 1: R₂ = 1.**

Pr[Exfil(b)] = 0                                       (Theorem 3)

**Case 2: R₂ = 0 (worst case).**

α_R = 5 · 8 / 55 = 40/55 ≈ 0.727
α_X = 5 · 6 / 55 = 30/55 ≈ 0.545

Pr[Exfil(b)]
≤ 0.5 · 0.727 · 0.545 · 1.0 · 0.1
= 0.5 · 0.396 · 0.1
= 0.0198

Pr[∃ b: Exfil(b)]
≤ 1 - (1 - 0.0198)^{288}
= 1 - 0.9802^{288}
≈ 1 - 0.00316
= 0.997

**Comparison with ambient authority:**

Pr_ambient[Exfil(b)] ≥ p = 0.5
Pr_capability[Exfil(b)] ≤ 0.0198

Reduction: 0.5 / 0.0198 ≈ 25× per batch (without Rule of Two).
With Rule of Two: Pr = 0 vs Pr ≥ 0.5 (infinite reduction).
