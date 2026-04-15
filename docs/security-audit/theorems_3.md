# Supplementary Note 3: Causal Taint Tracking Properties

## Definitions

**Definition 14 (Taint Lattice).** Let (Λ, ≤) be a totally ordered set Λ = {USER, INTERNAL, EXTERNAL} with USER < INTERNAL < EXTERNAL. Λ forms a bounded lattice with ⊥ = USER, ⊤ = EXTERNAL, meet a ∧ b = min(a, b), and join a ∨ b = max(a, b).

**Fact (Lattice Properties).** For all a, b ∈ Λ:
- (F1) max(a, b) ≥ a  (by definition of max on a total order)
- (F2) max(a, b) ≥ b  (symmetric)
- (F3) a ≤ ⊤ = EXTERNAL  (EXTERNAL is the top element)
- (F4) a = ⊤ ∧ b ≤ ⊤ ⟹ max(a, b) = ⊤  (top absorbs)

**Definition 15 (Tool Taint Assignment).** Each tool t ∈ 𝒯 has a fixed taint level taint: 𝒯 → Λ defined by the trust classification of t's data source. Concretely: taint(read_file) = INTERNAL, taint(web_fetch) = EXTERNAL, taint(memory_search) = USER.

**Definition 16 (CausalTaintTracker).** A CausalTaintTracker for a batch of m tool executions is a sequence (τ₀, τ₁, ..., τₘ) ∈ Λᵐ⁺¹ defined inductively:

τ₀ = ⊥ = USER     ... (init)
τₖ = max(τₖ₋₁, taint(tₖ))     for k = 1, ..., m     ... (update)

where tₖ ∈ 𝒯 is the tool executed at step k.

## Lemma 1: Monotonicity

**Lemma 1.** For all 0 ≤ i ≤ j ≤ m: τᵢ ≤ τⱼ.

**Proof.** We first prove the single-step case, then extend by transitivity.

*Single-step claim:* For all k ∈ {1, ..., m}: τₖ₋₁ ≤ τₖ.

By (update): τₖ = max(τₖ₋₁, taint(tₖ)). By (F1): max(τₖ₋₁, taint(tₖ)) ≥ τₖ₋₁. Therefore:

τₖ₋₁ ≤ max(τₖ₋₁, taint(tₖ)) = τₖ     ... (mono-step)

*General case:* Let 0 ≤ i ≤ j ≤ m. If i = j, then τᵢ ≤ τⱼ holds trivially (≤ is reflexive on Λ). If i < j, apply (mono-step) repeatedly:

τᵢ ≤ τᵢ₊₁ ≤ τᵢ₊₂ ≤ ... ≤ τⱼ     (j − i applications of (mono-step))

By transitivity of ≤: τᵢ ≤ τⱼ. ∎

## Corollary 1: Irrevocability

**Corollary 1 (Irrevocable Contamination).** If ∃ k₀ ∈ {0, ..., m} such that τₖ₀ = EXTERNAL, then ∀ k ∈ {k₀, ..., m}: τₖ = EXTERNAL.

**Proof.** Let k ≥ k₀. By Lemma 1:

τₖ ≥ τₖ₀     ... (by monotonicity, since k₀ ≤ k)
τₖ₀ = EXTERNAL = ⊤     ... (by assumption)
∴ τₖ ≥ ⊤     ... (combining the above)

Since ⊤ is the maximum element of Λ, τₖ ≤ ⊤ for all τₖ ∈ Λ (by F3). Combined with τₖ ≥ ⊤:

τₖ = ⊤ = EXTERNAL     ... (antisymmetry of ≤) ∎

## Proposition 1: Causal Blocking Guarantee

**Definition 17 (Causal Policy).** A causal policy for tool t specifies a maximum causal taint level max_causal(t) ∈ Λ. Tool t is **causally allowed** at step k if τₖ₋₁ ≤ max_causal(t), and **causally blocked** if τₖ₋₁ > max_causal(t).

**Proposition 1 (Causal Blocking).** Let t be a tool with max_causal(t) = τ_max ∈ Λ. Let tₖ₀ be a tool executed at step k₀ with taint(tₖ₀) = τ_high where τ_high > τ_max. Then for all k > k₀, tool t is causally blocked at step k.

**Proof.**

By (update) applied at step k₀:

τₖ₀ = max(τₖ₀₋₁, taint(tₖ₀))     ... (i)

By (F2): max(τₖ₀₋₁, taint(tₖ₀)) ≥ taint(tₖ₀). Combined with (i):

τₖ₀ ≥ taint(tₖ₀) = τ_high     ... (ii)

Now let k > k₀. Then k − 1 ≥ k₀. By Lemma 1:

τₖ₋₁ ≥ τₖ₀     (monotonicity, since k₀ ≤ k − 1)     ... (iii)

Combining (ii) and (iii) by transitivity:

τₖ₋₁ ≥ τₖ₀ ≥ τ_high > τ_max = max_causal(t)     ... (iv)

By Definition 17, τₖ₋₁ > max_causal(t) means t is causally blocked at step k. ∎

**Instantiation.** In our architecture, external-action tools have max_causal(t) = INTERNAL, and web_fetch has taint(web_fetch) = EXTERNAL. Since EXTERNAL > INTERNAL, Proposition 1 yields: after web_fetch executes at step k₀, all external-action tools are causally blocked at every subsequent step k > k₀ within the same batch.
