# Supplementary Notes: Formal Security Analysis

## 1. Model and Definitions

**Definition 1 (Agent System).** An agent system is a tuple $\mathcal{M} = (\mathcal{S}, s_0, \mathcal{T}, L, P)$ where:
- $\mathcal{S}$ is the set of states (encoding LLM context, session history, and tool results)
- $s_0 \in \mathcal{S}$ is the initial state
- $\mathcal{T} = \{t_1, \ldots, t_n\}$ is a finite set of n tools
- $L: \mathcal{S} \to \Delta(\mathcal{T} \times \text{Args} \cup \{\bot\})$ is the LLM planning function (stochastic), producing a tool invocation or $\bot$
- $P: \mathcal{S} \to \mathcal{P}(\mathcal{T})$ is the permission model assigning authorized tools to each state

**Definition 2 (Tool Classification).** The tool set $\mathcal{T}$ is classified into three (possibly overlapping) subsets:
- $\mathcal{T}_R$ = {t $\in$ $\mathcal{T}$ | reads_sensitive(t)}: tools that can read sensitive data, |$\mathcal{T}_R$| = n_R
- $\mathcal{T}_X$ = {t $\in$ $\mathcal{T}$ | sends_external(t)}: tools that can transmit data externally, |$\mathcal{T}_X$| = n_X
- $\mathcal{T}_U$ = {t $\in$ $\mathcal{T}$ | receives_untrusted(t)}: tools that introduce untrusted data, |$\mathcal{T}_U$| = n_U

**Definition 3 (CapabilityToken).** For each event batch b, a CapabilityToken $\tau_b$ specifies:
- $\tau_b$.tools $\subseteq$ $\mathcal{T}$ with |$\tau_b$.tools| = k_b (the granted tool set for this batch)
- $\tau_b$ is constructed from event metadata by a deterministic CapabilityIssuer, independent of L

Let k = max_b k_b denote the maximum token size across all batches.

**Assumption 1 (Worst-Case Token Model).** For the purpose of bounding attack probability, we model the token as selecting k tools uniformly at random from $\mathcal{T}$ without replacement. This is a worst-case model: the actual CapabilityIssuer selects tools deterministically based on event type, typically granting fewer tools and avoiding dangerous combinations. Any bound derived under the uniform model is therefore a valid upper bound on the actual system.

**Definition 4 (Batch Events).** For batch b, define the following events:
- A = {injection succeeds}: adversarial data in the batch causally influences L's output. Pr[A] = p.
- B_R = {$\tau_b$.tools $\cap$ $\mathcal{T}_R$ ≠ $\emptyset$}: at least one sensitive-read tool is in the token.
- B_X = {$\tau_b$.tools $\cap$ $\mathcal{T}_X$ ≠ $\emptyset$}: at least one external-send tool is in the token.
- C = {$\neg$$R_2$($\tau_b$)}: the Rule of Two does not block the attack chain. When $R_2$ is enforced ($R_2$($\tau_b$) = 1), the token construction guarantees that if both B_R and B_X hold, then $\tau_b$.tools $\cap$ $\mathcal{T}_U$ = $\emptyset$ — i.e., no untrusted-input tool is granted alongside sensitive-read and external-send tools. Therefore Pr[C | B_R, B_X, $R_2$ = 1] = 0.
- D = {taint tracking evaded}: both TaintStore and CausalTaintTracker fail to block the data flow. Pr[D | A, B_R, B_X, C] = q.
- Exfil(b) = A $\cap$ B_R $\cap$ B_X $\cap$ C $\cap$ D.

**Definition 5 (Taint Lattice).** ($\Lambda$, $\leq$) with $\Lambda$ = {USER, INTERNAL, EXTERNAL}, USER < INTERNAL < EXTERNAL, $\bot$ = USER, $\top$ = EXTERNAL.

**Definition 6 (CausalTaintTracker).** For a batch with m tool executions, the causal taint sequence is:

$\tau_0$ = $\bot$
$\tau_k$ = max($\tau_{k-1}$, taint(t_k))    for k = 1, ..., m

---

## 2. Theorem 1: Single-Batch Attack Probability Bound

**Theorem 1 (Multi-Barrier Exfiltration Bound).** In an agent system with n tools where each event batch is authorized a token of at most k tools, the probability that an adversary successfully exfiltrates sensitive data through a single batch is bounded by the product of five independent factors — each corresponding to a distinct architectural barrier that the attacker must simultaneously bypass:

Pr[Exfil(b)] $\leq$ p · min(kn_R/n, kn_X/n) · (1 - $R_2$) · q

where p is the prompt injection success rate, kn_R/n and kn_X/n are union-bound upper bounds on the probability that the token contains a sensitive-read or external-send tool respectively (the min reflects that exfiltration requires both), $R_2$ $\in$ {0,1} indicates whether the Rule of Two is enforced ($R_2$ = 1 zeroes the entire bound), and q is the cross-batch taint evasion probability.

**Proof.**

Pr[Exfil(b)]
= Pr[A $\cap$ B_R $\cap$ B_X $\cap$ C $\cap$ D]                                                        (Def. 4)
= Pr[A] · Pr[B_R $\cap$ B_X | A] · Pr[C | A, B_R, B_X] · Pr[D | A, B_R, B_X, C]        (chain rule)

We simplify each factor.

Pr[A] = p                                                                            (Def. 4)

For Pr[B_R $\cap$ B_X | A]: B_R and B_X are deterministic functions of $\tau_b$ (given $\tau_b$, each is 0 or 1). By Def. 3, $\tau_b$ = CapabilityIssuer(event_metadata), which does not depend on L. A is an event determined by L's behavior. Since $\tau_b \perp L$, any deterministic function of $\tau_b$ is independent of any event determined by L. Therefore:

B_R \perp A,    B_X \perp A
$\Longrightarrow$ Pr[B_R $\cap$ B_X | A] = Pr[B_R $\cap$ B_X]                                                (i)

For Pr[C | A, B_R, B_X]: C = {$\neg$$R_2$($\tau_b$)} is also a deterministic function of $\tau_b$, so C \perp A. When $R_2$ = 1 and B_R $\cap$ B_X holds, the token construction guarantees $\tau_b$.tools $\cap$ $\mathcal{T}_U$ = $\emptyset$ (Def. 4), so C cannot occur:

$R_2$ = 1 ∧ B_R ∧ B_X $\Longrightarrow$ $\tau_b$.tools $\cap$ $\mathcal{T}_U$ = $\emptyset$ $\Longrightarrow$ $\neg$C
$\Longrightarrow$ Pr[C | B_R, B_X, $R_2$ = 1] = 0

When $R_2$ = 0, no such guarantee exists, so Pr[C | B_R, B_X, $R_2$ = 0] $\leq$ 1. Combining:

Pr[C | B_R, B_X] $\leq$ 1 - $R_2$                                                           (ii)

Pr[D | A, B_R, B_X, C] = q                                                           (Def. 4)

Substituting into the chain rule:

Pr[Exfil(b)] $\leq$ p · Pr[B_R $\cap$ B_X] · (1 - $R_2$) · q                                    (iii)

We now bound Pr[B_R $\cap$ B_X]. Under Assumption 1, the token selects k tools uniformly from $\mathcal{T}$ without replacement.

First, bound Pr[B_R]. Let Z_R = |$\tau_b$.tools $\cap$ $\mathcal{T}_R$|. Then B_R = {Z_R $\geq$ 1}:

Pr[$\neg$B_R]
= Pr[Z_R = 0]
= C(n - n_R, k) / C(n, k)                                                           (hypergeometric)
= $\prod$_{i=0}^{k-1} (n - n_R - i) / (n - i)                                             (expand)

Each factor satisfies (n - n_R - i) $\leq$ (n - n_R) since i $\geq$ 0. So:

$\prod$_{i=0}^{k-1} (n - n_R - i) / (n - i)
$\leq$ $\prod$_{i=0}^{k-1} (n - n_R) / (n - i)                                                 (numerator: n-n_R-i $\leq$ n-n_R)
$\leq$ $\prod$_{i=0}^{k-1} (n - n_R) / (n - k + 1)                                             (denominator: n-i $\geq$ n-k+1)
= ((n - n_R) / (n - k + 1))^k

For the upper bound on Pr[B_R], we use the union bound directly:

Pr[B_R] = Pr[$\exists$ t $\in$ $\mathcal{T}_R$: t $\in$ $\tau_b$.tools]
        $\leq$ $\Sigma$_{t $\in$ $\mathcal{T}_R$} Pr[t $\in$ $\tau_b$.tools]                                             (union bound)
        = n_R · (k / n)                                                                (each tool selected with prob k/n)
        = kn_R / n
        =: $\alpha_R$                                                                        (iv)

By identical argument:

Pr[B_X] $\leq$ kn_X / n =: $\alpha_X$                                                            (v)

For Pr[B_R $\cap$ B_X], we use the trivial bound:

Pr[B_R $\cap$ B_X] $\leq$ min(Pr[B_R], Pr[B_X]) $\leq$ min($\alpha_R$, $\alpha_X$)                                (vi)

Since exfiltration requires BOTH a read tool and a send tool, we can write a tighter parameterized bound. B_R $\cap$ B_X requires the existence of t_r $\in$ $\mathcal{T}_R$ $\cap$ $\tau_b$.tools AND t_x $\in$ $\mathcal{T}_X$ $\cap$ $\tau_b$.tools (not necessarily distinct). By union bound on pairs:

Pr[B_R $\cap$ B_X]
= Pr[$\exists$ t_r $\in$ $\mathcal{T}_R$ $\cap$ $\tau_b$.tools, $\exists$ t_x $\in$ $\mathcal{T}_X$ $\cap$ $\tau_b$.tools]
$\leq$ $\Sigma$_{t_r $\in$ $\mathcal{T}_R$} $\Sigma$_{t_x $\in$ $\mathcal{T}_X$} Pr[t_r $\in$ $\tau_b$.tools ∧ t_x $\in$ $\tau_b$.tools]

For t_r ≠ t_x: Pr[t_r $\in$ τ ∧ t_x $\in$ τ] = k(k-1) / (n(n-1))                           (both drawn without replacement)
For t_r = t_x (when t $\in$ $\mathcal{T}_R$ $\cap$ $\mathcal{T}_X$): Pr[t $\in$ τ] = k/n

Let n_RX = |$\mathcal{T}_R$ $\cap$ $\mathcal{T}_X$|. Then:

Pr[B_R $\cap$ B_X]
$\leq$ n_RX · (k/n) + (n_R · n_X - n_RX) · k(k-1)/(n(n-1))                               (vii)
$\leq$ n_RX · (k/n) + n_R · n_X · k(k-1)/(n(n-1))                                         (drop -n_RX term)
$\leq$ n_RX · (k/n) + n_R · n_X · k²/n²                                                    (k(k-1)/(n(n-1)) $\leq$ k²/n² for n $\geq$ 2)
= (k/n) · [n_RX + n_R · n_X · k/n]                                                    (factor k/n)

For a clean bound, use (vi) directly: Pr[B_R $\cap$ B_X] $\leq$ min($\alpha_R$, $\alpha_X$).

Substituting (iii), (iv) or (v), and (ii):

Pr[Exfil(b)]
$\leq$ p · Pr[B_R $\cap$ B_X] · (1 - $R_2$) · q                                                   (from (iii))
$\leq$ p · min($\alpha_R$, $\alpha_X$) · (1 - $R_2$) · q                                                    (from (vi))
= p · min(kn_R/n, kn_X/n) · (1 - $R_2$) · q                                              ∎

---


## 3. Theorem 2: Session-Level Compound Bound

**Theorem 2 (Session-Level Security Degradation).** Over a session of B batches with no cross-batch state dependence (see Remark below), the probability that at least one batch is successfully exploited grows with B but remains bounded. Let $\varepsilon$ = p · min($\alpha_R$, $\alpha_X$) · (1-$R_2$) · q be the per-batch bound from Theorem 1. Then:

Pr[$\exists$ b $\in$ {1,...,B}: Exfil(b)] $\leq$ 1 - (1 - $\varepsilon$)^B

Under ambient authority, by contrast, this probability converges to 1 as B → ∞ for any injection rate p > 0, making prolonged agent sessions inherently unsafe.

**Proof.**

Pr[$\exists$ b: Exfil(b)]
= 1 - Pr[$\neg$Exfil(1) $\cap$ $\neg$Exfil(2) $\cap$ ··· $\cap$ $\neg$Exfil(B)]                            (complement)
= 1 - $\prod$_{b=1}^{B} Pr[$\neg$Exfil(b)]                                                (batches independent: distinct tokens, disjoint time windows)
= 1 - $\prod$_{b=1}^{B} (1 - Pr[Exfil(b)])                                            (Pr[$\neg$X] = 1 - Pr[X])

Since Pr[Exfil(b)] $\leq$ $\varepsilon$ for each b (Theorem 1):

1 - Pr[Exfil(b)] $\geq$ 1 - $\varepsilon$                                                        (monotonicity of subtraction)

Therefore:

$\prod$_{b=1}^{B} (1 - Pr[Exfil(b)])
$\geq$ $\prod$_{b=1}^{B} (1 - $\varepsilon$)                                                            (each factor $\geq$ 1 - $\varepsilon$)
= (1 - $\varepsilon$)^B

Substituting:

Pr[$\exists$ b: Exfil(b)]
= 1 - $\prod$_{b=1}^{B} (1 - Pr[Exfil(b)])
$\leq$ 1 - (1 - $\varepsilon$)^B                                                                   ∎

**Remark (Independence Assumption).** The product form $\prod$(1-Pr[Exfil(b)]) assumes batch independence. In practice, session history shared across batches introduces dependence (the source of cross-batch contamination bounded by q). If independence does not hold, a weaker union bound applies:

Pr[$\exists$ b: Exfil(b)] $\leq$ $\Sigma$_{b=1}^{B} Pr[Exfil(b)] $\leq$ B$\varepsilon$

which is valid without any independence assumption but is looser for small $\varepsilon$.

**Proposition 2.1 (Ambient Authority Baseline).**

Pr_amb[Exfil(b)]
$\geq$ Pr[A]                                              (all tools available, no $R_2$, no taint tracking)
= p

Pr_amb[$\exists$ b: Exfil(b)]
= 1 - $\prod$_{b=1}^{B} (1 - Pr_amb[Exfil(b)])
$\geq$ 1 - $\prod$_{b=1}^{B} (1 - p)                            (Pr_amb[Exfil(b)] $\geq$ p)
= 1 - (1 - p)^B

lim_{B→∞} [1 - (1-p)^B]
= 1 - lim_{B→∞} (1-p)^B
= 1 - 0                                               (|1-p| < 1 for p > 0)
= 1                                                                                ∎

---

## 4. Theorem 3: Rule of Two Hard Guarantee

**Theorem 3 (Rule of Two: Zero-Probability Guarantee).** When the Rule of Two is enforced on a batch token — i.e., the token does not simultaneously grant tools for untrusted input, sensitive data access, and external transmission — the exfiltration probability drops to exactly zero, regardless of the adversary's injection success rate or taint evasion capability:

Pr[Exfil(b)] = 0    when $R_2$ = 1

**Proof.**

Pr[Exfil(b)]
$\leq$ p · min($\alpha_R$, $\alpha_X$) · (1 - $R_2$) · q                   (Theorem 1)
= p · min($\alpha_R$, $\alpha_X$) · (1 - 1) · q                    ($R_2$ = 1)
= p · min($\alpha_R$, $\alpha_X$) · 0 · q
= 0

0 $\leq$ Pr[Exfil(b)] $\leq$ 0                                  (probability axiom + above)
$\Longrightarrow$ Pr[Exfil(b)] = 0                                                                 ∎

**Corollary 3.1.** If $R_2$ = 1 for all B batches:

Pr[$\exists$ b: Exfil(b)]
$\leq$ 1 - (1 - 0)^B                                       (Theorem 2 with $\varepsilon$ = 0)
= 1 - 1^B
= 1 - 1
= 0                                                                                 ∎

---

## 5. Theorem 4: CausalTaintTracker Monotonicity

**Theorem 4 (Causal Taint Irrevocability).** Once the CausalTaintTracker observes external data at any step within a batch, the taint level is permanently elevated for all subsequent steps — no tool execution can reduce it. This guarantees that within a single batch, the taint evasion probability is exactly zero (q_within = 0), and the only residual evasion risk q in Theorem 1 arises from cross-batch context contamination. Formally, the causal taint sequence is monotonically non-decreasing:

$\forall$ 0 $\leq$ i $\leq$ j $\leq$ m: $\tau_i$ $\leq$ $\tau_j$

**Proof.** For any k $\in$ {1, ..., m}:

$\tau_k$ = max($\tau_{k-1}$, taint(t_k))                        (Definition 6)
    $\geq$ $\tau_{k-1}$                                          (max(a,b) $\geq$ a for all a,b $\in$ $\Lambda$)

Applying this at k = i+1, i+2, ..., j:

$\tau_i$ $\leq$ τ_{i+1}                                          (k = i+1)
     $\leq$ τ_{i+2}                                         (k = i+2)
     $\leq$ ···
     $\leq$ $\tau_j$                                             (k = j)

By transitivity of $\leq$ on $\Lambda$:

$\tau_i$ $\leq$ $\tau_j$                                                                           ∎

**Corollary 4.1 (Irrevocability).** If $\tau_{k_0}$ = $\top$ for some k₀, then $\forall$ k $\geq$ k₀: $\tau_k$ = $\top$.

**Proof.**

$\tau_k$ $\geq$ $\tau_{k_0}$                                           (Theorem 4, k₀ $\leq$ k)
    = $\top$                                                 (assumption)

$\tau_k$ $\leq$ $\top$                                                 ($\top$ = max $\Lambda$, so $\forall$ x $\in$ $\Lambda$: x $\leq$ $\top$)

Combining:

$\top$ $\leq$ $\tau_k$ $\leq$ $\top$
$\Longrightarrow$ $\tau_k$ = $\top$                                              (antisymmetry of $\leq$)          ∎

**Proposition 4.2 (Within-Batch Evasion).** If taint(t_j) = $\top$ at step j, then q_within = 0.

**Proof.** Let k > j. Then k - 1 $\geq$ j.

$\tau_{k-1}$ $\geq$ $\tau_j$                                           (Theorem 4, j $\leq$ k-1)
         = max(τ_{j-1}, taint(t_j))                     (Definition 6)
         $\geq$ taint(t_j)                                    (max(a,b) $\geq$ b)
         = $\top$                                             (assumption)

$\Longrightarrow$ $\tau_{k-1}$ = $\top$                                           (Corollary 4.1)

For any tool t with max_causal(t) = INTERNAL < $\top$:

$\tau_{k-1}$ = $\top$ = EXTERNAL
        > INTERNAL
        = max_causal(t)

$\Longrightarrow$ $\tau_{k-1}$ > max_causal(t)
$\Longrightarrow$ t is causally blocked at step k

This holds $\forall$ k > j and $\forall$ t $\in$ $\mathcal{T}_X$ (all external-action tools have max_causal $\leq$ INTERNAL). No external transmission can occur after step j within this batch.

$\therefore$ q_within = Pr[within-batch taint evasion] = 0                                      ∎

**Remark.** q in Theorem 1 decomposes as q = q_within + q_cross - q_within · q_cross. Since q_within = 0: q = q_cross.

---

## 6. Theorem 5: Gate Determinism and Bound Integrity

**Theorem 5 (Bound Integrity: Adversary Cannot Weaken Structural Barriers).** The CapabilityGate function G is deterministic and LLM-independent. This ensures that the structural factors in Theorem 1's bound — the tool-visibility ratios $\alpha_R$, $\alpha_X$ and the Rule of Two indicator $R_2$ — are determined solely by event metadata and system configuration, not by the LLM. An adversary who successfully injects the LLM can influence the injection rate p and taint evasion probability q, but cannot manipulate the architectural barriers $\alpha_R$, $\alpha_X$, or $R_2$. Formally:

G: $\mathcal{A}$ × Token × TaintStore × Counter × ℝ$\geq$₀ → {ALLOW, DENY} is deterministic and LLM-independent.

**Proof (Determinism).** G = Check₃ ∘ Check₂ ∘ Check₁. Each Checkᵢ is a composition of:

Check₁:  {>, $\in$, $\in$}                  (real comparison, set membership ×2)
Check₂:  {SHA-256, dict.lookup, >}  (per parameter, |args| iterations)
Check₃:  {fnmatch, $\geq$, boolean}     (glob match, counter compare, $R_2$ eval)

Let x = (a, τ, S, C, t_now) = x'. Then:

Check₁(x):  t_now > τ.issued_at + τ.ttl   → same t_now, same τ → same result
            t $\in$ τ.granted_tools            → same t, same τ → same result
            t $\in$ τ.denied_tools             → same t, same τ → same result
$\Longrightarrow$ Check₁(x) = Check₁(x')

Check₂(x):  $\forall$ (p,v) $\in$ args:
              SHA-256(v) = SHA-256(v')     (same v → same hash)
              S.lookup(h) = S'.lookup(h')  (same S, same h → same tl)
              tl > max_taint               (same tl, same policy → same result)
$\Longrightarrow$ Check₂(x) = Check₂(x')

Check₃(x):  fnmatch(path, globs) = fnmatch(path', globs')   (same inputs)
            C[t] = C'[t']                                     (same counter)
            $R_2$(τ) = $R_2$(τ')                                    (same token)
$\Longrightarrow$ Check₃(x) = Check₃(x')

$\therefore$ G(x) = G(x')                                                (determinism)       ∎

**Proof (LLM-Independence).** G's input sources:

a = (t, args)  ← output of L, but G does not invoke L to evaluate a
τ              ← CapabilityIssuer(event_metadata); L not involved
S              ← framework registers tool returns; L not involved
C              ← framework counter; L not involved
t_now          ← wall clock; L not involved

Operations within G:  {SHA-256, $\in$, dict.lookup, >, <, $\geq$, $\leq$, =, fnmatch, ∧, ∨, $\neg$}
$\cap$ {operations that invoke L} = $\emptyset$

$\therefore$ G is LLM-independent                                                             ∎

**Corollary 5.1 (Bound Integrity).**

$\alpha_R$ = k · n_R / n = f(k, n_R, n)                      (constants, not functions of L)
$\alpha_X$ = k · n_X / n = f(k, n_X, n)                      (same)
$R_2$ = 𝟙[Rule of Two on $\tau_b$] = g($\tau_b$)                  (function of token, not L)
$\tau_b$ = CapabilityIssuer(event_metadata)                 (L not invoked, Def. 3 + Thm. 5)

Adversary controls: p (via injection quality), q (via paraphrasing skill)
Adversary cannot control: $\alpha_R$, $\alpha_X$, $R_2$ (determined before L runs)                  ∎
---

## 7. Concrete Instantiation

Parameters: n = 55, k = 5, n_R = 8, n_X = 6, p = 0.5, q = 0.1, $R_2$ $\in$ {0, 1}, B = 288.

**Case 1: $R_2$ = 1.**

Pr[Exfil(b)] = 0                                       (Theorem 3)

**Case 2: $R_2$ = 0 (worst case).**

Union bound (Theorem 1):

$\alpha_R$ = kn_R/n = 5 · 8 / 55 = 40/55 ≈ 0.727
$\alpha_X$ = kn_X/n = 5 · 6 / 55 = 30/55 ≈ 0.545
min($\alpha_R$, $\alpha_X$) = 0.545

Pr[Exfil(b)]
$\leq$ p · min($\alpha_R$, $\alpha_X$) · (1 - $R_2$) · q
= 0.5 · 0.545 · 1.0 · 0.1
= 0.0273

Exact hypergeometric values (for comparison with union bound):

Pr[B_R] = 1 - C(47,5)/C(55,5) = 1 - (47·46·45·44·43)/(55·54·53·52·51) ≈ 0.568
Pr[B_X] = 1 - C(49,5)/C(55,5) = 1 - (49·48·47·46·45)/(55·54·53·52·51) ≈ 0.452

The union bounds $\alpha_R$ = 0.727 and $\alpha_X$ = 0.545 are conservative (true values are 0.568 and 0.452). This confirms the bound is valid but not tight — the actual system is safer than the bound suggests.

Session level (union bound, no independence required):

Pr[$\exists$ b: Exfil(b)]
$\leq$ B · $\varepsilon$                                                (union bound)
= 288 · 0.0273
= 7.85

Since probabilities are capped at 1, this gives Pr $\leq$ 1 (trivial in worst case without $R_2$).

Under independence assumption:

Pr[$\exists$ b: Exfil(b)]
$\leq$ 1 - (1 - 0.0273)^{288}
= 1 - 0.9727^{288}
≈ 1 - 0.000275
= 0.9997

**Comparison with ambient authority:**

| Metric | Ambient authority | Event-scoped ($R_2$=0) | Event-scoped ($R_2$=1) |
|--------|-------------------|---------------------|----------------------|
| Pr[Exfil(b)] | $\geq$ 0.5 | $\leq$ 0.0273 | = 0 |
| Per-batch reduction | — | 18× | ∞ |
| Session (B=288) | ≈ 1.0 | $\leq$ 0.9997 | = 0 |
