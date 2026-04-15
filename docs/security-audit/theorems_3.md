## Theorem 3: CausalTaintTracker Monotonicity

### 3.1 Definitions

**Definition 11 (Taint Level).** TaintLevel is a totally ordered set {USER, INTERNAL, EXTERNAL} with ordering USER < INTERNAL < EXTERNAL. We write τ₁ ≤ τ₂ to denote that τ₁ is at most as tainted as τ₂.

**Definition 12 (CausalTaintTracker State).** Within an event batch, the CausalTaintTracker maintains a state variable τ_round ∈ TaintLevel, initialized to USER at the beginning of each batch:

τ_round(0) = USER

**Definition 13 (Update Rule).** After each tool execution at step k that returns a result with taint level τ_result(k), the tracker updates:

τ_round(k) = max(τ_round(k-1), τ_result(k))

where max is taken with respect to the total order on TaintLevel.

### 3.2 Theorem Statement

**Theorem 3 (Monotonicity).** For any event batch with tool execution steps 0 < k₁ < k₂, the causal taint level is monotonically non-decreasing:

τ_round(k₁) ≤ τ_round(k₂)

Equivalently: once τ_round reaches a given level, it never decreases within the batch.

### 3.3 Proof

By induction on the step index k.

**Base case (k=1).** τ_round(1) = max(τ_round(0), τ_result(1)) = max(USER, τ_result(1)) ≥ USER = τ_round(0). ✓

**Inductive step.** Assume τ_round(k) ≥ τ_round(k-1) for all steps up to k. At step k+1:

τ_round(k+1) = max(τ_round(k), τ_result(k+1))

Since max(a, b) ≥ a for any a, b in a totally ordered set:

τ_round(k+1) = max(τ_round(k), τ_result(k+1)) ≥ τ_round(k)

Therefore τ_round(k+1) ≥ τ_round(k). By induction, τ_round is monotonically non-decreasing. ∎

### 3.4 Corollary: Irrevocability of Contamination

**Corollary.** If at any step k₀ within a batch, τ_round(k₀) = EXTERNAL, then for all subsequent steps k > k₀ in the same batch: τ_round(k) = EXTERNAL.

Proof. By Theorem 3, τ_round(k) ≥ τ_round(k₀) = EXTERNAL. Since EXTERNAL is the maximum element of TaintLevel, τ_round(k) = EXTERNAL. ∎

This corollary formalizes the "irrevocable contamination" property: once external data enters the LLM's context within a batch, the CausalTaintTracker permanently classifies the batch as contaminated. No subsequent tool execution can reduce the taint level.

### 3.5 Security Implication

If a tool policy requires causal taint ≤ INTERNAL for a tool t (e.g., tools performing external actions), then once any EXTERNAL data is observed in the batch, t is permanently blocked for the remainder of that batch. This provides a fail-safe guarantee independent of TaintStore's content-level tracking.

---

## Theorem 4: Privilege Exposure Bound

### 4.1 Definitions

**Definition 14 (Privilege Exposure).** The privilege exposure of an agent system over a time interval [0, T] is defined as:

Ξ = ∫₀ᵀ |P(t)| dt

where P(t) ⊆ T is the set of tools authorized at time t, and |P(t)| is its cardinality. Ξ measures the cumulative "tool-time" exposure — the total opportunity for an attacker to exploit any authorized tool.

### 4.2 Privilege Exposure Under Ambient Authority

Under ambient authority, P(t) = P₀ for all t ∈ [0, T_session]:

Ξ_ambient = |P₀| × T_session

For a system with |P₀| = 55 tools and a 24-hour session (T_session = 86,400s):

Ξ_ambient = 55 × 86,400 = 4,752,000 tool-seconds

### 4.3 Privilege Exposure Under Event-Scoped Capability Authority

Under event-scoped capability authority, the session is divided into event batches b₁, b₂, ..., b_B. Each batch bᵢ receives a token with tool set P_bᵢ and TTL Δᵢ ≤ TTL_max (= 300s). Between batches, P(t) = ∅.

Ξ_capability = Σᵢ₌₁ᴮ |P_bᵢ| × Δᵢ

### 4.4 Theorem Statement

**Theorem 4 (Privilege Exposure Bound).** Let k = max_i |P_bᵢ| be the maximum number of tools granted per batch. Then:

Ξ_capability ≤ B × k × TTL_max

and the reduction ratio satisfies:

Ξ_capability / Ξ_ambient ≤ (k / |P₀|) × (B × TTL_max / T_session)

### 4.5 Proof

Ξ_capability = Σᵢ₌₁ᴮ |P_bᵢ| × Δᵢ ≤ Σᵢ₌₁ᴮ k × TTL_max = B × k × TTL_max

Dividing by Ξ_ambient = |P₀| × T_session:

Ξ_capability / Ξ_ambient ≤ (B × k × TTL_max) / (|P₀| × T_session) = (k / |P₀|) × (B × TTL_max / T_session) ∎

### 4.6 Concrete Bound

With k = 5, |P₀| = 55, TTL_max = 300s, T_session = 86,400s:

- If batches arrive at most once per TTL: B ≤ T_session / TTL_max = 288
- Ξ_capability / Ξ_ambient ≤ (5/55) × (288 × 300 / 86,400) = 0.0909 × 1.0 = **0.0909**

The event-scoped architecture exposes at most **9.1%** of the privilege surface of the ambient authority model under worst-case batch frequency. In practice, batches are typically shorter than TTL_max and k is often 2-3 (not 5), yielding significantly lower exposure.

### 4.7 Note on Interpretation

Privilege exposure Ξ is not a probability. It is a **structural measure of opportunity**: the total tool-time window during which an attacker who has achieved prompt injection could exploit a sensitive tool. A lower Ξ means the attacker has less opportunity to act — even if the injection succeeds.
