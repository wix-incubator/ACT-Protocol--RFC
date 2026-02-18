# Design Log #4: Open Issues (v0.4 Review)

## Background

A full pass over the v0.4 proposal identified 14 issues ranging from internal inconsistencies to missing protocol elements. This design log triages, groups, and proposes resolutions.

## Issues

### A. Inconsistencies (fix directly in proposal)

**A1. `intent_match` type mismatch (§5.2)**
- Table says Boolean, generic example shows `"failure"` (string), hotel example shows `true` (boolean).
- **Resolution**: Make it a Boolean. The table definition is correct. Fix the generic example from `"failure"` to `false`.

**A2. Authorization wording in §4.1**
- Says session "was originally intended for the requesting `act_origin`" — but `act_origin` is the Global Agent, not the Local Agent.
- The context pull is server-to-server, so the Global Agent cannot verify the caller's domain from the request itself. Any self-declared header (e.g., `ACT-Origin`) is trivially spoofable.
- **Resolution**: This is a **known limitation** of the protocol, accepted as a trade-off for simpler implementation. The real protection is that session IDs are cryptographically random, short-lived (5 min handoff window), and effectively single-use — the "unguessable URL" pattern. Rewrite §4.1 to remove the misleading authorization bullet and instead state that the session ID itself serves as the authorization token, with implementation notes on entropy and TTL.

**A3. `act_session_id` in context response (§4)**
- Generic example omits it; hotel example includes it.
- **Resolution**: Include it. The response should echo the session ID so the Local Agent can confirm it received the right context. Add to generic example and field table.

**A4. `payload` vs. `intent`/`preferences`/`constraints` relationship (§4)**
- Both exist in the response, relationship unclear.
- **Resolution**: Clarify that `intent`, `preferences`, and `constraints` are **human-readable, unstructured** fields for Local Agents that don't support Schema.org parsing. The `payload` is the **machine-readable, structured** representation. They are complementary views of the same data. The Local Agent MAY use either or both. Add a brief note after the field table.

### B. Underspecified Areas (expand in proposal)

**B1. Local Agent authentication in §4.1**
- "User-Agent or domain-specific identifier" is not real authentication.
- Since the context pull is server-to-server, domain-based verification is not feasible — any header the Local Agent sends is self-declared and spoofable.
- **Resolution**: Acknowledge as a **known limitation**. The protocol deliberately trades caller authentication for simplicity and zero-setup adoption. Security relies on the unguessable-URL pattern: cryptographically random session IDs + short TTL + single-use semantics. Remove the misleading "Authentication" bullet from §4.1 and replace with an honest description of the trust model. Note that stronger mechanisms (mutual TLS, pre-registered API keys) can be layered on by implementations that need them.

**B2. Context signing mechanism (§6.3)**
- Says "SHOULD be signed (e.g., using JWS)" but doesn't say how.
- **Resolution**: Define two options, recommend one:
  - Option 1: JWS Compact Serialization — entire response body is the JWS payload, signature in `ACT-Signature` response header. Simple, doesn't change the JSON structure.
  - Option 2: JWS JSON Serialization — response body is a JWS envelope wrapping the context. Changes the response structure.
  - **Decision**: Option 1. Keeps the response body as clean JSON (easier to debug, parse, log). The signature travels in a header. The Local Agent can optionally verify.

**B3. `.well-known/act-pubkey.json` format (§6.3)**
- No key format specified.
- **Resolution**: Use JWK Set (RFC 7517), consistent with OIDC discovery. Supports key rotation via `kid` (Key ID). The `ACT-Signature` header includes the `kid` so the Local Agent knows which key to verify with.

**B4. Feedback response body (§5)**
- Undefined what the Global Agent returns.
- **Resolution**: The Global Agent SHOULD return `200 OK` with a minimal JSON acknowledgment:
  ```json
  { "status": "accepted" }
  ```
  This confirms receipt. No other fields required. Error cases are covered by §4.2 response codes.

**B5. Multiple pulls and multiple feedback events**
- Implied but not stated.
- **Resolution**: Make explicit:
  - Context pull: The Local Agent MAY pull context multiple times during the Active phase. The Global Agent returns the same cached response. This supports retries and multi-page architectures.
  - Feedback: The Local Agent MAY send multiple feedback POSTs (e.g., `engagement` followed by `conversion`). Each is independent.

### C. Missing Protocol Elements

**C1. Protocol version (§3)**
- No versioning mechanism.
- **Resolution**: Add `act_version` URL parameter. Value is the protocol major version (e.g., `1`). Absent = version 1 (backward compatible default). The Global Agent includes it in the link; the Local Agent can check compatibility before pulling.

**C2. ACT discovery mechanism**
- No way for Local Agents to advertise ACT support.
- **Questions**:
  - Q: Is this needed for v1?
  - A: Not strictly. The Global Agent can try sending ACT links to any site — if the site doesn't support ACT, the `act_` parameters are simply ignored and the user lands on a normal page. ACT degrades gracefully.
  - Q: When would discovery matter?
  - A: For the Global Agent to know it can *recommend* ACT-enabled sites over non-ACT sites. Also for search engines to index ACT capability.
- **Resolution**: Define as optional. Local Agents MAY publish `/.well-known/act.json` with capabilities. Not required for the protocol to function. Defer detailed schema to a future version.

**C3. `abandonment` action in feedback (§5.2)**
- Only `engagement` and `conversion` exist.
- **Resolution**: Add `abandonment` to the enum. This lets the Local Agent signal that the user left without completing intent. Useful for the Global Agent's routing decisions. The Local Agent is not required to send it (it may not know the user left), but MAY send it if it can detect session end (e.g., server-side session timeout).

**C4. Rate limiting / abuse guidance**
- The `act_callback_url` is public in the URL.
- **Resolution**: Add an implementation note:
  - The Global Agent SHOULD rate-limit context pulls per session (one successful pull is expected; allow a small retry window).
  - The Global Agent SHOULD rate-limit feedback POSTs per session (a handful of events is expected, not a stream).
  - The `act_session_id` SHOULD be cryptographically random (min 128 bits) to prevent guessing.

### D. Security Edge Case

**D1. URL parameter tampering**
- If an attacker modifies `act_callback_url` and `act_origin` to point to their own domain, the Local Agent fetches from the attacker and validates against the attacker's keys.
- **Analysis**: This requires the attacker to intercept and modify the URL *before* the user clicks it (MITM on the Global Agent's response). If the attacker has that capability, they could replace the entire link. This is not an ACT-specific vulnerability — it's a general link-integrity problem.
- **Mitigations already in place**:
  - The Global Agent serves links over HTTPS (transport security).
  - The user sees the destination domain in the browser (unchanged by ACT params).
  - The Local Agent can maintain an allowlist of trusted `act_origin` domains.
- **Resolution**: Add a brief note in §6.3 acknowledging this threat model and recommending that Local Agents maintain a trusted-origins allowlist. No protocol change needed.

## Implementation Plan

**Phase 1 — Quick fixes (A1–A4):** Fix inconsistencies directly in the proposal. No design decisions needed.

**Phase 2 — Expand underspecified areas (B1–B5):** Update §4.1 (Local Agent auth), §6.3 (signing mechanism, key format), §5 (feedback response, multiple events).

**Phase 3 — Add missing elements (C1–C4):** Add `act_version` to §3, `abandonment` to §5.2, implementation notes for abuse prevention, optional discovery note.

**Phase 4 — Security note (D1):** Brief addition to §6.3.

## Trade-offs

**No caller auth vs. stronger Local Agent auth (A2/B1):**
- ✅ No auth: Zero setup, any website can participate, lowest possible adoption barrier
- ❌ No auth: Anyone with the session ID can pull context
- Mitigation: Cryptographically random IDs + 5 min TTL + single-use = unguessable URL pattern. The session ID only appears in the URL the user clicks — an attacker would need to intercept it in transit (HTTPS prevents this) or guess it (128+ bits prevents this).
- Decision: Known limitation for v1. Stronger mechanisms (mTLS, pre-shared keys) can layer on for high-security use cases.

**Signature in header vs. JWS envelope (B2):**
- ✅ Header: Clean JSON body, easy to debug, optional verification
- ❌ Header: Non-standard, requires custom parsing
- ✅ Envelope: Standard JWS, single artifact
- ❌ Envelope: Wraps the JSON, harder to inspect without tooling
- Decision: Header (`ACT-Signature`). Keeps the happy path simple.

**Discovery mechanism now vs. later (C2):**
- ✅ Now: Global Agents can prioritize ACT-enabled sites
- ❌ Now: Adds complexity, unclear what capabilities to advertise, premature
- Decision: Defer. ACT degrades gracefully — non-ACT sites just ignore the parameters.
