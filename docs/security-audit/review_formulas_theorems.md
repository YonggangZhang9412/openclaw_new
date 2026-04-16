# Formula Review: Supplementary Notes (theorems_v2_1.md)

## Definitions (Section 1)

**Line 5 — Definition 1:**
`$\mathcal{M} = (\mathcal{S}, s_0, \mathcal{T}, L, P)$`
✅ CORRECT. All calligraphic letters properly formatted.

**Line 7:**
`$s_0 \in \mathcal{S}$`
✅ CORRECT.

**Line 8:**
`$\mathcal{T} = \{t_1, \ldots, t_n\}$`
✅ CORRECT.

**Line 9:**
`$L: \mathcal{S} \to \Delta(\mathcal{T} \times \text{Args} \cup \{\bot\})$`
⚠️ ISSUE. Operator precedence ambiguous. Is it $\Delta((\mathcal{T} \times \text{Args}) \cup \{\bot\})$ or $\Delta(\mathcal{T} \times (\text{Args} \cup \{\bot\}))$? The intended meaning is distributions over (tool invocations OR no-action), i.e., $\Delta(\mathcal{T} \times \text{Args} \cup \{\bot\})$. This is correct but could benefit from explicit parentheses: `$L: \mathcal{S} \to \Delta((\mathcal{T} \times \text{Args}) \cup \{\bot\})$`

**Lines 13-15 — Definition 2:**
`$\mathcal{T}_R$ = {t $\in$ $\mathcal{T}$ | reads_sensitive(t)}`

⚠️ ISSUE. Fragmented: `{t`, `$\in$`, `$\mathcal{T}$`, `|` are mix of text and LaTeX. Should be one block:
`$\mathcal{T}_R = \{t \in \mathcal{T} \mid \text{reads\_sensitive}(t)\}$`

Same issue for $\mathcal{T}_X$ and $\mathcal{T}_U$ on lines 14-15.

Also: `|$\mathcal{T}_R$| = n_R` — the cardinality bars and n_R are outside math mode. Should be: `$|\mathcal{T}_R| = n_R$`

**Lines 17-18 — Definition 3:**
`$\tau_b$.tools $\subseteq$ $\mathcal{T}$`

⚠️ ISSUE. Fragmented. Should be: `$\tau_b.\text{tools} \subseteq \mathcal{T}$`

`|$\tau_b$.tools| = k_b` — same: `$|\tau_b.\text{tools}| = k_b$`

**Lines 27-31 — Definition 4:**
Multiple fragmented formulas. Examples:
- `{$\tau_b$.tools $\cap$ $\mathcal{T}_R$ ≠ $\emptyset$}` — should be: `$\{\tau_b.\text{tools} \cap \mathcal{T}_R \neq \emptyset\}$`
  Note: the `≠` is still Unicode, not `$\neq$`
- `{$\neg$$R_2$($\tau_b$)}` — double `$` creates empty math. Should be: `$\{\neg R_2(\tau_b)\}$`
- `Exfil(b) = A $\cap$ B_R $\cap$ B_X $\cap$ C $\cap$ D` — should be: `$\text{Exfil}(b) = A \cap B_R \cap B_X \cap C \cap D$`

**Line 33 — Definition 5:**
`($\Lambda$, $\leq$)` — should be `$(\Lambda, \leq)$`

**Lines 37-38 — Definition 6:**
`$\tau_0$ = $\bot$` — should be `$\tau_0 = \bot$`
`$\tau_k$ = max($\tau_{k-1}$, taint(t_k))` — should be `$\tau_k = \max(\tau_{k-1}, \text{taint}(t_k))$`

## Theorem 1 Proof (Section 2)

**Line 46 — Theorem statement:**
`Pr[Exfil(b)] $\leq$ p · min(kn_R/n, kn_X/n) · (1 - $R_2$) · q`

⚠️ ISSUE. Severely fragmented. `Pr[...]` not in math mode, `·` is Unicode middle dot not `\cdot`, `min` not `\min`. Should be:
`$\Pr[\text{Exfil}(b)] \leq p \cdot \min(kn_R/n, kn_X/n) \cdot (1 - R_2) \cdot q$`

**Lines 52-54 — Chain rule expansion:**
`Pr[Exfil(b)]` — not in math mode
`= Pr[A $\cap$ B_R $\cap$ B_X $\cap$ C $\cap$ D]` — fragmented

These proof lines should ideally be in a `\begin{align*}...\end{align*}` environment. At minimum, each line should be a complete `$...$` block.

**Line 60:**
`$\tau_b \perp L$` ✅ CORRECT (independence symbol fixed).

**Line 62:**
`B_R \perp A` — `\perp` is outside math mode. Should be `$B_R \perp A$`.

**Lines 84-94 — Hypergeometric expansion:**
`Pr[$\neg$B_R]` — `Pr` not in math, `$\neg$` is isolated.
`= $\prod$_{i=0}^{k-1}` — `$\prod$` is correct but `_{i=0}^{k-1}` is outside math mode, so subscript/superscript won't render.

Should be: `$\Pr[\neg B_R] = \prod_{i=0}^{k-1} \frac{n - n_R - i}{n - i}$`

**Lines 98-102 — Union bound:**
`Pr[B_R] = Pr[$\exists$ t $\in$ $\mathcal{T}_R$: t $\in$ $\tau_b$.tools]`

Extremely fragmented. Should be:
`$\Pr[B_R] = \Pr[\exists\, t \in \mathcal{T}_R : t \in \tau_b.\text{tools}]$`

**Line 106:**
`Pr[B_X] $\leq$ kn_X / n =: $\alpha_X$`
Fragmented. Should be: `$\Pr[B_X] \leq kn_X / n =: \alpha_X$`

**Line 110:**
`Pr[B_R $\cap$ B_X] $\leq$ min(Pr[B_R], Pr[B_X]) $\leq$ min($\alpha_R$, $\alpha_X$)`
Fragmented. Should be: `$\Pr[B_R \cap B_X] \leq \min(\Pr[B_R], \Pr[B_X]) \leq \min(\alpha_R, \alpha_X)$`

## Theorems 2-5 (Sections 3-6)

**Theorem 2 proof (lines ~170-200):** Same fragmentation pattern — `Pr[...]` outside math mode, `$\leq$` as isolated inline.

**Theorem 3 proof (lines ~205-215):**
`$\leq$ p · min($\alpha_R$, $\alpha_X$) · (1 - $R_2$) · q`
Same fragmentation issue as Theorem 1.

**Theorem 4 proof (lines ~225-250):**
`$\tau_k$ = max($\tau_{k-1}$, taint(t_k))` — should be one block
`$\tau_i$ $\leq$ $\tau_{i+1}$` — should be `$\tau_i \leq \tau_{i+1}$`
`$\top$ $\leq$ $\tau_k$ $\leq$ $\top$` — should be `$\top \leq \tau_k \leq \top$`

**Theorem 5 proof (lines ~260-290):**
`$\tau_b$ = CapabilityIssuer(event_metadata)` — should be `$\tau_b = \text{CapabilityIssuer}(\text{event\_metadata})$`

## Concrete Instantiation (Section 7)

**Line ~300:**
`$\alpha_R$ = kn_R/n = 5 · 8 / 55 = 40/55 ≈ 0.727`
Should be: `$\alpha_R = kn_R/n = 5 \cdot 8 / 55 = 40/55 \approx 0.727$`

Hypergeometric comparison values have similar issues with `≈` vs `$\approx$`.

## Summary

The Supplementary Notes have a **systematic fragmentation problem**: almost every formula is broken into multiple `$...$` fragments interspersed with bare text operators (`=`, `≤`, `≥`, `·`, `≠`, `≈`). While each `$...$` fragment is individually correct LaTeX, the overall rendering will be inconsistent — math-mode text will be italic while operators between fragments will be roman, and subscripts/superscripts on `\prod`, `\sum` will not render when they appear outside `$...$`.

**Root cause:** The sed-based batch conversion replaced individual Unicode symbols with LaTeX equivalents, but did not consolidate multi-symbol expressions into single math-mode blocks.

**Required fix:** Every mathematical expression (including operators, comparisons, and subscripts) that forms a logical unit must be enclosed in a single `$...$` block. For display equations in proofs, `$$...$$` or `\begin{align*}...\end{align*}` should be used.

| Issue type | Count | Severity |
|-----------|-------|----------|
| Fragmented formulas (multiple $...$ per expression) | ~60 | HIGH |
| Unicode operators outside math mode (≠, ≈, ·, ∧) | ~15 | HIGH |
| Currency $ misinterpreted as LaTeX | 4 | HIGH |
| Subscript/superscript outside math mode | ~10 | MEDIUM |
| Operator precedence ambiguity | 1 | LOW |
