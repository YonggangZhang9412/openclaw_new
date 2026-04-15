# Supplementary Note 4: Privilege Exposure Bound

## Definition

**Definition 18 (Privilege Exposure).** For an agent system with permission model P: S → 2^T operating over a session partitioned into discrete time intervals, the privilege exposure is:

Ξ = Σₖ₌₁ᴺ |P(sₖ)| · Δtₖ

where N is the number of time intervals, P(sₖ) is the permission set during interval k, and Δtₖ is the duration of interval k. Ξ has units of tool-seconds and measures the cumulative authorization surface available to an attacker.

## Proposition 2: Ambient Authority Exposure

Under ambient authority, P(s) = P₀ for all states. The session consists of a single interval of duration T_session:

Ξ_ambient = |P₀| · T_session

## Proposition 3: Event-Scoped Authority Exposure Bound

Under event-scoped authority, the session is partitioned into B event batches, each with token validity Δᵢ ≤ TTL_max and tool grant set P_bᵢ with |P_bᵢ| ≤ k. Between batches, P(s) = ∅.

**Proposition 3.** Ξ_capability ≤ B · k · TTL_max, and the reduction ratio satisfies:

Ξ_capability / Ξ_ambient ≤ (k / |P₀|) · (B · TTL_max / T_session)

**Proof.**

Ξ_capability = Σᵢ₌₁ᴮ |P_bᵢ| · Δᵢ

Since |P_bᵢ| ≤ k and Δᵢ ≤ TTL_max:

≤ Σᵢ₌₁ᴮ k · TTL_max = B · k · TTL_max

Dividing both sides by Ξ_ambient = |P₀| · T_session:

Ξ_capability / Ξ_ambient ≤ (B · k · TTL_max) / (|P₀| · T_session)
                         = (k / |P₀|) · (B · TTL_max / T_session)  ∎

**Example.** With k = 5, |P₀| = 55, TTL_max = 300s, T_session = 86,400s, and B ≤ T_session / TTL_max = 288 (worst case: one batch per TTL window):

Ξ_capability / Ξ_ambient ≤ (5/55) · (288 · 300 / 86,400) = 0.091 · 1.0 = 0.091

That is, event-scoped authority exposes at most 9.1% of the authorization surface of ambient authority under worst-case batch arrival rates. With typical k = 2-3 and lower batch frequency, the ratio is substantially smaller.

**Interpretation.** Ξ is not a probability of attack success. It is a structural measure of opportunity: the total tool-time window during which an attacker who has achieved prompt injection could exploit an authorized tool. Reducing Ξ narrows the attack window proportionally.
