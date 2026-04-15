# Supplementary Note 2: Rule of Two Sufficiency

## Definitions

We extend the formal model of Supplementary Note 1.

**Definition 8 (Data Item).** A data item is a pair d = (content, label) where content is an arbitrary byte string and label ∈ {PUBLIC, SENSITIVE}. Let D_S = {d ∈ D | d.label = SENSITIVE} denote the set of sensitive data items.

**Definition 9 (System Boundary).** The system boundary ∂ partitions all storage locations into Internal (within the agent's host) and External (accessible by entities outside the agent's control). A data item d is internal if its storage location ℓ(d) ∈ Internal.

**Definition 10 (Tool Classification).** Each tool t ∈ 𝒯 is classified along three boolean dimensions:

- reads_sensitive(t) = true iff executing t can produce output containing d ∈ D_S
- sends_external(t) = true iff executing t can transmit data to a location ℓ ∈ External
- receives_untrusted(t) = true iff executing t introduces data from sources ∉ Trusted into the agent's state

We define:
- 𝒯_R = {t ∈ 𝒯 | reads_sensitive(t) = true}
- 𝒯_X = {t ∈ 𝒯 | sends_external(t) = true}
- 𝒯_U = {t ∈ 𝒯 | receives_untrusted(t) = true}

Note: these sets are not necessarily disjoint. A tool t may belong to multiple sets (e.g., bash ∈ 𝒯_R ∩ 𝒯_X).

**Definition 11 (Adversary-Triggered Exfiltration).** Let σ = ((t₁, a₁), (t₂, a₂), ..., (tₘ, aₘ)) be a tool invocation sequence executed within a single event batch. σ constitutes an adversary-triggered exfiltration if and only if all of the following hold:

(E1) **Untrusted influence.** The event batch triggering σ contains data from an untrusted source that causally influenced L's generation of σ. Formally: ∃ d_u with source(d_u) ∉ Trusted such that d_u ∈ state(batch), and letting s = state_with(d_u) and s' = state_without(d_u):

∃ a ∈ 𝒜: Pr[L(s) = a] ≠ Pr[L(s') = a]

That is, the presence of d_u measurably alters L's output distribution over tool invocations.

(E2) **Sensitive data read.** ∃ i ∈ {1,...,m}: tᵢ ∈ 𝒯_R ∧ result(tᵢ, aᵢ) ∩ D_S ≠ ∅.

(E3) **External transmission of sensitive content.** ∃ j ∈ {1,...,m}: tⱼ ∈ 𝒯_X ∧ transmitted(tⱼ, aⱼ) ∩ content(D_S) ≠ ∅, where content(D_S) = {d.content | d ∈ D_S} is the set of sensitive content values and transmitted(t, a) is the set of data items sent to External by invocation (t, a).

E1 is part of the definition, not a theorem conclusion. This avoids circularity: an "attack" is definitionally adversary-triggered.

**Definition 12 (Batch-Level Condition Predicates).** For a CapabilityToken τ issued for an event batch, define:

- U(τ) ≡ τ.granted_tools ∩ 𝒯_U ≠ ∅ (token grants tools that receive untrusted data)
- S(τ) ≡ τ.granted_tools ∩ 𝒯_R ≠ ∅ (token grants tools that read sensitive data)
- X(τ) ≡ τ.granted_tools ∩ 𝒯_X ≠ ∅ (token grants tools that send externally)

**Definition 13 (Rule of Two Constraint).** A CapabilityToken τ satisfies the Rule of Two if:

|{U(τ), S(τ), X(τ)} ∩ {true}| ≤ 2

Equivalently: ¬(U(τ) ∧ S(τ) ∧ X(τ)).

## Theorem Statement

**Theorem 2 (Rule of Two Sufficiency).** Let σ be a tool invocation sequence executed under a CapabilityToken τ that satisfies the Rule of Two. Assume that the CapabilityGate enforces τ.granted_tools (i.e., ∀ i: tᵢ ∈ τ.granted_tools). Then σ cannot constitute an adversary-triggered exfiltration.

## Proof

We prove by contradiction. Assume σ is an adversary-triggered exfiltration under token τ satisfying the Rule of Two.

**Step 1 (From E2 to S).** By Definition 11 (E2), ∃ tᵢ ∈ 𝒯_R in σ. Since the Gate enforces τ.granted_tools, tᵢ ∈ τ.granted_tools. Therefore τ.granted_tools ∩ 𝒯_R ≠ ∅, which means S(τ) = true.

**Step 2 (From E3 to X).** By Definition 11 (E3), ∃ tⱼ ∈ 𝒯_X in σ. By the same Gate enforcement argument, tⱼ ∈ τ.granted_tools. Therefore τ.granted_tools ∩ 𝒯_X ≠ ∅, which means X(τ) = true.

**Step 3 (From E1 to U).** By Definition 11 (E1), the batch contains untrusted data that causally influenced σ. For this untrusted data to enter the agent's state during this batch, either:
- (a) The untrusted data was already in session history from a prior batch (cross-batch contamination — outside scope; see Remark below), or
- (b) Some tool tₖ ∈ 𝒯_U was invoked in this batch, introducing untrusted data. By Gate enforcement, tₖ ∈ τ.granted_tools, so τ.granted_tools ∩ 𝒯_U ≠ ∅, meaning U(τ) = true.

Under case (b): U(τ) = true.

**Step 4 (Contradiction).** From Steps 1-3 (case b): U(τ) ∧ S(τ) ∧ X(τ) = true. Therefore |{U(τ), S(τ), X(τ)} ∩ {true}| = 3 > 2, contradicting the Rule of Two constraint (Definition 13).

Therefore, under case (b), σ cannot be an adversary-triggered exfiltration. ∎

## Remarks

**On case (a): cross-batch contamination.** If untrusted data from a prior batch persists in session history and influences the current batch's LLM behavior without any 𝒯_U tool being invoked in the current batch, then U(τ) may be false while E1 is still satisfied through historical context. In this case, the Rule of Two does not prevent exfiltration, because the condition U is evaluated at the token level (which tools are granted) rather than at the context level (what data has historically influenced the LLM).

This is the cross-batch contamination gap discussed in Section 7.3 of the main text. The gap is bounded: it requires (i) a prior batch that processed untrusted data, (ii) that data persisting in session history, (iii) the current batch being classified as trusted (granting sensitive + external tools), and (iv) the LLM being influenced by the historical untrusted data to construct an exfiltration sequence. Quantifying the practical frequency of this scenario is Experiment 4 in the evaluation framework.

**On compound tools.** A tool t ∈ 𝒯_R ∩ 𝒯_X (e.g., bash, which can both read files and send network requests) is classified in both 𝒯_R and 𝒯_X. This does not weaken the theorem: if τ grants such a tool, then both S(τ) and X(τ) are true, requiring U(τ) = false for the Rule of Two to hold. This means any batch where bash is granted must not also grant untrusted-input tools — a correct and intended restriction.

**On the definition of 𝒯_R, 𝒯_X, 𝒯_U.** The correctness of the Rule of Two depends on the accuracy of tool classification. If a tool is incorrectly classified (e.g., a tool that can exfiltrate data is not placed in 𝒯_X), the theorem's conclusion still holds formally, but the real-world security guarantee is weakened. Policy completeness is discussed in Section 7.4 of the main text.
