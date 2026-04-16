## Methods

### Event model and EventBus

All system triggers are normalized into a typed Event — a frozen dataclass with fields: `id` (12-char hex UUID), `source` (originating module), `type` (event class), `priority` (CRITICAL > HIGH > NORMAL > LOW), `payload` (event-specific dictionary), `origin_chain` (immutable provenance tuple), and `cascade_depth` (auto-incremented by framework; hard limit: 20).

The EventBus receives events through registered EventSources (8 source types: Timer, Cron, FileWatch, ProcessWatch, NetworkWatch, Discovery, SkillEvent, Custom), applies the EventInjectionGate (per-source policy: event type whitelist, sliding-window rate limiting, payload size cap, cascade depth check; CRITICAL priority requires unforgeable EscalationToken), routes through EventFilter (debounce 0.5s, priority routing) into a priority queue (capacity: 10,000; overflow: lowest-priority eviction). EventConsumer dequeues batches (2s window, max 20 events).

### Device authentication

Remote devices authenticate via HMAC challenge-response (s39_device_auth.py): 32-byte random secret per device, `hmac.compare_digest()` for constant-time comparison, `_DUMMY_SECRET` placeholder for unknown devices (ensuring identical computation path to prevent timing-based enumeration), 300-second clock skew tolerance, nonce one-time use. Authenticated devices are assigned a DeviceScope (ADMIN/CLI/MOBILE), which maps to an InjectionPolicy determining rate limits, payload caps, and allowed event types.

### Trust classification

`_classify_event_trust()` inspects `event.source` and `event.origin_chain`:
- **LOCAL_TRUSTED**: 10 internal sources (timer, cron, file, process, network, node, discovery, contract, task, skill_source)
- **REMOTE_VERIFIED**: authenticated remote devices (verified by HMAC handshake)
- **REMOTE_OPEN**: custom events without internal module markers in origin_chain

Trust level determines tool grants via `EVENT_TOOL_GRANTS` mapping. For REMOTE_OPEN: high-risk tools (bash, run_command, daemon_restart, send_email, curl, wget) are removed.

### CapabilityIssuer: event-to-token mapping

For each event batch, the CapabilityIssuer performs:
1. **Trust classification** → determines base tool set
2. **Tool grant selection** → `EVENT_TOOL_GRANTS[event_type]` intersected with trust-level restrictions
3. **Path inference** → `event.payload["path"]` resolved to absolute path, parent directory granted via glob
4. **Rule-of-Two evaluation** → if granted tools span 𝒯_U ∩ 𝒯_R ∩ 𝒯_X simultaneously, external-action tools removed
5. **Token construction** → frozen CapabilityToken with `granted_tools`, `denied_tools`, `granted_paths`, `ttl` (300s), Rule-of-Two flags

### CapabilityGate: three-check verification

For each tool call `(t, args)`:
- **Check 1 (Token)**: t_now > τ.issued_at + τ.ttl → DENY; t ∉ τ.granted_tools → DENY; t ∈ τ.denied_tools → DENY
- **Check 2 (Taint)**: for each param value, query TaintStore (SHA-256 fingerprint → dict lookup → substring match → boundary marker detection); if taint > policy.max_taint → DENY
- **Check 3 (Structural)**: path ∉ granted_paths → DENY; call_count ≥ max_per_round → DENY; causal Rule of Two violated → DENY

### External content wrapping

`wrap_external_content(content, source)` (s12_security.py + tools/_shared/external_content.py):
1. Generate `marker_id = secrets.token_hex(8)` (16 hex chars, 2⁶⁴ possibilities)
2. Sanitize existing boundary markers: replace with `[[MARKER_SANITIZED]]`
3. Normalize Unicode homoglyphs: full-width `＜` → ASCII `<`, CJK `〈` → `<` (13 invisible character classes stripped)
4. Wrap: `<<<EXTERNAL_UNTRUSTED_CONTENT id="{marker_id}">>>` ... `<<<//EXTERNAL_UNTRUSTED_CONTENT>>>`
5. Register content + taint level in TaintStore (SHA-256 fingerprint + substring extraction for content >5KB)

### TaintStore

Registration: `TaintStore.register(content, taint_level, origin_tool)` computes SHA-256 fingerprint (first 16 hex chars), extracts email/URL/path substrings for large content, stores with TTL 600s. Capacity: 10,000 records, LRU eviction.

Query: short-circuit strategy — boundary markers → fingerprint match → substring match → None (fail-open).

CausalTaintTracker: τ₀ = USER; τ_k = max(τ_{k-1}, taint(t_k)). Monotonic (Theorem 4). Reset per batch.

### Code and data availability

The architecture is implemented in Python as part of the ShadowClaw project (55 tools across 16 module categories). Source code available at [repository URL]. Tool enumeration in Supplementary Table 1.
