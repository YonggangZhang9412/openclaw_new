# Formula Review: draft3 Main Text

## draft3_1.md (Abstract + Section 1)

**Line 9 — Theorem 1 bound in Abstract:**
`$\Pr[\text{Exfil}] \leq p \cdot \min(\alpha_R, \alpha_X) \cdot (1-R_2) \cdot q$`

✅ CORRECT. LaTeX renders properly. \Pr produces upright "Pr", \text{Exfil} in Roman, \min is operator, \alpha_R/\alpha_X are correct Greek subscript, R_2 is italic with subscript 2.

**Line 17 — "$65 billion" and "$45 million":**
`$65 billion` and `$45 million`

⚠️ ISSUE. The dollar signs around "65" and "45" are currency symbols, NOT LaTeX delimiters. But in a LaTeX document, `$65` would start math mode and render "65" as math italic. This must be escaped: `\$65 billion` and `\$45 million`.

Similarly on line 33: `$45 million in security incidents` — same issue.

**Fix needed:** `$65` → `\$65`, `$45` → `\$45` in all currency references.

---

## draft3_2.md (Section 2)

**Line 9 — "$500":**
`her request to transfer $500 reflects Alice's intent`

⚠️ SAME ISSUE. Currency `$500` will be interpreted as LaTeX math mode. Needs `\$500`.

No other formulas in this file. All mathematical content is in prose form (C1, C2, C3 are text labels, not formulas).

---

## draft3_3.md (Section 3)

**Line 48 — Trust table, REMOTE_VERIFIED row:**
`$\alpha_R$, $\alpha_X$ reduced; tighter bound`
✅ CORRECT.

**Line 49 — Trust table, REMOTE_OPEN row:**
`$\alpha_X$ = 0 → $\Pr[\text{Exfil}]$ = 0`
⚠️ ISSUE. The `= 0` and `→` are outside LaTeX. Should be: `$\alpha_X = 0 \Rightarrow \Pr[\text{Exfil}] = 0$`

**Line 51 — Corollary formula:**
`$\Pr[\text{Exfil} \mid \text{REMOTE\_OPEN}]$ ≤ $p \cdot \min(\alpha_R, 0) \cdot (1-R_2) \cdot q$ = 0`

⚠️ ISSUE. Formula is fragmented into multiple `$...$` blocks with `≤` and `= 0` outside LaTeX. Should be one block:
`$\Pr[\text{Exfil} \mid \text{REMOTE\_OPEN}] \leq p \cdot \min(\alpha_R, 0) \cdot (1-R_2) \cdot q = 0$`

**Line 53:**
`$R_2$ = 1` — should be `$R_2 = 1$` (keep comparison inside math mode)

**Line 63 — marker space:**
`$2^{64}$` ✅ CORRECT.

**Line 67 — taint verification:**
`round taint $\leq$ policy threshold` ✅ CORRECT (inline comparison).

**Line 69 — Rule of Two paragraph:**
Multiple `$R_2$ = 0` — should be `$R_2 = 0$`
`$\mathcal{T}_U, \mathcal{T}_R, \mathcal{T}_X$` ✅ CORRECT.

**Line 79 — Auditable decisions:**
`$\leq$ USER` — should be `$\leq$~USER` or `$\leq \text{USER}$`

**Line 85 — CVE case:**
`$\alpha_X$ = 0 and $\Pr[\text{Exfil}]$ = 0` — should consolidate: `$\alpha_X = 0$ and $\Pr[\text{Exfil}] = 0$`

---

## draft3_4.md (Section 4-5)

**Line 11 — Theorem 1 display equation:**
`$$\Pr[\text{Exfil}(b)] \leq p \cdot \min\!\left(\frac{kn_R}{n},\, \frac{kn_X}{n}\right) \cdot (1 - R_2) \cdot q$$`
✅ CORRECT. Display math, \min\! removes extra space, \left/\right for auto-sizing, \frac for fractions.

**Line 13 — inline refs:**
`$kn_R/n$` — ⚠️ COULD IMPROVE: `$kn_R/n$` renders as `knR/n` in italics which looks like variable multiplication. Better: `$\frac{kn_R}{n}$` or `$kn_R\!/n$`.

**Line 15 — Theorem 2:**
`$\Pr[\exists b: \text{Exfil}(b)]$ $\leq$ $1 - (1-\varepsilon)^B$`

⚠️ ISSUE. Fragmented into three separate `$...$` blocks. Should be one:
`$\Pr[\exists b: \text{Exfil}(b)] \leq 1 - (1-\varepsilon)^B$`

Similarly: `$\Pr[\exists b: \text{Exfil}(b)]$ $\geq$ $1-(1-p)^B$` → consolidate.

**Line 17 — Theorem 3:**
`$R_2$ = 1: $\Pr[\text{Exfil}(b)]$ = 0` — should be `$R_2 = 1$: $\Pr[\text{Exfil}(b)] = 0$`

**Line 19 — Theorem 4:**
`$\tau_i \leq \tau_j$` ✅ CORRECT.
`$q_{\text{within}}$ = 0` — should be `$q_{\text{within}} = 0$`

**Line 37 — Table:**
`$\geq$ 0.5` ✅ OK for table cell (inline).
Line 37 has bare `≈ 1.0` — should be `$\approx 1.0$`

---

## draft3_5a.md (Section 6-7)

**Line 39:**
`$\mathcal{T}_R$, $\mathcal{T}_X$, $\mathcal{T}_U$` ✅ CORRECT.

---

## draft3_5b.md (Methods)

**Line 28:**
`$\mathcal{T}_U$ $\cap$ $\mathcal{T}_R$ $\cap$ $\mathcal{T}_X$`
⚠️ ISSUE. Fragmented. Should be: `$\mathcal{T}_U \cap \mathcal{T}_R \cap \mathcal{T}_X$`

**Line 34:**
`t $\notin$ τ.granted_tools` — mixed: `t` and `τ` should both be in math mode: `$t \notin \tau.\text{granted\_tools}$`

Mixed LaTeX/Unicode on same line: `t ∈ τ.denied_tools` — the `∈` is still Unicode, not LaTeX.

**Line 36:**
`$\notin$` and `$\geq$` — correct LaTeX operators, but surrounding text `path` and `granted_paths` should be in `\text{}`.

**Line 41:**
`$2^{64}$` ✅ CORRECT.

**Line 53:**
`$\tau_0$`, `$\tau_k$`, `$\tau_{k-1}$` ✅ CORRECT.

---

## draft3_5c.md (Conclusion + References)

**Line 19:**
`$45M Crypto Security Breach` — this is a reference title containing "$45M". The `$` will be interpreted as LaTeX delimiter. Must escape: `\$45M`.

---

## Summary of Issues Found

| File | Line | Issue | Severity |
|------|------|-------|----------|
| draft3_1.md | 17, 33 | `$65`, `$45` currency as LaTeX | HIGH |
| draft3_2.md | 9 | `$500` currency as LaTeX | HIGH |
| draft3_3.md | 49, 51 | Fragmented formulas across multiple $...$ | MEDIUM |
| draft3_3.md | 53, 69, 85 | `$R_2$ = 0` should be `$R_2 = 0$` | MEDIUM |
| draft3_4.md | 15 | Theorem 2 formula fragmented into 3 blocks | MEDIUM |
| draft3_4.md | 17 | Theorem 3 same issue | MEDIUM |
| draft3_4.md | 19 | `$q_{\text{within}}$ = 0` | LOW |
| draft3_4.md | 37 | `≈ 1.0` should be `$\approx 1.0$` | LOW |
| draft3_5b.md | 28 | `$\cap$` fragmented between three $...$ | MEDIUM |
| draft3_5b.md | 34 | Mixed Unicode ∈ and LaTeX $\notin$ | HIGH |
| draft3_5c.md | 19 | `$45M` in reference title | HIGH |
