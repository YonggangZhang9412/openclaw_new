# Supplementary Note 2: Rule of Two Sufficiency

## Definitions

We extend the formal model of Supplementary Note 1.

**Definition 10 (Data Item).** A data item is a pair d = (content, label) where content ∈ {0,1}* is an arbitrary byte string and label ∈ {PUBLIC, SENSITIVE}. Let D_S = {d ∈ D | d.label = SENSITIVE}.

**Definition 11 (System Boundary).** The system boundary ∂ partitions all storage locations into Internal and External. We write ℓ(d) for the storage location of data item d.

**Definition 12 (Tool Classification).** Each tool t ∈ 𝒯 is classified by three predicates:
- reads_sensitive(t) ≡ ∃ d ∈ D_S: d ∈ PossibleOutputs(t)
- sends_external(t) ≡ ∃ ℓ ∈ External: ℓ ∈ PossibleDestinations(t)
- receives_untrusted(t) ≡ ∃ d: source(d) ∉ Trusted ∧ d ∈ PossibleIntroductions(t)

We define:
- 𝒯_R = {t ∈ 𝒯 | reads_sensitive(t)}
- 𝒯_X = {t ∈ 𝒯 | sends_external(t)}
- 𝒯_U = {t ∈ 𝒯 | receives_untrusted(t)}

These sets may overlap. For compound tools (e.g., bash): bash ∈ 𝒯_R ∩ 𝒯_X is possible.

**Definition 13 (Data Flow Relation).** For a tool invocation sequence σ = ((t₁, a₁), ..., (tₘ, aₘ)), we define the data flow relation →_σ over data items:

d →_σ d' ≡ ∃ i, j with i ≤ j: d ∈ result(tᵢ, aᵢ) ∧ d.content ⊑ d'.content ∧ d' ∈ transmitted(tⱼ, aⱼ)

where ⊑ denotes the substring relation on byte strings (d.content is a contiguous subsequence of d'.content). Intuitively, d →_σ d' means data item d, produced by tool invocation i, has its content (or content containing it) transmitted by tool invocation j.

**Definition 14 (Adversary-Triggered Exfiltration).** A tool invocation sequence σ executed within a single event batch constitutes an adversary-triggered exfiltration if and only if:

**(E1)** ∃ d_u ∈ data(state(batch)): source(d_u) ∉ Trusted ∧ ∃ a ∈ 𝒜: Pr[L(s) = a] ≠ Pr[L(s \ {d_u}) = a]

**(E2)** ∃ i ∈ {1,...,m}: tᵢ ∈ 𝒯_R ∧ result(tᵢ, aᵢ) ∩ D_S ≠ ∅

**(E3)** ∃ d_s ∈ D_S, ∃ d' ∈ transmitted(tⱼ, aⱼ) for some j: d_s →_σ d'

That is: (E1) adversarial influence is present, (E2) some tool reads sensitive data, and (E3) sensitive data flows to an external destination through the data flow relation.

**Definition 15 (Batch-Level Condition Predicates).** For a CapabilityToken τ:

- U(τ) ≡ τ.granted_tools ∩ 𝒯_U ≠ ∅
- S(τ) ≡ τ.granted_tools ∩ 𝒯_R ≠ ∅
- X(τ) ≡ τ.granted_tools ∩ 𝒯_X ≠ ∅

**Definition 16 (Rule of Two Constraint).** Token τ satisfies the Rule of Two if:

¬(U(τ) ∧ S(τ) ∧ X(τ))     ... (R2)

**Definition 17 (Exfiltration Feasibility).** For a token τ under Gate enforcement, define:

Φ(τ) = 𝟙[U(τ)] · 𝟙[S(τ)] · 𝟙[X(τ)]

Φ(τ) = 1 if and only if the token simultaneously grants untrusted-input, sensitive-read, and external-action tools — the three necessary capabilities for an exfiltration chain. Under (R2): Φ(τ) = 0.

## Theorem

**Theorem 2 (Rule of Two Sufficiency).** Let σ = ((t₁, a₁), ..., (tₘ, aₘ)) be executed under token τ with Gate enforcement (∀ i: tᵢ ∈ τ.granted_tools) and no cross-batch contamination. If τ satisfies (R2), then Φ(τ) = 0 and σ cannot be an adversary-triggered exfiltration.

## Proof

Assume for contradiction that σ is an adversary-triggered exfiltration under token τ satisfying (R2), with no cross-batch contamination.

**Step 1.** By E2 (Definition 14):

∃ i ∈ {1,...,m}: tᵢ ∈ 𝒯_R ∧ result(tᵢ, aᵢ) ∩ D_S ≠ ∅                  ... (1)

By Gate enforcement: tᵢ ∈ τ.granted_tools. Combining with tᵢ ∈ 𝒯_R:

τ.granted_tools ∩ 𝒯_R ⊇ {tᵢ} ≠ ∅                                       ... (2)
⟹ S(τ) = true     [by Definition 15]                                   ... (3)

**Step 2.** By E3 (Definition 14):

∃ d_s ∈ D_S, ∃ j ∈ {1,...,m}: d_s →_σ d' ∧ d' ∈ transmitted(tⱼ, aⱼ)    ... (4)

Since transmitted(tⱼ, aⱼ) ≠ ∅, by Definition 12: sends_external(tⱼ) = true, so tⱼ ∈ 𝒯_X. By Gate enforcement: tⱼ ∈ τ.granted_tools. Therefore:

τ.granted_tools ∩ 𝒯_X ⊇ {tⱼ} ≠ ∅                                       ... (5)
⟹ X(τ) = true     [by Definition 15]                                   ... (6)

**Step 3.** By E1 (Definition 14):

∃ d_u ∈ data(state(batch)): source(d_u) ∉ Trusted                       ... (7)

By the no-cross-batch assumption, d_u was introduced by some tₖ in σ:

receives_untrusted(tₖ) = true ⟹ tₖ ∈ 𝒯_U     [by Definition 12]       ... (8)

By Gate enforcement: tₖ ∈ τ.granted_tools. Therefore:

τ.granted_tools ∩ 𝒯_U ⊇ {tₖ} ≠ ∅                                       ... (9)
⟹ U(τ) = true     [by Definition 15]                                   ... (10)

**Step 4.** Collecting (3), (6), (10) and computing Φ(τ) (Definition 17):

Φ(τ) = 𝟙[U(τ)] · 𝟙[S(τ)] · 𝟙[X(τ)]
     = 𝟙[true] · 𝟙[true] · 𝟙[true]         [by (3), (6), (10)]
     = 1 · 1 · 1
     = 1                                                                  ... (11)

But τ satisfies (R2), so by Definition 17:

Φ(τ) = 0                                                                 ... (12)

(11) and (12) give Φ(τ) = 1 = 0, a contradiction. ∎

## Remarks

**On the no-cross-batch-contamination assumption.** If untrusted data d_u entered data(state(batch)) from a prior batch's session history rather than from a tool in σ, then Step 3 fails: no tₖ ∈ 𝒯_U was invoked in σ, so U(τ) may be false even though E1 is satisfied. This is the cross-batch contamination gap discussed in Section 7.3 of the main text. The theorem's scope is explicitly per-batch.

**On compound tools.** If t ∈ 𝒯_R ∩ 𝒯_X (e.g., bash can both read files and make network requests), then a single tool invocation (t, a) can satisfy both E2 and E3 in one step — in particular, d_s →_σ d' may hold with i = j. The proof is unaffected: Step 1 yields S(τ) = true, Step 2 yields X(τ) = true (possibly from the same tool), and Step 3 independently requires U(τ) = true. The constraint ¬(U ∧ S ∧ X) then forces U(τ) = false, meaning tokens granting compound tools like bash must not grant untrusted-input tools in the same batch.

**On tool classification correctness.** The theorem holds relative to the classification 𝒯_R, 𝒯_X, 𝒯_U. If a tool is misclassified (e.g., a tool that can exfiltrate is not in 𝒯_X), then the formal conclusion still holds but the real-world security guarantee is weakened. Policy correctness is an orthogonal concern discussed in Section 7.4.
