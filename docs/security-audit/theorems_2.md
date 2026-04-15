# Supplementary Note 2: Rule of Two Sufficiency

## Definitions

We extend the formal model of Supplementary Note 1.

**Definition 8 (Data Item).** A data item is a pair d = (content, label) where content ∈ {0,1}* is an arbitrary byte string and label ∈ {PUBLIC, SENSITIVE}. Let D_S = {d ∈ D | d.label = SENSITIVE}.

**Definition 9 (System Boundary).** The system boundary ∂ partitions all storage locations into Internal and External. We write ℓ(d) for the storage location of data item d.

**Definition 10 (Tool Classification).** Each tool t ∈ 𝒯 is classified by three predicates:
- reads_sensitive(t) ≡ ∃ d ∈ D_S: d ∈ PossibleOutputs(t)
- sends_external(t) ≡ ∃ ℓ ∈ External: ℓ ∈ PossibleDestinations(t)
- receives_untrusted(t) ≡ ∃ d: source(d) ∉ Trusted ∧ d ∈ PossibleIntroductions(t)

We define:
- 𝒯_R = {t ∈ 𝒯 | reads_sensitive(t)}
- 𝒯_X = {t ∈ 𝒯 | sends_external(t)}
- 𝒯_U = {t ∈ 𝒯 | receives_untrusted(t)}

These sets may overlap. For compound tools (e.g., bash): bash ∈ 𝒯_R ∩ 𝒯_X is possible.

**Definition 11 (Data Flow Relation).** For a tool invocation sequence σ = ((t₁, a₁), ..., (tₘ, aₘ)), we define the data flow relation →_σ over data items:

d →_σ d' ≡ ∃ i, j with i ≤ j: d ∈ result(tᵢ, aᵢ) ∧ d.content ⊑ d'.content ∧ d' ∈ transmitted(tⱼ, aⱼ)

where ⊑ denotes the substring relation on byte strings (d.content is a contiguous subsequence of d'.content). Intuitively, d →_σ d' means data item d, produced by tool invocation i, has its content (or content containing it) transmitted by tool invocation j.

**Definition 12 (Adversary-Triggered Exfiltration).** A tool invocation sequence σ executed within a single event batch constitutes an adversary-triggered exfiltration if and only if:

**(E1)** ∃ d_u ∈ data(state(batch)): source(d_u) ∉ Trusted ∧ ∃ a ∈ 𝒜: Pr[L(s) = a] ≠ Pr[L(s \ {d_u}) = a]

**(E2)** ∃ i ∈ {1,...,m}: tᵢ ∈ 𝒯_R ∧ result(tᵢ, aᵢ) ∩ D_S ≠ ∅

**(E3)** ∃ d_s ∈ D_S, ∃ d' ∈ transmitted(tⱼ, aⱼ) for some j: d_s →_σ d'

That is: (E1) adversarial influence is present, (E2) some tool reads sensitive data, and (E3) sensitive data flows to an external destination through the data flow relation.

**Definition 13 (Batch-Level Condition Predicates).** For a CapabilityToken τ:

- U(τ) ≡ τ.granted_tools ∩ 𝒯_U ≠ ∅
- S(τ) ≡ τ.granted_tools ∩ 𝒯_R ≠ ∅
- X(τ) ≡ τ.granted_tools ∩ 𝒯_X ≠ ∅

**Definition 14 (Rule of Two Constraint).** Token τ satisfies the Rule of Two if:

¬(U(τ) ∧ S(τ) ∧ X(τ))     ... (R2)

## Theorem

**Theorem 2 (Rule of Two Sufficiency).** Let σ = ((t₁, a₁), ..., (tₘ, aₘ)) be a tool invocation sequence executed under a CapabilityToken τ satisfying (R2). Assume Gate enforcement: ∀ i ∈ {1,...,m}: tᵢ ∈ τ.granted_tools. If all untrusted data in the current batch was introduced by tools within σ (no cross-batch contamination), then σ is not an adversary-triggered exfiltration.

## Proof

Assume for contradiction that σ is an adversary-triggered exfiltration under token τ satisfying (R2) with the no-cross-batch-contamination assumption.

**Step 1 (E2 implies S(τ)).**

By Definition 12 (E2):
∃ i: tᵢ ∈ 𝒯_R ∧ result(tᵢ, aᵢ) ∩ D_S ≠ ∅     ... (i)

By Gate enforcement: tᵢ ∈ τ.granted_tools. Combined with tᵢ ∈ 𝒯_R:

tᵢ ∈ τ.granted_tools ∩ 𝒯_R ≠ ∅     ... (ii)

By Definition 13: S(τ) = true     ... (1)

**Step 2 (E3 implies X(τ)).**

By Definition 12 (E3):
∃ d_s ∈ D_S, ∃ j: d_s →_σ d' ∧ d' ∈ transmitted(tⱼ, aⱼ)     ... (iii)

The existence of transmitted(tⱼ, aⱼ) ≠ ∅ implies tⱼ ∈ 𝒯_X (by Definition 10, sends_external(tⱼ) holds). By Gate enforcement: tⱼ ∈ τ.granted_tools. Therefore:

tⱼ ∈ τ.granted_tools ∩ 𝒯_X ≠ ∅     ... (iv)

By Definition 13: X(τ) = true     ... (2)

**Step 3 (E1 + no-cross-batch implies U(τ)).**

By Definition 12 (E1):
∃ d_u: source(d_u) ∉ Trusted ∧ d_u ∈ data(state(batch))     ... (v)

By the no-cross-batch-contamination assumption, d_u was introduced by some tool tₖ in σ. A tool that introduces untrusted data satisfies receives_untrusted(tₖ), so tₖ ∈ 𝒯_U (Definition 10). By Gate enforcement: tₖ ∈ τ.granted_tools. Therefore:

tₖ ∈ τ.granted_tools ∩ 𝒯_U ≠ ∅     ... (vi)

By Definition 13: U(τ) = true     ... (3)

**Step 4 (Contradiction).**

From (1), (2), (3):

U(τ) = true ∧ S(τ) = true ∧ X(τ) = true

Therefore:

U(τ) ∧ S(τ) ∧ X(τ) = true     ... (4)

But τ satisfies (R2) (Definition 14):

¬(U(τ) ∧ S(τ) ∧ X(τ))     ... (5)

(4) and (5) yield ⊥. Contradiction. ∎

## Remarks

**On the no-cross-batch-contamination assumption.** If untrusted data d_u entered data(state(batch)) from a prior batch's session history rather than from a tool in σ, then Step 3 fails: no tₖ ∈ 𝒯_U was invoked in σ, so U(τ) may be false even though E1 is satisfied. This is the cross-batch contamination gap discussed in Section 7.3 of the main text. The theorem's scope is explicitly per-batch.

**On compound tools.** If t ∈ 𝒯_R ∩ 𝒯_X (e.g., bash can both read files and make network requests), then a single tool invocation (t, a) can satisfy both E2 and E3 in one step — in particular, d_s →_σ d' may hold with i = j. The proof is unaffected: Step 1 yields S(τ) = true, Step 2 yields X(τ) = true (possibly from the same tool), and Step 3 independently requires U(τ) = true. The constraint ¬(U ∧ S ∧ X) then forces U(τ) = false, meaning tokens granting compound tools like bash must not grant untrusted-input tools in the same batch.

**On tool classification correctness.** The theorem holds relative to the classification 𝒯_R, 𝒯_X, 𝒯_U. If a tool is misclassified (e.g., a tool that can exfiltrate is not in 𝒯_X), then the formal conclusion still holds but the real-world security guarantee is weakened. Policy correctness is an orthogonal concern discussed in Section 7.4.
