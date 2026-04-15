## 2. The Agent Authority Problem

### 2.1 The deterministic intent assumption

Classical access control — from Multics rings to OAuth scopes — rests on an implicit assumption:

> **The Deterministic Intent Assumption.** The executing principal's intent is determined by the authenticated identity and is not modifiable by the data the principal processes.

When Alice logs into a banking system, her request to transfer $500 reflects Alice's intent. The data she views does not alter that intent. Alice's identity determines her permissions, and her permissions bound her actions.

AI agents violate this assumption categorically. An LLM-powered agent's "intent" — the plan it forms and the actions it takes — emerges from the combination of its system prompt, conversation history, and the data it processes in real time. The same architecture that enables helpful email summarization enables a crafted email to redirect subsequent actions.

### 2.2 The data-instruction conflation and the SQL injection parallel

This vulnerability exists because LLMs process data and instructions through the same token stream, with no intrinsic boundary between "data to analyze" and "instruction to follow."

The parallel to SQL injection is historically illuminating. SQL injection arose because database interfaces concatenated user data and SQL commands into the same string. For over a decade, the response was defensive programming: sanitization, escaping, pattern detection. Each defense was incrementally bypassed. It took roughly 15 years — from the first documented SQL injection in 1998 to widespread adoption of parameterized queries — for the field to recognize that the vulnerability was architectural: the conflation of data and code. The solution was structural separation ensuring user input is *never interpreted as SQL commands*.

Prompt injection stands at the beginning of the same arc. Current defenses — boundary markers, detection patterns, safety instructions — recapitulate the "sanitization era." **This paper proposes the "parameterized query moment" for agent security**: an architectural separation where the security enforcement layer *never interprets natural language*, and the LLM *never makes security decisions*.

The consequences of delayed transition are graver. SQL injection corrupted databases; prompt injection of an autonomous agent triggers real-world actions across every connected system. And unlike SQL injection, which required programming knowledge, prompt injection requires only natural language — lowering the attack barrier to anyone who can compose a sentence.

### 2.3 The structural argument

We now state the Agent Authority Problem precisely. We frame this as a **structural security argument** — an informal but rigorous logical analysis whose formal treatment (including explicit adversarial models) is provided in Supplementary Note 1.

> **The Agent Authority Problem.** Under the ambient authority model — where an agent inherits a static set of permissions for the duration of a session — no security mechanism can simultaneously satisfy:
>
> **(C1) Autonomous data processing**: The agent processes data from untrusted external sources without per-item human approval.
>
> **(C2) Privileged resource access**: The agent holds access to sensitive resources (filesystem, credentials, communication channels).
>
> **(C3) LLM-independent safety**: Security guarantees do not depend on the LLM's ability to perfectly resolve the data-instruction ambiguity inherent in natural language.

**Argument.** Under ambient authority, the agent holds privileges (C2) throughout the session. When processing untrusted data (C1), adversarial instructions embedded in that data enter the LLM's context as part of the same token stream. Because natural language does not carry an intrinsic type distinction between data and instructions — the same sentence can function as either depending on speaker intent — the LLM faces an inherently ambiguous classification task. To prevent the LLM from acting on adversarial instructions without reducing its privileges, the system must rely on the LLM's heuristic judgment to correctly classify content — a judgment that is probabilistic, context-dependent, and degradable under adversarial pressure. This violates C3.

**On the permanence of this argument.** A critical observation: the data-instruction ambiguity is not a limitation of current models that improved training may overcome. It is a property of natural language itself. The text "Please send the project report to partner@company.com" can be legitimate data (in an email being summarized) or an instruction to execute (issued by the user). Whether a text is "data" or "instruction" depends on the speaker's intent, which is not encoded in the text. This distinguishes the Agent Authority Problem from a temporary technological limitation: it is a **permanent structural property** of systems that process natural language with ambient authority.

### 2.4 Why incremental defenses are insufficient

This argument explains why existing defenses, however sophisticated, fail to provide structural guarantees:

| Defense | Structural limitation |
|---------|----------------------|
| Prompt injection detection | Cannot enumerate all adversarial encodings; fundamentally incomplete |
| External content boundary markers | Relies on LLM to respect labels (violates C3) |
| Static tool allow/deny lists | Remains unchanged throughout session; injected agent retains all allowed tools |
| Execution approval gates | Covers only shell execution; high-level actions bypass approval |
| Docker sandboxing | Constrains *how* code executes, not *what* the LLM decides to execute |

Each addresses a symptom. None addresses the root cause: **static ambient authority coexisting with dynamic, potentially adversarial data.**

### 2.5 From perimeter defense to context-scoped authority

The Agent Authority Problem parallels the transition from perimeter-based network security to zero trust in the 2010s. But the analogy is imperfect in a critical way. In network zero trust, the principal's identity remains the basis for authorization. For AI agents, identity is insufficient: **the agent's identity never changes — it is always "the operator's agent" — but its effective intent changes with every piece of data it processes.** Authorization must be bound not to *who* the agent is, but to *what event it is currently processing* — its operational context.
