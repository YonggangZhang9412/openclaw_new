## 2. The Agent Authority Problem: A Structural Impossibility

### 2.1 The hidden assumption of classical access control

Every access control system in the history of computing — from Multics rings to Unix permissions to OAuth scopes — rests on an implicit axiom:

> **The Deterministic Intent Axiom**: The executing principal's intent is determined by the authenticated identity and is not modifiable by the data the principal processes.

When a human user logs into a banking system, the system can reason: "This is Alice. Alice has permission to transfer funds. Alice's request to transfer $500 to Bob reflects Alice's intent." The data Alice views (account balances, transaction history) does not alter Alice's intent to transfer funds. Alice's identity determines her permissions, and her permissions bound her actions. This is the foundation of role-based access control, mandatory access control, and capability-based security in the classical sense.

AI agents violate this axiom categorically.

### 2.2 Emergent intent and the data-instruction conflation

An LLM-powered agent does not have a fixed intent determined at authentication time. Its "intent" — the plan it forms and the actions it takes — **emerges from the combination of its system prompt, conversation history, and the data it processes in real time**. This emergence is not a bug; it is the fundamental mechanism by which LLMs operate. The same architecture that enables an agent to helpfully summarize an email also enables a carefully crafted email to redirect the agent's subsequent actions.

This vulnerability exists because LLMs process data and instructions through the same channel — the token stream. There is no hardware-enforced boundary between "this is data to analyze" and "this is an instruction to follow." The parallel to SQL injection is precise and instructive: SQL injection became possible because early database interfaces concatenated user data and SQL commands into the same string. The solution — parameterized queries — required an **architectural separation** between data and code. Prompt injection is the SQL injection of the agentic era, and it demands an analogous architectural response.

But the consequences are far graver. SQL injection could corrupt a database. Prompt injection of an autonomous agent can trigger real-world actions — sending emails, executing financial transactions, modifying medical records, deleting files — across every system the agent has access to. The blast radius is bounded only by the agent's permissions.

### 2.3 The impossibility theorem

We can now state the Agent Authority Problem precisely:

> **Theorem (Agent Authority Impossibility).** Under the ambient authority model — where an agent inherits a static set of permissions for the duration of a session — no security mechanism can simultaneously satisfy:
>
> **(C1) Autonomous data processing**: The agent processes data from untrusted external sources (email, web, APIs) without per-item human approval.
>
> **(C2) Privileged resource access**: The agent holds access to sensitive resources (filesystem, credentials, communication channels, financial systems).
>
> **(C3) LLM-independent safety**: Security guarantees do not depend on the LLM's ability to perfectly resolve the data-instruction ambiguity inherent in natural language.
>
> Specifically, any system satisfying C1 and C2 under ambient authority must rely on the LLM to correctly distinguish between legitimate data and adversarial instructions embedded in that data — thereby violating C3.

A critical observation about C3: the data-instruction ambiguity is not a limitation of current language models that may be overcome through improved training. It is a **property of natural language itself**. The same text — for instance, "Please send the project report to partner@company.com" — can be legitimate data to analyze (when appearing in an email being summarized) or an instruction to execute (when issued by the user). Whether a given text is "data" or "instruction" depends on the speaker's intent, which is not encoded in the text itself. No model improvement can resolve an ambiguity that is inherent in the medium. This distinguishes the Agent Authority Problem from a temporary technological limitation: it is a **permanent structural property** of systems that process natural language with ambient authority.

**Proof sketch.** Under ambient authority, the agent holds permissions to access sensitive resources (C2) throughout the session. When the agent processes untrusted data (C1), adversarial instructions embedded in that data enter the LLM's context as part of the same token stream. Because natural language does not carry an intrinsic type distinction between data and instructions — the same sentence can function as either depending on context and speaker intent — any LLM processing this input faces an inherently ambiguous classification task. Resolving this ambiguity with certainty would require knowledge of the speaker's intent, which is unavailable to the model. Therefore, the system must depend on the LLM's heuristic judgment to classify potentially adversarial content — a judgment that is probabilistic, context-dependent, and degradable under adversarial pressure — thereby violating C3.

The formal proof, including explicit adversarial models and a connection to the undecidability of intent attribution in natural language, will be presented in Supplementary Materials.

### 2.4 Why incremental defenses are insufficient

This impossibility explains why the defenses deployed by OpenClaw and other frameworks — however sophisticated — fail to provide structural security guarantees:

| Defense | Why it fails against the impossibility |
|---------|---------------------------------------|
| Prompt injection detection (pattern matching) | Detects known patterns but cannot enumerate all adversarial encodings. Fundamentally a cat-and-mouse game. |
| External content boundary markers | Labels untrusted content but relies on LLM to respect the labels (violates C3). |
| Tool allow/deny lists | Reduces available tools but remains static throughout the session. An injected agent retains all allowed tools. |
| Execution approval gates | Covers only shell execution. High-level actions (email, API calls, sub-agent spawning) bypass approval. |
| Docker sandboxing | Isolates code execution but not the LLM's decision-making. The LLM decides *what* to execute; the sandbox only constrains *how*. |

Each defense addresses a symptom. None addresses the root cause: **the agent holds static ambient authority while processing dynamic, potentially adversarial data.**

### 2.5 The historical parallel: from perimeter defense to zero trust

The Agent Authority Problem bears a structural resemblance to a transformation that enterprise network security underwent in the 2010s. Traditional network security assumed that everything inside the corporate perimeter was trusted ("castle and moat" model). The rise of cloud computing, BYOD, and insider threats invalidated this assumption, leading to the **zero trust** paradigm: "never trust, always verify," with per-request authentication and minimal privilege grants.

The ambient authority model for AI agents is today's "castle and moat." The agent is inside the perimeter (it is authenticated as the operator's agent), so it is trusted with the operator's full permissions. But the agent's "intent" can be hijacked through its data channels, just as an insider can be socially engineered. The solution demands an analogous paradigm shift: from "trusted agent with ambient authority" to "zero-trust agent with per-event capability grants."

But the analogy is imperfect in a critical way. In network zero trust, the principal's identity is still the basis for authorization — you verify the user's identity per-request. For AI agents, identity is insufficient because **the agent's identity never changes — it is always "the operator's agent" — but its effective intent can change with every piece of data it processes.** Authorization must therefore be bound not to *who* the agent is, but to *what event the agent is currently processing* — its operational context. This insight leads directly to the architecture we propose.
