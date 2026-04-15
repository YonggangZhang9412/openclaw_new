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
