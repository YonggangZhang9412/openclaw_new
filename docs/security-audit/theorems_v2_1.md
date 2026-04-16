# Supplementary Notes: Formal Security Analysis

## 1. Model and Definitions

**Definition 1 (Agent System).** An agent system is a tuple ℳ = (𝒮, s₀, 𝒯, L, P) where:
- 𝒮 is the set of states (encoding LLM context, session history, and tool results)
- s₀ ∈ 𝒮 is the initial state
- 𝒯 = {t₁, ..., t_n} is a finite set of n tools
- L: 𝒮 → Δ(𝒯 × Args ∪ {⊥}) is the LLM planning function (stochastic), producing a tool invocation or ⊥
- P: 𝒮 → 𝒫(𝒯) is the permission model assigning authorized tools to each state

**Definition 2 (Tool Classification).** The tool set 𝒯 is classified into three (possibly overlapping) subsets:
- 𝒯_R = {t ∈ 𝒯 | reads_sensitive(t)}: tools that can read sensitive data, |𝒯_R| = n_R
- 𝒯_X = {t ∈ 𝒯 | sends_external(t)}: tools that can transmit data externally, |𝒯_X| = n_X
- 𝒯_U = {t ∈ 𝒯 | receives_untrusted(t)}: tools that introduce untrusted data, |𝒯_U| = n_U

**Definition 3 (CapabilityToken).** For each event batch b, a CapabilityToken τ_b specifies:
- τ_b.tools ⊆ 𝒯 with |τ_b.tools| = k_b (the granted tool set for this batch)
- τ_b is constructed from event metadata by a deterministic CapabilityIssuer, independent of L

Let k = max_b k_b denote the maximum token size across all batches.

**Assumption 1 (Worst-Case Token Model).** For the purpose of bounding attack probability, we model the token as selecting k tools uniformly at random from 𝒯 without replacement. This is a worst-case model: the actual CapabilityIssuer selects tools deterministically based on event type, typically granting fewer tools and avoiding dangerous combinations. Any bound derived under the uniform model is therefore a valid upper bound on the actual system.

**Definition 4 (Batch Events).** For batch b, define the following events:
- A = {injection succeeds}: adversarial data in the batch causally influences L's output. Pr[A] = p.
- B_R = {τ_b.tools ∩ 𝒯_R ≠ ∅}: at least one sensitive-read tool is in the token.
- B_X = {τ_b.tools ∩ 𝒯_X ≠ ∅}: at least one external-send tool is in the token.
- C = {¬R₂(τ_b)}: the Rule of Two does not block the attack chain. When R₂ is enforced (R₂(τ_b) = 1), the token construction guarantees that if both B_R and B_X hold, then τ_b.tools ∩ 𝒯_U = ∅ — i.e., no untrusted-input tool is granted alongside sensitive-read and external-send tools. Therefore Pr[C | B_R, B_X, R₂ = 1] = 0.
- D = {taint tracking evaded}: both TaintStore and CausalTaintTracker fail to block the data flow. Pr[D | A, B_R, B_X, C] = q.
- Exfil(b) = A ∩ B_R ∩ B_X ∩ C ∩ D.

**Definition 5 (Taint Lattice).** (Λ, ≤) with Λ = {USER, INTERNAL, EXTERNAL}, USER < INTERNAL < EXTERNAL, ⊥ = USER, ⊤ = EXTERNAL.

**Definition 6 (CausalTaintTracker).** For a batch with m tool executions, the causal taint sequence is:

τ₀ = ⊥
τ_k = max(τ_{k-1}, taint(t_k))    for k = 1, ..., m

---

## 2. Theorem 1: Single-Batch Attack Probability Bound

**Theorem 1 (Multi-Barrier Exfiltration Bound).** In an agent system with n tools where each event batch is authorized a token of at most k tools, the probability that an adversary successfully exfiltrates sensitive data through a single batch is bounded by the product of five independent factors — each corresponding to a distinct architectural barrier that the attacker must simultaneously bypass:

Pr[Exfil(b)] ≤ p · min(kn_R/n, kn_X/n) · (1 - R₂) · q

where p is the prompt injection success rate, kn_R/n and kn_X/n are union-bound upper bounds on the probability that the token contains a sensitive-read or external-send tool respectively (the min reflects that exfiltration requires both), R₂ ∈ {0,1} indicates whether the Rule of Two is enforced (R₂ = 1 zeroes the entire bound), and q is the cross-batch taint evasion probability.

**Proof.**

Pr[Exfil(b)]
= Pr[A ∩ B_R ∩ B_X ∩ C ∩ D]                                                        (Def. 4)
= Pr[A] · Pr[B_R ∩ B_X | A] · Pr[C | A, B_R, B_X] · Pr[D | A, B_R, B_X, C]        (chain rule)

We simplify each factor.

Pr[A] = p                                                                            (Def. 4)

For Pr[B_R ∩ B_X | A]: B_R and B_X are deterministic functions of τ_b (given τ_b, each is 0 or 1). By Def. 3, τ_b = CapabilityIssuer(event_metadata), which does not depend on L. A is an event determined by L's behavior. Since τ_b ⊥ L, any deterministic function of τ_b is independent of any event determined by L. Therefore:

B_R ⊥ A,    B_X ⊥ A
⟹ Pr[B_R ∩ B_X | A] = Pr[B_R ∩ B_X]                                                (i)

For Pr[C | A, B_R, B_X]: C = {¬R₂(τ_b)} is also a deterministic function of τ_b, so C ⊥ A. When R₂ = 1 and B_R ∩ B_X holds, the token construction guarantees τ_b.tools ∩ 𝒯_U = ∅ (Def. 4), so C cannot occur:

R₂ = 1 ∧ B_R ∧ B_X ⟹ τ_b.tools ∩ 𝒯_U = ∅ ⟹ ¬C
⟹ Pr[C | B_R, B_X, R₂ = 1] = 0

When R₂ = 0, no such guarantee exists, so Pr[C | B_R, B_X, R₂ = 0] ≤ 1. Combining:

Pr[C | B_R, B_X] ≤ 1 - R₂                                                           (ii)

Pr[D | A, B_R, B_X, C] = q                                                           (Def. 4)

Substituting into the chain rule:

Pr[Exfil(b)] ≤ p · Pr[B_R ∩ B_X] · (1 - R₂) · q                                    (iii)

We now bound Pr[B_R ∩ B_X]. Under Assumption 1, the token selects k tools uniformly from 𝒯 without replacement.

First, bound Pr[B_R]. Let Z_R = |τ_b.tools ∩ 𝒯_R|. Then B_R = {Z_R ≥ 1}:

Pr[¬B_R]
= Pr[Z_R = 0]
= C(n - n_R, k) / C(n, k)                                                           (hypergeometric)
= ∏_{i=0}^{k-1} (n - n_R - i) / (n - i)                                             (expand)

Each factor satisfies (n - n_R - i) ≤ (n - n_R) since i ≥ 0. So:

∏_{i=0}^{k-1} (n - n_R - i) / (n - i)
≤ ∏_{i=0}^{k-1} (n - n_R) / (n - i)                                                 (numerator: n-n_R-i ≤ n-n_R)
≤ ∏_{i=0}^{k-1} (n - n_R) / (n - k + 1)                                             (denominator: n-i ≥ n-k+1)
= ((n - n_R) / (n - k + 1))^k

For the upper bound on Pr[B_R], we use the union bound directly:

Pr[B_R] = Pr[∃ t ∈ 𝒯_R: t ∈ τ_b.tools]
        ≤ Σ_{t ∈ 𝒯_R} Pr[t ∈ τ_b.tools]                                             (union bound)
        = n_R · (k / n)                                                                (each tool selected with prob k/n)
        = kn_R / n
        =: α_R                                                                        (iv)

By identical argument:

Pr[B_X] ≤ kn_X / n =: α_X                                                            (v)

For Pr[B_R ∩ B_X], we use the trivial bound:

Pr[B_R ∩ B_X] ≤ min(Pr[B_R], Pr[B_X]) ≤ min(α_R, α_X)                                (vi)

Since exfiltration requires BOTH a read tool and a send tool, we can write a tighter parameterized bound. B_R ∩ B_X requires the existence of t_r ∈ 𝒯_R ∩ τ_b.tools AND t_x ∈ 𝒯_X ∩ τ_b.tools (not necessarily distinct). By union bound on pairs:

Pr[B_R ∩ B_X]
= Pr[∃ t_r ∈ 𝒯_R ∩ τ_b.tools, ∃ t_x ∈ 𝒯_X ∩ τ_b.tools]
≤ Σ_{t_r ∈ 𝒯_R} Σ_{t_x ∈ 𝒯_X} Pr[t_r ∈ τ_b.tools ∧ t_x ∈ τ_b.tools]

For t_r ≠ t_x: Pr[t_r ∈ τ ∧ t_x ∈ τ] = k(k-1) / (n(n-1))                           (both drawn without replacement)
For t_r = t_x (when t ∈ 𝒯_R ∩ 𝒯_X): Pr[t ∈ τ] = k/n

Let n_RX = |𝒯_R ∩ 𝒯_X|. Then:

Pr[B_R ∩ B_X]
≤ n_RX · (k/n) + (n_R · n_X - n_RX) · k(k-1)/(n(n-1))                               (vii)
≤ n_RX · (k/n) + n_R · n_X · k(k-1)/(n(n-1))                                         (drop -n_RX term)
≤ n_RX · (k/n) + n_R · n_X · k²/n²                                                    (k(k-1)/(n(n-1)) ≤ k²/n² for n ≥ 2)
= (k/n) · [n_RX + n_R · n_X · k/n]                                                    (factor k/n)

For a clean bound, use (vi) directly: Pr[B_R ∩ B_X] ≤ min(α_R, α_X).

Substituting (iii), (iv) or (v), and (ii):

Pr[Exfil(b)]
≤ p · Pr[B_R ∩ B_X] · (1 - R₂) · q                                                   (from (iii))
≤ p · min(α_R, α_X) · (1 - R₂) · q                                                    (from (vi))
= p · min(kn_R/n, kn_X/n) · (1 - R₂) · q                                              ∎

---


## 3. Theorem 2: Session-Level Compound Bound

**Theorem 2 (Session-Level Security Degradation).** Over a session of B batches with no cross-batch state dependence (see Remark below), the probability that at least one batch is successfully exploited grows with B but remains bounded. Let ε = p · min(α_R, α_X) · (1-R₂) · q be the per-batch bound from Theorem 1. Then:

Pr[∃ b ∈ {1,...,B}: Exfil(b)] ≤ 1 - (1 - ε)^B

Under ambient authority, by contrast, this probability converges to 1 as B → ∞ for any injection rate p > 0, making prolonged agent sessions inherently unsafe.

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

**Remark (Independence Assumption).** The product form ∏(1-Pr[Exfil(b)]) assumes batch independence. In practice, session history shared across batches introduces dependence (the source of cross-batch contamination bounded by q). If independence does not hold, a weaker union bound applies:

Pr[∃ b: Exfil(b)] ≤ Σ_{b=1}^{B} Pr[Exfil(b)] ≤ Bε

which is valid without any independence assumption but is looser for small ε.

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

**Theorem 3 (Rule of Two: Zero-Probability Guarantee).** When the Rule of Two is enforced on a batch token — i.e., the token does not simultaneously grant tools for untrusted input, sensitive data access, and external transmission — the exfiltration probability drops to exactly zero, regardless of the adversary's injection success rate or taint evasion capability:

Pr[Exfil(b)] = 0    when R₂ = 1

**Proof.**

Pr[Exfil(b)]
≤ p · min(α_R, α_X) · (1 - R₂) · q                   (Theorem 1)
= p · min(α_R, α_X) · (1 - 1) · q                    (R₂ = 1)
= p · min(α_R, α_X) · 0 · q
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

**Theorem 4 (Causal Taint Irrevocability).** Once the CausalTaintTracker observes external data at any step within a batch, the taint level is permanently elevated for all subsequent steps — no tool execution can reduce it. This guarantees that within a single batch, the taint evasion probability is exactly zero (q_within = 0), and the only residual evasion risk q in Theorem 1 arises from cross-batch context contamination. Formally, the causal taint sequence is monotonically non-decreasing:

∀ 0 ≤ i ≤ j ≤ m: τ_i ≤ τ_j

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

**Theorem 5 (Bound Integrity: Adversary Cannot Weaken Structural Barriers).** The CapabilityGate function G is deterministic and LLM-independent. This ensures that the structural factors in Theorem 1's bound — the tool-visibility ratios α_R, α_X and the Rule of Two indicator R₂ — are determined solely by event metadata and system configuration, not by the LLM. An adversary who successfully injects the LLM can influence the injection rate p and taint evasion probability q, but cannot manipulate the architectural barriers α_R, α_X, or R₂. Formally:

G: 𝒜 × Token × TaintStore × Counter × ℝ≥₀ → {ALLOW, DENY} is deterministic and LLM-independent.

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

Union bound (Theorem 1):

α_R = kn_R/n = 5 · 8 / 55 = 40/55 ≈ 0.727
α_X = kn_X/n = 5 · 6 / 55 = 30/55 ≈ 0.545
min(α_R, α_X) = 0.545

Pr[Exfil(b)]
≤ p · min(α_R, α_X) · (1 - R₂) · q
= 0.5 · 0.545 · 1.0 · 0.1
= 0.0273

Exact hypergeometric values (for comparison with union bound):

Pr[B_R] = 1 - C(47,5)/C(55,5) = 1 - (47·46·45·44·43)/(55·54·53·52·51) ≈ 0.568
Pr[B_X] = 1 - C(49,5)/C(55,5) = 1 - (49·48·47·46·45)/(55·54·53·52·51) ≈ 0.452

The union bounds α_R = 0.727 and α_X = 0.545 are conservative (true values are 0.568 and 0.452). This confirms the bound is valid but not tight — the actual system is safer than the bound suggests.

Session level (union bound, no independence required):

Pr[∃ b: Exfil(b)]
≤ B · ε                                                (union bound)
= 288 · 0.0273
= 7.85

Since probabilities are capped at 1, this gives Pr ≤ 1 (trivial in worst case without R₂).

Under independence assumption:

Pr[∃ b: Exfil(b)]
≤ 1 - (1 - 0.0273)^{288}
= 1 - 0.9727^{288}
≈ 1 - 0.000275
= 0.9997

**Comparison with ambient authority:**

| Metric | Ambient authority | Event-scoped (R₂=0) | Event-scoped (R₂=1) |
|--------|-------------------|---------------------|----------------------|
| Pr[Exfil(b)] | ≥ 0.5 | ≤ 0.0273 | = 0 |
| Per-batch reduction | — | 18× | ∞ |
| Session (B=288) | ≈ 1.0 | ≤ 0.9997 | = 0 |
