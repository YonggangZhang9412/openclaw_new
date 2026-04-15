# Supplementary Note 5: CapabilityGate Determinism and LLM-Independence

## Definitions

**Definition 19 (Deterministic Function).** A function f: X → Y is deterministic if:

∀ x₁, x₂ ∈ X: x₁ = x₂ ⟹ f(x₁) = f(x₂)     ... (det)

**Definition 20 (LLM-Independent Function).** A function f is LLM-independent with respect to agent system ℳ = (𝒮, s₀, 𝒯, 𝒜, δ, L, E) if f's computation does not invoke L and does not read any component of 𝒮 exclusively determined by L's prior outputs.

**Definition 21 (Primitive Operations).** We call an operation *primitive-deterministic* if it is one of: real-number comparison (<, >, ≤, ≥, =), set membership test (∈ on a finite set), cryptographic hash (SHA-256), dictionary lookup (key → value), glob pattern matching (fnmatch), integer arithmetic (+, −, ×, comparison), or boolean operations (∧, ∨, ¬).

**Fact (Closure).** The composition of finitely many primitive-deterministic operations through a fixed control flow (sequential execution, if-then-else with deterministic branch condition) is deterministic. This follows from (det) applied inductively: if each step is deterministic and the control flow is determined by deterministic branch conditions, then the composite function is deterministic.

**Definition 22 (CapabilityGate Function).** The CapabilityGate function G is:

G: 𝒜 × Token × TaintStore × CallCounter × ℝ≥₀ → {ALLOW, DENY} × Reason

For input (a, τ, S, C, t_now) where a = (t, args):

```
G(a, τ, S, C, t_now) =
  let r₁ = Check₁(t, τ, t_now) in
  if r₁ ∈ DENY then return r₁
  else let r₂ = Check₂(t, args, S) in
  if r₂ ∈ DENY then return r₂
  else return Check₃(t, args, τ, C)
```

**Check₁(t, τ, t_now):**
```
  if t_now > τ.issued_at + τ.ttl:       return DENY("token_expired")
  if t ∉ τ.granted_tools:                return DENY("tool_not_granted")
  if t ∈ τ.denied_tools:                 return DENY("tool_denied")
  return CONTINUE
```
Operations: real comparison (>), set membership (∈) ×2. All primitive-deterministic.

**Check₂(t, args, S):**
```
  for each (pname, pvalue) ∈ args:
    tl ← S.query(pvalue)                // SHA-256 + dict lookup + string match
    if tl ≠ None:
      let max_t ← policy(t, pname).max_taint
      if tl > max_t:                     return DENY("taint_violation")
  return CONTINUE
```
Operations per iteration: SHA-256 hash (deterministic), dictionary lookup, string matching, integer comparison (taint levels are integers). Loop iterates |args| times (finite, bounded by max_params). All primitive-deterministic.

**Check₃(t, args, τ, C):**
```
  if policy(t).requires_path:
    if ¬glob_match(args.path, τ.granted_paths): return DENY("path_denied")
  if C[t] ≥ max_calls_per_round:                 return DENY("frequency_exceeded")
  if causal_rule_of_two_violated(t, τ):           return DENY("rule_of_two")
  return ALLOW
```
Operations: glob matching (deterministic string algorithm), dictionary lookup + integer comparison, boolean evaluation. All primitive-deterministic.

## Lemma 2: Totality

**Lemma 2.** G is a total function: for every valid input (a, τ, S, C, t_now), G terminates and produces an output in {ALLOW, DENY} × Reason.

**Proof.** G has a fixed sequential structure: Check₁ → Check₂ → Check₃.

Check₁ executes exactly 3 comparisons. Terminates in O(1).     ... (t1)

Check₂ iterates over args. Let p = |args|. Each iteration performs:
- S.query: computes SHA-256 (O(|pvalue|)), looks up in dictionary of size |S| ≤ 10,000 (O(1) amortized hash table lookup), performs bounded substring scan. Each step terminates.
- One integer comparison.
Total: O(p · |pvalue_max|) steps, all finite.     ... (t2)

Check₃ performs at most 3 operations (glob match, counter lookup, boolean). Each terminates.     ... (t3)

G's control flow is a fixed sequence of at most 3 checks, each terminating by (t1)-(t3). No check invokes recursion or unbounded loops. Therefore G terminates for all valid inputs.

Every execution path reaches exactly one of the DENY or ALLOW return statements in Check₁, Check₂, or Check₃. Therefore G produces an output in {ALLOW, DENY} × Reason. ∎

## Theorem 4: Gate Determinism

**Theorem 4(a) (Determinism).** G is deterministic: for all valid inputs x₁ = x₂, G(x₁) = G(x₂).

**Proof.** Let x = (a, τ, S, C, t_now) and x' = (a', τ', S', C', t_now') with x = x'. Then a = a', τ = τ', S = S', C = C', t_now = t_now'.

Check₁ computes:
- t_now > τ.issued_at + τ.ttl: real comparison of equal inputs → equal output     ... (d1)
- t ∈ τ.granted_tools: set membership of equal element in equal set → equal output     ... (d2)
- t ∈ τ.denied_tools: same reasoning as (d2)     ... (d3)

If Check₁ returns DENY at any point, it returns the same DENY for both inputs (by d1-d3). If Check₁ returns CONTINUE, both inputs proceed to Check₂.

Check₂ iterates over args = args' (equal by assumption). For each (pname, pvalue):
- S.query(pvalue) = S'.query(pvalue'): S = S' and pvalue = pvalue', so SHA-256(pvalue) = SHA-256(pvalue') (SHA-256 is deterministic), dictionary lookup in equal dictionaries with equal keys returns equal results     ... (d4)
- tl > max_t: integer comparison of equal values → equal output     ... (d5)

Check₂ returns the same result for both inputs (by d4-d5 applied to each iteration).

Check₃ by identical reasoning:
- glob_match(args.path, τ.granted_paths) = glob_match(args'.path, τ'.granted_paths) (equal inputs to deterministic algorithm)     ... (d6)
- C[t] = C'[t'] (equal counters, equal keys)     ... (d7)
- causal_rule_of_two_violated(t, τ) = causal_rule_of_two_violated(t', τ') (equal inputs to boolean function)     ... (d8)

Therefore G(x) = G(x'). By Definition 19, G is deterministic. ∎

## Theorem 4(b): LLM-Independence

**Theorem 4(b) (LLM-Independence).** G is LLM-independent.

**Proof.** We verify that G's computation never invokes L and never reads L-exclusive state, by inspecting each input component of G:

| Input | Source | Involves L? |
|-------|--------|-------------|
| a = (t, args) | Proposed by L | No: G receives a as input; it does not invoke L to obtain or validate a |
| τ (Token) | Created by CapabilityIssuer from event metadata | No: CapabilityIssuer reads event.source, event.payload — not L's state |
| S (TaintStore) | Populated by tool return value registration | No: S.register is called by the execution framework after tool execution, not by L |
| C (CallCounter) | Incremented by execution framework | No: framework bookkeeping, not L-dependent |
| t_now | Wall-clock time | No: physical time, independent of L |

G's internal computation uses only primitive-deterministic operations (Definition 21) on these inputs. None of these operations invokes L or reads L's internal state (weights, activations, attention patterns, token probabilities, or reasoning trace).

Note: a = (t, args) is *generated by* L — the LLM chose the tool and arguments. But G does not invoke L to process a; it evaluates a against static policies and data provenance records using only the operations listed in Definition 22. The distinction is between "using L's output as input data" (which any enforcement function must do) and "invoking L as part of the enforcement computation" (which G does not do). ∎

## Security Implication

**Corollary (Prompt Injection Immunity).** No adversarial input processed by L can alter G's decision on a given (a, τ, S, C, t_now).

**Proof.** By Theorem 4(b), G does not invoke L. By Theorem 4(a), G's output is determined entirely by its input tuple (a, τ, S, C, t_now). An adversary who manipulates L can alter which a the LLM proposes, but cannot alter how G evaluates any specific a — because G's evaluation depends only on (a, τ, S, C, t_now), none of which are modifiable by the adversary through the LLM's token stream after they are fixed as inputs to G.

Formally: let Adv be any adversary that can influence L's output distribution. For any fixed (τ, S, C, t_now), define D(a) = G(a, τ, S, C, t_now). Then D is a deterministic function (by Theorem 4a) that Adv cannot modify (by Theorem 4b). Adv can choose the distribution over a that L produces, but D(a) is fixed for each a. In particular, if D(a) = DENY for some a, no adversarial manipulation of L can change this to ALLOW. ∎
