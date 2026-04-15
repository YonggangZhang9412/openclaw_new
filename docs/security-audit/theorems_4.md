# Supplementary Note 4: Privilege Exposure Bound

## Definitions

**Definition 18 (Privilege Exposure).** For an agent system with permission model P: 𝒮 → 𝒫(𝒯) operating over a session divided into N discrete time intervals, the privilege exposure is:

Ξ = Σₖ₌₁ᴺ |P(sₖ)| · Δtₖ     ... (Ξ-def)

where P(sₖ) is the permission set during interval k and Δtₖ > 0 is the duration of interval k. Ξ has units of tool-seconds.

## Proposition 2: Ambient Authority Exposure

**Proposition 2.** Under ambient authority P(s) = P₀, a session of duration T_session has:

Ξ_ambient = |P₀| · T_session     ... (amb)

**Proof.** Under ambient authority, the session is a single interval (N = 1) with P(s₁) = P₀ and Δt₁ = T_session. Substituting into (Ξ-def):

Ξ_ambient = Σₖ₌₁¹ |P(sₖ)| · Δtₖ = |P₀| · T_session ∎

## Proposition 3: Event-Scoped Authority Exposure Bound

Under event-scoped authority, the session is partitioned into B event batches with inter-batch gaps where P(s) = ∅. Batch bᵢ has tool grant set P_bᵢ ⊆ 𝒯 with |P_bᵢ| ≤ k (maximum tools per token) and active duration Δᵢ ≤ TTL_max (token time-to-live).

**Proposition 3.** Ξ_capability ≤ B · k · TTL_max, and:

Ξ_capability / Ξ_ambient ≤ (k / |P₀|) · (B · TTL_max / T_session)     ... (ratio)

**Proof.**

Starting from (Ξ-def) applied to the event-scoped model. The session has N = 2B − 1 intervals (B batch intervals + B − 1 gap intervals). For gap intervals, P(s) = ∅, so |P(s)| = 0 and those terms vanish:

Ξ_capability = Σᵢ₌₁ᴮ |P_bᵢ| · Δᵢ + Σⱼ₌₁ᴮ⁻¹ |∅| · Δgapⱼ
             = Σᵢ₌₁ᴮ |P_bᵢ| · Δᵢ + 0
             = Σᵢ₌₁ᴮ |P_bᵢ| · Δᵢ     ... (i)

By assumption, |P_bᵢ| ≤ k for all i. Therefore:

|P_bᵢ| · Δᵢ ≤ k · Δᵢ     for each i     ... (ii)

By assumption, Δᵢ ≤ TTL_max for all i. Therefore:

k · Δᵢ ≤ k · TTL_max     for each i     ... (iii)

Summing (iii) over i = 1, ..., B:

Σᵢ₌₁ᴮ |P_bᵢ| · Δᵢ ≤ Σᵢ₌₁ᴮ k · TTL_max = B · k · TTL_max     ... (iv)

Combining (i) and (iv):

Ξ_capability ≤ B · k · TTL_max     ... (bound)

Dividing (bound) by (amb):

Ξ_capability / Ξ_ambient ≤ (B · k · TTL_max) / (|P₀| · T_session)     ... (v)

Factoring:

= (k / |P₀|) · (B · TTL_max / T_session)     ... (ratio) ∎

## Concrete Instantiation

With k = 5, |P₀| = 55, TTL_max = 300s, T_session = 86,400s.

Worst-case batch count: batches can arrive at most once per TTL_max (each batch occupies its full TTL window before the next begins):

B ≤ T_session / TTL_max = 86,400 / 300 = 288     ... (B-bound)

Substituting into (ratio):

Ξ_capability / Ξ_ambient ≤ (5 / 55) · (288 · 300 / 86,400)
                         = (5/55) · (86,400 / 86,400)
                         = 5/55
                         = 1/11
                         ≈ 0.091     ... (concrete)

Event-scoped authority exposes at most **9.1%** of the privilege surface of ambient authority under worst-case assumptions. With typical k = 2–3 and lower batch frequency, the bound is substantially tighter.

## Interpretation

Ξ is not a probability of attack success. It is a structural measure of opportunity: the total tool-time window during which a successful prompt injection could exploit an authorized tool. Reducing Ξ narrows this window proportionally, independent of the injection technique used.
