# Supplementary Note 3: Causal Taint Tracking Properties

## Definitions

**Definition 14 (Taint Lattice).** Let (L, ≤) be a totally ordered set L = {USER, INTERNAL, EXTERNAL} with USER < INTERNAL < EXTERNAL. L forms a bounded lattice with ⊥ = USER, ⊤ = EXTERNAL, meet a ∧ b = min(a, b), and join a ∨ b = max(a, b).

**Definition 15 (Tool Taint Assignment).** Each tool t ∈ T has a fixed taint level assignment taint: T → L defined by the source trust of the tool's return values. For example: taint(read_file) = INTERNAL, taint(web_fetch) = EXTERNAL, taint(memory_search) = USER.

**Definition 16 (CausalTaintTracker).** A CausalTaintTracker for a batch of m tool executions is a sequence of taint levels τ = (τ₀, τ₁, ..., τₘ) defined inductively:
- τ₀ = USER (initial state at batch start)
- τₖ = τₖ₋₁ ∨ taint(tₖ) = max(τₖ₋₁, taint(tₖ)) for k = 1, ..., m

where tₖ is the tool executed at step k.

## Lemma 1: Monotonicity

**Lemma 1.** For all 0 ≤ i ≤ j ≤ m: τᵢ ≤ τⱼ.

**Proof.** By induction on j − i.

*Base case (j = i):* τᵢ ≤ τᵢ holds trivially.

*Inductive step:* Assume τᵢ ≤ τⱼ for all i ≤ j ≤ k. We show τᵢ ≤ τₖ₊₁.

By definition, τₖ₊₁ = max(τₖ, taint(tₖ₊₁)). Since max(a, b) ≥ a for all a, b ∈ L:

τₖ₊₁ = max(τₖ, taint(tₖ₊₁)) ≥ τₖ ≥ τᵢ

where the last inequality holds by the inductive hypothesis. ∎

## Corollary 1: Irrevocability

**Corollary 1 (Irrevocable Contamination).** If ∃ k₀ ≤ m such that τₖ₀ = EXTERNAL, then ∀ k ≥ k₀: τₖ = EXTERNAL.

**Proof.** By Lemma 1, τₖ ≥ τₖ₀ = EXTERNAL. Since EXTERNAL = ⊤ (the maximum element of L), τₖ = EXTERNAL. ∎

## Proposition 1: Causal Blocking Guarantee

**Definition 17 (Causal Policy).** A causal policy for tool t specifies a maximum causal taint level max_causal(t) ∈ L. Tool t is causally blocked at step k if τₖ₋₁ > max_causal(t).

**Proposition 1 (Causal Blocking).** Let t be a tool with causal policy max_causal(t) = τ_max for some τ_max ∈ L. Let tₖ₀ be a tool executed at step k₀ with taint(tₖ₀) = τ_high where τ_high > τ_max. Then for all k > k₀, tool t is causally blocked at step k.

**Proof.** By Definition 16, τₖ₀ = max(τₖ₀₋₁, taint(tₖ₀)) ≥ taint(tₖ₀) = τ_high. By Lemma 1 (monotonicity), for all k > k₀: τₖ₋₁ ≥ τₖ₀ ≥ τ_high > τ_max = max_causal(t). By Definition 17, t is causally blocked at step k. ∎

**Instantiation.** In our architecture, external-action tools (send_email, curl, bash) have max_causal(t) = INTERNAL, and tools that fetch untrusted data (web_fetch, web_search) have taint level EXTERNAL. Proposition 1 with τ_max = INTERNAL and τ_high = EXTERNAL yields: once any EXTERNAL data source is consulted in a batch, all external-action tools are permanently blocked for the remainder of that batch, regardless of intermediate tool results.
