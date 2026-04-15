## Theorem 2: Rule of Two Sufficiency

### 2.1 Definitions

**Definition 6 (Tool Invocation Sequence).** A tool invocation sequence within an event batch is σ = ((t₁, a₁), (t₂, a₂), ..., (tₘ, aₘ)) where tᵢ ∈ T is a tool and aᵢ ∈ Args is its argument set.

**Definition 7 (Sensitive Data).** Let S ⊂ Data be the set of sensitive data items (credentials, private keys, personal information, confidential files). A data item s ∈ S is initially internal: it resides within the system boundary and is not accessible to external entities.

**Definition 8 (External Endpoint).** Let Ext be the set of external endpoints (remote servers, email recipients, API endpoints outside the system boundary).

**Definition 9 (Exfiltration Attack).** A tool invocation sequence σ constitutes an exfiltration attack if there exist s ∈ S and e ∈ Ext such that:
1. Before executing σ, s is internal (not accessible to e)
2. After executing σ, the content of s (or information derived from s) has been transmitted to e

**Definition 10 (Attack Conditions).** For a tool invocation sequence σ, define three predicates:

- **U(σ)** (Untrusted input): The event batch that triggered σ contains data from an untrusted external source that influenced L's generation of σ. Formally: ∃ d_u ∈ C such that d_u originates from an untrusted source and L(C) ≠ L(C \ {d_u}), i.e., removing d_u would change the output.

- **S(σ)** (Sensitive data access): At least one tool invocation in σ reads sensitive data. Formally: ∃ i ∈ {1,...,m} such that tᵢ ∈ T_read and result(tᵢ, aᵢ) ∩ S ≠ ∅.

- **X(σ)** (External action): At least one tool invocation in σ transmits data to an external endpoint. Formally: ∃ j ∈ {1,...,m} such that tⱼ ∈ T_ext and dest(aⱼ) ∈ Ext.

**Note on compound tools.** A tool t that can both read sensitive data and transmit externally (e.g., `bash`) satisfies t ∈ T_read ∩ T_ext. This does not affect the theorem — such a tool contributes to both S(σ) and X(σ) simultaneously.

### 2.2 Theorem Statement

**Theorem 2 (Necessity of Three Conditions).** If σ is an exfiltration attack, then U(σ) ∧ S(σ) ∧ X(σ).

Equivalently, by contrapositive: if ¬U(σ) ∨ ¬S(σ) ∨ ¬X(σ), then σ is not an exfiltration attack.

### 2.3 Proof

We prove each condition is necessary by showing that its negation precludes exfiltration.

**Claim 1: Exfiltration(σ) ⟹ S(σ).**

Proof. By Definition 9, exfiltration requires that sensitive data s ∈ S is read and subsequently transmitted. If ¬S(σ), then no tool in σ reads any s ∈ S. Therefore no sensitive data enters the execution context, and no sensitive data can be transmitted to any external endpoint. Hence ¬Exfiltration(σ). By contrapositive, Exfiltration(σ) ⟹ S(σ). ∎

**Claim 2: Exfiltration(σ) ⟹ X(σ).**

Proof. By Definition 9, exfiltration requires that s reaches an external endpoint e ∈ Ext. If ¬X(σ), then no tool in σ transmits data to any external endpoint. The system boundary is not breached, so s remains internal. Hence ¬Exfiltration(σ). By contrapositive, Exfiltration(σ) ⟹ X(σ). ∎

**Claim 3: Exfiltration(σ) ⟹ U(σ).**

Proof. If ¬U(σ), then no untrusted external data influenced L's generation of σ. The sequence σ was generated entirely from the operator's system prompt, conversation history of trusted interactions, and the agent's own prior outputs. In this case, σ reflects the operator's intent (possibly mediated through the LLM's interpretation, but without adversarial influence). By definition, an action that faithfully reflects the operator's intent — even if it involves reading sensitive data and sending it externally — is an authorized operation, not an attack. Therefore ¬Exfiltration(σ) as an attack. By contrapositive, Exfiltration(σ) ⟹ U(σ). ∎

**Combining Claims 1-3:** Exfiltration(σ) ⟹ U(σ) ∧ S(σ) ∧ X(σ). ∎

### 2.4 Corollary: Rule of Two

**Corollary (Rule of Two).** Let σ be a tool invocation sequence within an event batch. If at most two of {U(σ), S(σ), X(σ)} are true, then σ is not an exfiltration attack.

Proof. By contrapositive of Theorem 2: Exfiltration(σ) requires U(σ) ∧ S(σ) ∧ X(σ), which requires all three predicates to be true. If at most two are true, then ¬(U(σ) ∧ S(σ) ∧ X(σ)), therefore ¬Exfiltration(σ). ∎

### 2.5 Application to CapabilityToken

The CapabilityToken enforces the Rule of Two at the authorization level. During token issuance, the CapabilityIssuer evaluates:

- U_token: whether the granted tools include untrusted-input sources (web_fetch, web_search)
- S_token: whether the granted tools include sensitive data readers (read_file with sensitive path patterns)
- X_token: whether the granted tools include external action tools (send_email, curl, bash)

If U_token ∧ S_token ∧ X_token, the issuer removes external action tools from the grant. This ensures:

|{U_token, S_token, X_token} ∩ {true}| ≤ 2

By the Corollary, no exfiltration attack can be completed within this token's scope.

### 2.6 Scope Limitation

Theorem 2 and its corollary hold within the scope of a **single event batch**. The conditions U, S, X are evaluated per-batch. Cross-batch context contamination — where untrusted data from a prior batch persists in session history and satisfies U for a subsequent batch without being recognized as such — falls outside the theorem's scope. See Section 7.3 of the main text for discussion of this limitation.

Additionally, Claim 3 relies on the definition of "attack" as requiring adversarial influence. An authorized but unwise operation (the operator intentionally reads sensitive data and sends it externally) is not classified as an attack under this model. The Rule of Two protects against adversarial subversion of agent behavior, not against operator error.
