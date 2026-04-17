# Supplementary Notes: Formal Security Analysis

## 1. Model and Definitions

**Definition 1 (Agent System).** An agent system is a tuple $\mathcal{M} = (\mathcal{S}, s_0, \mathcal{T}, L, P)$ where:
- $\mathcal{S}$ is the set of states (encoding LLM context, session history, and tool results)
- $s_0 \in \mathcal{S}$ is the initial state
- $\mathcal{T} = \{t_1, \ldots, t_n\}$ is a finite set of $n$ tools
- $L: \mathcal{S} \to \Delta\bigl((\mathcal{T} \times \text{Args}) \cup \{\bot\}\bigr)$ is the LLM planning function (stochastic), producing a tool invocation or $\bot$ (no action)
- $P: \mathcal{S} \to \mathcal{P}(\mathcal{T})$ is the permission model assigning authorized tools to each state

**Definition 2 (Tool Classification).** The tool set $\mathcal{T}$ is classified into three (possibly overlapping) subsets:
- $\mathcal{T}_R = \{t \in \mathcal{T} \mid \text{reads\_sensitive}(t)\}$: tools that can read sensitive data, $|\mathcal{T}_R| = n_R$
- $\mathcal{T}_X = \{t \in \mathcal{T} \mid \text{sends\_external}(t)\}$: tools that can transmit data externally, $|\mathcal{T}_X| = n_X$
- $\mathcal{T}_U = \{t \in \mathcal{T} \mid \text{receives\_untrusted}(t)\}$: tools that introduce untrusted data, $|\mathcal{T}_U| = n_U$

**Definition 3 (CapabilityToken).** For each event batch $b$, a CapabilityToken $\tau_b$ specifies:
- $\tau_b.\text{tools} \subseteq \mathcal{T}$ with $|\tau_b.\text{tools}| = k_b$ (the granted tool set for this batch)
- $\tau_b$ is constructed from event metadata by a deterministic CapabilityIssuer, independent of $L$

Let $k = \max_b k_b$ denote the maximum token size across all batches.

**Assumption 1 (Worst-Case Token Model).** For the purpose of bounding attack probability, we model the token as selecting $k$ tools uniformly at random from $\mathcal{T}$ without replacement. This is a worst-case model: the actual CapabilityIssuer selects tools deterministically based on event type, typically granting fewer tools and avoiding dangerous combinations. Any bound derived under the uniform model is therefore a valid upper bound on the actual system.

**Definition 4 (Batch Events).** For batch $b$, define the following events:
- $A = \{\text{injection succeeds}\}$: adversarial data in the batch causally influences $L$'s output. $\Pr[A] = p$.
- $B_R = \{\tau_b.\text{tools} \cap \mathcal{T}_R \neq \emptyset\}$: at least one sensitive-read tool is in the token.
- $B_X = \{\tau_b.\text{tools} \cap \mathcal{T}_X \neq \emptyset\}$: at least one external-send tool is in the token.
- $C = \{\neg R_2(\tau_b)\}$: the Rule of Two does not block the attack chain. When $R_2$ is enforced ($R_2(\tau_b) = 1$), the token construction guarantees that if both $B_R$ and $B_X$ hold, then $\tau_b.\text{tools} \cap \mathcal{T}_U = \emptyset$. Therefore $\Pr[C \mid B_R, B_X, R_2 = 1] = 0$.
- $D = \{\text{taint tracking evaded}\}$: both TaintStore and CausalTaintTracker fail to block the data flow. $\Pr[D \mid A, B_R, B_X, C] = q$.
- $\text{Exfil}(b) = A \cap B_R \cap B_X \cap C \cap D$.

**Definition 5 (Taint Lattice).** $(\Lambda, \leq)$ with $\Lambda = \{\text{USER}, \text{INTERNAL}, \text{EXTERNAL}\}$, $\text{USER} < \text{INTERNAL} < \text{EXTERNAL}$, $\bot = \text{USER}$, $\top = \text{EXTERNAL}$.

**Definition 6 (CausalTaintTracker).** For a batch with $m$ tool executions, the causal taint sequence is:

$$\tau_0 = \bot, \qquad \tau_k = \max(\tau_{k-1},\, \text{taint}(t_k)) \quad \text{for } k = 1, \ldots, m$$

---

## 2. Theorem 1: Single-Batch Attack Probability Bound

**Theorem 1 (Multi-Barrier Exfiltration Bound).** In an agent system with $n$ tools where each event batch is authorized a token of at most $k$ tools, the probability that an adversary successfully exfiltrates sensitive data through a single batch is bounded by:

$$\Pr[\text{Exfil}(b)] \leq p \cdot \min\!\left(\frac{kn_R}{n},\, \frac{kn_X}{n}\right) \cdot (1 - R_2) \cdot q$$

where $p$ is the prompt injection success rate, $kn_R/n$ and $kn_X/n$ are union-bound upper bounds on the probability that the token contains a sensitive-read or external-send tool respectively (the $\min$ reflects that exfiltration requires both), $R_2 \in \{0,1\}$ indicates whether the Rule of Two is enforced ($R_2 = 1$ zeroes the entire bound), and $q$ is the cross-batch taint evasion probability.

**Proof.**

$$\Pr[\text{Exfil}(b)] = \Pr[A \cap B_R \cap B_X \cap C \cap D] \tag{Def.~4}$$

$$= \Pr[A] \cdot \Pr[B_R \cap B_X \mid A] \cdot \Pr[C \mid A, B_R, B_X] \cdot \Pr[D \mid A, B_R, B_X, C] \tag{chain rule}$$

We simplify each factor.

$\Pr[A] = p$ by Definition 4.

For $\Pr[B_R \cap B_X \mid A]$: $B_R$ and $B_X$ are deterministic functions of $\tau_b$. By Definition 3, $\tau_b = \text{CapabilityIssuer}(\text{event\_metadata})$, which does not depend on $L$. Since $\tau_b \perp L$, any deterministic function of $\tau_b$ is independent of any event determined by $L$. Therefore $B_R \perp A$ and $B_X \perp A$, giving:

$$\Pr[B_R \cap B_X \mid A] = \Pr[B_R \cap B_X] \tag{i}$$

For $\Pr[C \mid A, B_R, B_X]$: $C = \{\neg R_2(\tau_b)\}$ is also a deterministic function of $\tau_b$, so $C \perp A$. When $R_2 = 1$ and $B_R \cap B_X$ holds, the token construction guarantees $\tau_b.\text{tools} \cap \mathcal{T}_U = \emptyset$, so $C$ cannot occur:

$$R_2 = 1 \;\wedge\; B_R \;\wedge\; B_X \;\Longrightarrow\; \tau_b.\text{tools} \cap \mathcal{T}_U = \emptyset \;\Longrightarrow\; \neg C$$

$$\Longrightarrow\; \Pr[C \mid B_R, B_X, R_2 = 1] = 0$$

When $R_2 = 0$, no such guarantee exists, so $\Pr[C \mid B_R, B_X, R_2 = 0] \leq 1$. Combining:

$$\Pr[C \mid B_R, B_X] \leq 1 - R_2 \tag{ii}$$

$\Pr[D \mid A, B_R, B_X, C] = q$ by Definition 4.

Substituting into the chain rule:

$$\Pr[\text{Exfil}(b)] \leq p \cdot \Pr[B_R \cap B_X] \cdot (1 - R_2) \cdot q \tag{iii}$$

We now bound $\Pr[B_R \cap B_X]$. Under Assumption 1, the token selects $k$ tools uniformly from $\mathcal{T}$ without replacement. Let $Z_R = |\tau_b.\text{tools} \cap \mathcal{T}_R|$. Then $B_R = \{Z_R \geq 1\}$:

$$\Pr[\neg B_R] = \Pr[Z_R = 0] = \frac{\binom{n - n_R}{k}}{\binom{n}{k}} = \prod_{i=0}^{k-1} \frac{n - n_R - i}{n - i} \tag{hypergeometric}$$

For the upper bound on $\Pr[B_R]$, we use the union bound:

$$\Pr[B_R] = \Pr\!\left[\exists\, t \in \mathcal{T}_R : t \in \tau_b.\text{tools}\right] \leq \sum_{t \in \mathcal{T}_R} \Pr[t \in \tau_b.\text{tools}] = n_R \cdot \frac{k}{n} = \frac{kn_R}{n} =: \alpha_R \tag{iv}$$

By identical argument replacing $n_R$ with $n_X$:

$$\Pr[B_X] \leq \frac{kn_X}{n} =: \alpha_X \tag{v}$$

For $\Pr[B_R \cap B_X]$, we use the trivial bound:

$$\Pr[B_R \cap B_X] \leq \min(\Pr[B_R],\, \Pr[B_X]) \leq \min(\alpha_R,\, \alpha_X) \tag{vi}$$

Substituting (iii), (iv), (v), (vi):

$$\Pr[\text{Exfil}(b)] \leq p \cdot \Pr[B_R \cap B_X] \cdot (1 - R_2) \cdot q \leq p \cdot \min(\alpha_R, \alpha_X) \cdot (1 - R_2) \cdot q$$

$$= p \cdot \min\!\left(\frac{kn_R}{n},\, \frac{kn_X}{n}\right) \cdot (1 - R_2) \cdot q \qquad \blacksquare$$

---

## 3. Theorem 2: Session-Level Compound Bound

**Theorem 2 (Session-Level Security Degradation).** Over a session of $B$ batches with no cross-batch state dependence (see Remark below), let $\varepsilon = p \cdot \min(\alpha_R, \alpha_X) \cdot (1-R_2) \cdot q$. Then:

$$\Pr[\exists\, b : \text{Exfil}(b)] \leq 1 - (1 - \varepsilon)^B$$

**Proof.**

$$\Pr[\exists\, b : \text{Exfil}(b)] = 1 - \Pr[\neg\text{Exfil}(1) \cap \cdots \cap \neg\text{Exfil}(B)] \tag{complement}$$

$$= 1 - \prod_{b=1}^{B} \Pr[\neg\text{Exfil}(b)] \tag{independence}$$

$$= 1 - \prod_{b=1}^{B} \bigl(1 - \Pr[\text{Exfil}(b)]\bigr)$$

Since $\Pr[\text{Exfil}(b)] \leq \varepsilon$ for each $b$ (Theorem 1), we have $1 - \Pr[\text{Exfil}(b)] \geq 1 - \varepsilon$, so:

$$\prod_{b=1}^{B} \bigl(1 - \Pr[\text{Exfil}(b)]\bigr) \geq \prod_{b=1}^{B} (1 - \varepsilon) = (1 - \varepsilon)^B$$

Therefore:

$$\Pr[\exists\, b : \text{Exfil}(b)] \leq 1 - (1 - \varepsilon)^B \qquad \blacksquare$$

**Remark (Independence Assumption).** The product form assumes batch independence. If session history introduces dependence, a weaker union bound applies: $\Pr[\exists\, b : \text{Exfil}(b)] \leq B\varepsilon$, valid without independence.

**Proposition 2.1 (Ambient Authority Baseline).** Under ambient authority, $\Pr_{\text{amb}}[\text{Exfil}(b)] \geq p$ (all tools available, no structural constraint). Therefore:

$$\Pr_{\text{amb}}[\exists\, b : \text{Exfil}(b)] \geq 1 - (1-p)^B \xrightarrow{B \to \infty} 1 \qquad \blacksquare$$

---

## 4. Theorem 3: Rule of Two Hard Guarantee

**Theorem 3 (Rule of Two: Zero-Probability Guarantee).** When $R_2 = 1$:

$$\Pr[\text{Exfil}(b)] = 0$$

**Proof.**

$$\Pr[\text{Exfil}(b)] \leq p \cdot \min(\alpha_R, \alpha_X) \cdot (1 - R_2) \cdot q = p \cdot \min(\alpha_R, \alpha_X) \cdot 0 \cdot q = 0$$

Since $\Pr[\text{Exfil}(b)] \geq 0$ (probability axiom) and $\Pr[\text{Exfil}(b)] \leq 0$ (above): $\Pr[\text{Exfil}(b)] = 0$. $\blacksquare$

**Corollary 3.1.** If $R_2 = 1$ for all $B$ batches: $\Pr[\exists\, b : \text{Exfil}(b)] \leq 1 - (1 - 0)^B = 0$. $\blacksquare$

---

## 5. Theorem 4: CausalTaintTracker Monotonicity

**Theorem 4 (Causal Taint Irrevocability, within a single batch).** Fix an event batch $b$ with reasoning steps indexed $0, 1, \ldots, m$, and let $(\tau_0, \tau_1, \ldots, \tau_m)$ be the within-batch causal taint sequence produced by $\tau_k = \max(\tau_{k-1}, \text{taint}(t_k))$. Then $\forall\, 0 \leq i \leq j \leq m$: $\tau_i \leq \tau_j$. The statement is restricted to a single batch; cross-batch residuals are not covered by this monotonicity property.

**Proof.** For any $k \in \{1, \ldots, m\}$:

$$\tau_k = \max(\tau_{k-1}, \text{taint}(t_k)) \geq \tau_{k-1} \qquad (\max(a,b) \geq a)$$

Chaining at $k = i+1, i+2, \ldots, j$:

$$\tau_i \leq \tau_{i+1} \leq \tau_{i+2} \leq \cdots \leq \tau_j$$

By transitivity of $\leq$ on $\Lambda$: $\tau_i \leq \tau_j$. $\blacksquare$

**Corollary 4.1 (Irrevocability).** If $\tau_{k_0} = \top$ for some $k_0$, then $\forall\, k \geq k_0$: $\tau_k = \top$.

**Proof.**

$$\top \leq \tau_k \leq \top \qquad (\text{Theorem 4 and } \top = \max \Lambda)$$

By antisymmetry: $\tau_k = \top$. $\blacksquare$

**Proposition 4.2 (Within-Batch Evasion).** If $\text{taint}(t_j) = \top$ at step $j$, then $q_{\text{within}} = 0$.

**Proof.** Let $k > j$. Then $k - 1 \geq j$.

$$\tau_{k-1} \geq \tau_j = \max(\tau_{j-1}, \text{taint}(t_j)) \geq \text{taint}(t_j) = \top$$

So $\tau_{k-1} = \top$ (Corollary 4.1). For any tool $t$ with $\max\_\text{causal}(t) = \text{INTERNAL} < \top$:

$$\tau_{k-1} = \top > \text{INTERNAL} = \max\_\text{causal}(t)$$

Therefore $t$ is causally blocked at step $k$. This holds $\forall\, k > j$ and $\forall\, t \in \mathcal{T}_X$. No external transmission can occur after step $j$.

$$\therefore\; q_{\text{within}} = 0 \qquad \blacksquare$$

**Remark.** $q$ in Theorem 1 decomposes as $q = q_{\text{within}} + q_{\text{cross}} - q_{\text{within}} \cdot q_{\text{cross}}$. Since $q_{\text{within}} = 0$: $q = q_{\text{cross}}$.

---

## 6. Theorem 5: Gate Determinism and Bound Integrity

**Theorem 5 (Bound Integrity).** The CapabilityGate function $G: \mathcal{A} \times \text{Token} \times \text{TaintStore} \times \text{Counter} \times \mathbb{R}_{\geq 0} \to \{\text{ALLOW}, \text{DENY}\}$ is deterministic and LLM-independent.

**Proof (Determinism).** $G = \text{Check}_3 \circ \text{Check}_2 \circ \text{Check}_1$. Each check uses only:

$$\text{Check}_1: \{>,\, \in,\, \in\} \qquad \text{Check}_2: \{\text{SHA-256},\, \text{dict.lookup},\, >\} \qquad \text{Check}_3: \{\text{fnmatch},\, \geq,\, \text{boolean}\}$$

Let $x = (a, \tau, S, C, t_{\text{now}}) = x'$. Then for each check, equal inputs to deterministic operations yield equal outputs:

$$\text{Check}_1(x) = \text{Check}_1(x') \;\Longrightarrow\; \text{Check}_2(x) = \text{Check}_2(x') \;\Longrightarrow\; \text{Check}_3(x) = \text{Check}_3(x')$$

$$\therefore\; G(x) = G(x') \qquad \blacksquare$$

**Proof (LLM-Independence).** $G$'s inputs: $a = (t, \text{args})$ ← output of $L$, but $G$ does not invoke $L$ to evaluate $a$; $\tau$ ← $\text{CapabilityIssuer}(\text{event\_metadata})$, $L$ not involved; $S$ ← framework registers tool returns, $L$ not involved; $C$ ← framework counter; $t_{\text{now}}$ ← wall clock.

$$\{\text{operations in } G\} \cap \{\text{operations invoking } L\} = \emptyset$$

$$\therefore\; G \text{ is LLM-independent} \qquad \blacksquare$$

**Corollary 5.1 (Bound Integrity).** $\alpha_R = kn_R/n$, $\alpha_X = kn_X/n$, and $R_2$ are all functions of $\tau_b$ (not of $L$). Since $\tau_b = \text{CapabilityIssuer}(\text{event\_metadata})$ and $G$ is LLM-independent (Theorem 5), the adversary can influence $p$ (injection success) and $q$ (taint evasion) but cannot manipulate $\alpha_R$, $\alpha_X$, or $R_2$. $\blacksquare$

---

## 7. Concrete Instantiation

Parameters: $n = 55$, $k = 5$, $n_R = 8$, $n_X = 6$, $p = 0.5$, $q = 0.1$, $R_2 \in \{0, 1\}$, $B = 288$.

**Case 1:** $R_2 = 1$. By Theorem 3: $\Pr[\text{Exfil}(b)] = 0$.

**Case 2:** $R_2 = 0$ (worst case).

$$\alpha_R = \frac{kn_R}{n} = \frac{5 \cdot 8}{55} = \frac{40}{55} \approx 0.727, \qquad \alpha_X = \frac{kn_X}{n} = \frac{5 \cdot 6}{55} = \frac{30}{55} \approx 0.545$$

$$\min(\alpha_R, \alpha_X) = 0.545$$

$$\Pr[\text{Exfil}(b)] \leq 0.5 \times 0.545 \times 1.0 \times 0.1 = 0.0273$$

Exact hypergeometric values (for comparison):

$$\Pr[B_R] = 1 - \frac{\binom{47}{5}}{\binom{55}{5}} \approx 0.568, \qquad \Pr[B_X] = 1 - \frac{\binom{49}{5}}{\binom{55}{5}} \approx 0.452$$

The union bounds $\alpha_R = 0.727$ and $\alpha_X = 0.545$ are conservative (true values are 0.568 and 0.452), confirming the bound is valid but not tight.

Session level (union bound): $\Pr[\exists\, b : \text{Exfil}(b)] \leq B\varepsilon = 288 \times 0.0273 = 7.85$, capped at 1.

Under independence: $\Pr[\exists\, b : \text{Exfil}(b)] \leq 1 - (1 - 0.0273)^{288} \approx 0.9997$.

**Comparison with ambient authority:**

| Metric | Ambient authority | Event-scoped ($R_2=0$) | Event-scoped ($R_2=1$) |
|--------|-------------------|------------------------|------------------------|
| $\Pr[\text{Exfil}(b)]$ | $\geq 0.5$ | $\leq 0.0273$ | $= 0$ |
| Per-batch reduction | — | $\approx 18\times$ | $\infty$ |
| Session ($B=288$) | $\approx 1.0$ | $\leq 0.9997$ | $= 0$ |
