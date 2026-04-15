# Supplementary Note 3: Causal Taint Tracking Properties

## Definitions

**Definition 18 (Taint Lattice).** Let (Λ, ≤) be a totally ordered set Λ = {USER, INTERNAL, EXTERNAL} with USER < INTERNAL < EXTERNAL. Λ forms a bounded lattice with ⊥ = USER, ⊤ = EXTERNAL, meet a ∧ b = min(a, b), and join a ∨ b = max(a, b).

**Fact (Lattice Properties).** For all a, b ∈ Λ:
- (F1) max(a, b) ≥ a  (by definition of max on a total order)
- (F2) max(a, b) ≥ b  (symmetric)
- (F3) a ≤ ⊤ = EXTERNAL  (EXTERNAL is the top element)
- (F4) a = ⊤ ∧ b ≤ ⊤ ⟹ max(a, b) = ⊤  (top absorbs)

**Definition 19 (Tool Taint Assignment).** Each tool t ∈ 𝒯 has a fixed taint level taint: 𝒯 → Λ defined by the trust classification of t's data source. Concretely: taint(read_file) = INTERNAL, taint(web_fetch) = EXTERNAL, taint(memory_search) = USER.

**Definition 20 (CausalTaintTracker).** A CausalTaintTracker for a batch of m tool executions is a sequence (τ₀, τ₁, ..., τₘ) ∈ Λᵐ⁺¹ defined inductively:

τ₀ = ⊥ = USER     ... (init)
τₖ = max(τₖ₋₁, taint(tₖ))     for k = 1, ..., m     ... (update)

where tₖ ∈ 𝒯 is the tool executed at step k.

## Lemma 1: Monotonicity

**Lemma 1.** For all 0 ≤ i ≤ j ≤ m: τᵢ ≤ τⱼ.

**Proof.** First, the single-step case. For any k ∈ {1, ..., m}:

τₖ = max(τₖ₋₁, taint(tₖ))         [by (update)]
   ≥ τₖ₋₁                          [by (F1): max(a,b) ≥ a]             ... (†)

For the general case, let 0 ≤ i < j ≤ m. Applying (†) at k = i+1, i+2, ..., j:

τᵢ ≤ τᵢ₊₁ ≤ τᵢ₊₂ ≤ ··· ≤ τⱼ      [j − i applications of (†)]

By transitivity of ≤ on Λ: τᵢ ≤ τⱼ. The case i = j holds by reflexivity. ∎

## Corollary 1: Irrevocability

**Corollary 1 (Irrevocable Contamination).** If ∃ k₀ ∈ {0, ..., m} such that τₖ₀ = EXTERNAL, then ∀ k ∈ {k₀, ..., m}: τₖ = EXTERNAL.

**Proof.** Let k ≥ k₀.

⊤ = EXTERNAL = τₖ₀          [by assumption]
              ≤ τₖ           [by Lemma 1, since k₀ ≤ k]
              ≤ ⊤            [by (F3): ∀ a ∈ Λ, a ≤ ⊤]

Therefore τₖ = ⊤ = EXTERNAL (by antisymmetry of ≤). ∎

## Proposition 1: Causal Blocking Guarantee

**Definition 21 (Causal Policy).** A causal policy for tool t specifies a maximum causal taint level max_causal(t) ∈ Λ. Tool t is **causally allowed** at step k if τₖ₋₁ ≤ max_causal(t), and **causally blocked** if τₖ₋₁ > max_causal(t).

**Proposition 1 (Causal Blocking).** Let t be a tool with max_causal(t) = τ_max ∈ Λ. Let tₖ₀ be a tool executed at step k₀ with taint(tₖ₀) = τ_high where τ_high > τ_max. Then for all k > k₀, tool t is causally blocked at step k.

**Proof.** Let k > k₀. Then k − 1 ≥ k₀. We derive:

τₖ₋₁ ≥ τₖ₀                               [by Lemma 1, since k₀ ≤ k − 1]
     = max(τₖ₀₋₁, taint(tₖ₀))           [by (update)]
     ≥ taint(tₖ₀)                         [by (F2): max(a,b) ≥ b]
     = τ_high                              [by assumption]
     > τ_max                               [by assumption: τ_high > τ_max]
     = max_causal(t)                       [by assumption]

Therefore τₖ₋₁ > max_causal(t), so t is causally blocked at step k (Definition 21). ∎

**Instantiation.** In our architecture, external-action tools have max_causal(t) = INTERNAL, and web_fetch has taint(web_fetch) = EXTERNAL. Since EXTERNAL > INTERNAL, Proposition 1 yields: after web_fetch executes at step k₀, all external-action tools are causally blocked at every subsequent step k > k₀ within the same batch.
