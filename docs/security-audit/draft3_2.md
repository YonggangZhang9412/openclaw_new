## 2. The Agent Authority Problem

### 2.1 The deterministic intent assumption

Classical access control — from Multics rings to OAuth scopes — rests on an implicit assumption:

> **The Deterministic Intent Assumption.** The executing principal's intent is determined by the authenticated identity and is not modifiable by the data the principal processes.

When Alice logs into a banking system, her request to transfer \$500 reflects Alice's intent. The data she views does not alter that intent. AI agents violate this assumption categorically: an LLM-powered agent's "intent" emerges from its system prompt, conversation history, and the data it processes in real time. The same architecture that enables helpful email summarization enables a crafted email to redirect subsequent actions.

### 2.2 The data-instruction conflation and the SQL injection parallel

This vulnerability exists because LLMs process data and instructions through the same token stream, with no intrinsic boundary between "data to analyze" and "instruction to follow."

The parallel to SQL injection is historically illuminating. SQL injection arose because database interfaces concatenated user data and SQL commands into the same string. For over a decade, the response was defensive programming: sanitization, escaping, pattern detection. Each defense was incrementally bypassed. It took roughly 15 years — from the first documented SQL injection in 1998 to widespread adoption of parameterized queries — for the field to recognize that the vulnerability was architectural: the conflation of data and code. The solution was structural separation ensuring user input is *never interpreted as SQL commands*.

Prompt injection stands at the beginning of the same arc. Current defenses — boundary markers, detection patterns, safety instructions — recapitulate the "sanitization era." **This paper proposes the "parameterized query moment" for agent security**: an architectural separation where the security enforcement layer *never interprets natural language*, and the LLM *never makes security decisions*.

The consequences of delayed transition are graver. SQL injection corrupted databases; prompt injection of an autonomous agent triggers real-world actions across every connected system. And unlike SQL injection, which required programming knowledge, prompt injection requires only natural language — lowering the attack barrier to anyone who can compose a sentence.

### 2.3 The structural argument

We state the Agent Authority Problem precisely with formal treatment in Supplementary Note 1.

> **The Agent Authority Problem.** Under the ambient authority model — where an agent inherits a static set of permissions for the duration of a session — no security mechanism can simultaneously satisfy:
>
> **(C1) Autonomous data processing**: The agent processes data from untrusted external sources without per-item human approval.
>
> **(C2) Privileged resource access**: The agent holds access to sensitive resources (filesystem, credentials, communication channels).
>
> **(C3) LLM-independent safety**: Security guarantees do not depend on the LLM's ability to perfectly resolve the data-instruction ambiguity inherent in natural language.

**Argument.** Under ambient authority, the agent holds privileges (C2) throughout the session. When processing untrusted data (C1), adversarial instructions enter the LLM's processing context. If data and instructions share a processing channel — as they do in all current LLM architectures — the LLM faces an inherently ambiguous classification task. To prevent the LLM from acting on adversarial instructions without reducing its privileges, the system must rely on the LLM's heuristic judgment — violating C3.

**On scope.** The Agent Authority Problem is conditional on the dominant architecture of LLM-based agents — systems where data and instructions share a processing channel. This includes all current and foreseeable architectures where the agent must *reason about* external data. A hypothetical architecture with perfect channel separation would escape the impossibility but would itself constitute a form of the solution we propose. CaMeL¹² takes a step in this direction but still requires its Quarantined LLM to process untrusted data *and* generate tool-call decisions within the same component.

### 2.4 Why incremental defenses are insufficient

| Defense | Structural limitation |
|---------|----------------------|
| Prompt injection detection | Cannot enumerate all adversarial encodings; fundamentally incomplete |
| External content boundary markers | Relies on LLM to respect labels (violates C3) |
| Static tool allow/deny lists | Remains unchanged throughout session; injected agent retains all allowed tools |
| Execution approval gates | Covers only shell execution; high-level actions bypass approval |
| Docker sandboxing | Constrains *how* code executes, not *what* the LLM decides to execute |

### 2.5 From perimeter defense to context-scoped authority

The Agent Authority Problem parallels the transition from perimeter-based network security to zero trust. But the analogy is imperfect: in network zero trust, the principal's identity remains the basis for authorization. For AI agents, identity is insufficient — **the agent's identity never changes, but its effective intent changes with every piece of data it processes.** Authorization must be bound not to *who* the agent is, but to *what event it is currently processing*.
