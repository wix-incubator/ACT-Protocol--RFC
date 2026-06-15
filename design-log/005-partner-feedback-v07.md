# Design Log #5: Partner Feedback & v0.7 Changes

## Background

An external partner reviewed the v0.6 spec and provided three pieces of technical feedback. While they declined to co-author, all three points were substantive and drove spec changes in v0.7.

## Feedback and Resolutions

### F1. User context can leak to unintended recipients

**Feedback**: The `act_session_id` travels in a browser URL query string. The session ID is the only thing gating access to the user's intent and constraints. Browser extensions, link-preview fetchers, corporate proxies, or user forwarding can observe the URL and pull the full context. The `consent_token.aud` and `ACT-Signature` prove authenticity, not that the right party is reading the data. mTLS fixes this but breaks zero-setup.

**Analysis**: The spec already had the unguessable-URL pattern (128-bit entropy, short TTLs) and URL scrubbing (§3.1), but scrubbing was SHOULD-level and could be skipped. The reviewer's insight was that URL observation is a distinct threat from URL guessing — different mitigations apply.

With a mandatory server-side redirect, we traced which observation vectors survive:

| Vector | Eliminated by redirect? |
|--------|------------------------|
| Link-preview fetchers (Slack, iMessage) | Yes — they'd trigger the context pull themselves, locking out the real Local Agent |
| Referrer headers to third parties | Yes — clean URL is what propagates |
| Browser history / address bar | Yes — redirect replaces the URL before it persists |
| User forwarding / screenshots | Yes — user copies the clean URL |
| HTTPS-inspecting corporate proxies | No — they terminate TLS |
| Browser extensions with `webRequest` | No — they see the URL before redirect |

The surviving vectors require privileged access to the user's TLS-terminated traffic — an endpoint-compromise threat model. At that privilege level, the attacker can read page content, steal cookies, etc. The `act_session_id` is not incrementally worse.

**Resolution**:
- §3.1: Server-side redirect upgraded from SHOULD to MUST. Added sequence diagram showing the full redirect flow.
- §4.1: Trust model paragraph expanded to explicitly enumerate remaining observation vectors, scope them as endpoint-compromise (out of scope, same as OAuth), and cross-reference first-pull binding.
- First-pull binding (already in spec) provides a second defense: after the legitimate Local Agent pulls context, the `feedback_token` is required for re-pulls and feedback. An observer who is slow gets locked out.

**Trade-off**: Making redirect mandatory increases implementation burden for Local Agents. Accepted because the security benefit is significant and the implementation cost is a standard HTTP redirect — well-understood infrastructure.

### F2. Feedback channel is an unguarded path back into the user's AI

**Feedback**: §5.3 authenticates who sends feedback (via `feedback_token`) but places no constraints on what they send. Free-text `description` and arbitrary `payload` flow into the Global Agent's conversation and memory. This is a prompt-injection surface. Additionally, the consent model in §6.1 only covers Global-to-Local — there's no consent story for data flowing in the reverse direction (the §8 example sends name, reservation number, and total price back without user consent).

**Analysis**: This was the strongest point. We identified two distinct problems:

1. **Third-party injection**: A browser extension or proxy observing the session ID could send fake feedback. This is now closed by the mandatory server-side redirect — only the Local Agent server holds the `feedback_token`.

2. **Legitimate Local Agent as a vector**: Even the authorized Local Agent could send malicious content via `description` (prompt injection) or push unwanted data into the Global Agent's memory via `payload` (reverse data flow without consent).

For the reverse consent question, we realized that the consent prompt is already happening — the user is already in a decision moment about data sharing with this specific Local Agent. Bundling both directions into one consent flow is natural UX rather than a second interruption.

The segmentation question — what categories make sense for reverse consent — yielded three tiers based on risk:

| Category | Risk | Rationale |
|----------|------|-----------|
| `order_status` | Low | The user expects to hear what happened |
| `transaction` | Low-Medium | Useful but financially sensitive |
| `personal_data` | High | Local Agent pushing PII it holds from its own records into the Global Agent |

**Resolution**:
- §5.2: Size limits added — `description` ≤ 1024 chars, `payload` ≤ 16 KB. `feedback_category` field added to feedback request body.
- §5.6 (new): Feedback Safety section. Mandates treating all feedback as untrusted input. Four rules: use structured fields for state updates, extract data by schema type, generate summaries from structured data, present `description` as quoted third-party text if surfaced.
- §6.1: `feedback_categories` claim added to the consent token JWT. Three categories defined. The Global Agent enforces on receipt — feedback with a `feedback_category` not in the consented list is rejected with `403 Forbidden`.
- §4.2: Added `403 Forbidden` and `413 Payload Too Large` to response codes table.
- §8: Hotel example updated — `feedback_category: "transaction"` added, `underName` (user's name) removed since `personal_data` wasn't consented, with explanatory note.

**Key design decision — bidirectional consent in one prompt**: Rather than a separate consent mechanism for the reverse direction, the existing consent flow captures both directions. The UX is: *"Share your dietary preferences and budget with Italy Eats? Italy Eats may send back order status and pricing to your assistant."* This avoids consent fatigue while giving the user control.

### F3. Zero-setup vs. security model tension

**Feedback**: §1 promises no pre-registration, but the Local Agent allowlist in §4.1(c) and §6.3 is itself pre-registration. The realistic deployment model is a small curated set of Global Agent origins.

**Analysis**: The reviewer is technically correct. The allowlist is configuration. However, it's *unilateral* configuration — the Local Agent decides who to trust without coordinating with the Global Agent. This is analogous to configuring trusted CAs for TLS or SSO identity providers ("Sign in with Google/Apple/Facebook"). No API keys are exchanged, no shared secrets, no registration flows.

The realistic deployment model is indeed a small set of major Global Agent origins. A Local Agent's allowlist might have 5-10 entries. This is "configure once, works with everyone" — not "zero configuration."

**Resolution**:
- §1: Reframed from "zero-setup adoption" to "low-barrier adoption." Explicitly states that Local Agents maintain a unilateral allowlist, analogous to configuring trusted CAs. Preserves the claim that no bilateral agreements are needed.

**Trade-off**: Honesty over marketing. "Low-barrier" is less catchy than "zero-setup" but more accurate, and builds trust with technical reviewers who would notice the gap.

## Summary of v0.7 Changes

| Section | Change |
|---------|--------|
| §1 | "zero-setup" → "low-barrier adoption"; allowlist acknowledged |
| §3.1 | Server-side redirect MUST; sequence diagram added |
| §4.1 | Threat model scoped; endpoint-compromise declared out of scope |
| §4.2 | Added 403 and 413 response codes |
| §5.2 | Size limits; `feedback_category` field added |
| §5.6 | New section: Feedback Safety (prompt injection, enforcement) |
| §6.1 | `feedback_categories` in consent token; bidirectional consent; category table |
| §8 | Hotel example updated with `feedback_category`; `underName` removed |
