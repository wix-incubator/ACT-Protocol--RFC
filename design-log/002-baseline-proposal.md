# Design Log #2: Baseline Proposal (v0.3)

## Background

Captures the scope and structure of the ACT Protocol proposal as of v0.3, before further expansions. Serves as a reference point for what was established and what was left underspecified.

## Proposal Structure

| Section | Content | Status |
|---|---|---|
| §1 Introduction | Protocol purpose: eliminate cold-start when user clicks from Global Agent to website | Stable |
| §2 Terminology | Global Agent, Local Agent, ACT Session, Context Pull | Stable |
| §3 Handoff (URL Params) | `act_session_id`, `act_origin`, `act_callback_url` | Stable |
| §4 Context Retrieval | Pull model: Local Agent GETs context from Global Agent | Stable |
| §5 Feedback Loop | Local Agent POSTs milestones back to Global Agent | Stable |
| §5.3 Feedback Auth | `feedback_token` issued in context response, required in feedback POST | Added in v0.3 |
| §6.1 Consent Orchestrator | Global Agent manages consent before releasing data | Stable (consent_token underspecified) |
| §6.2 Cookie-Free Evolution | Intent over ID, zero-knowledge start, session de-identification | Stable |
| §6.3 Security & Verification | Bidirectional trust: JWS for context, feedback_token for feedback | Added in v0.3 |
| §6.4 Identity & User IDs | No protocol-level user ID; optional SSO identity hints via OIDC | Added in v0.3 |
| §7 Design Alternatives | Rejected alternatives + comparison table (ACT vs WebMCP vs A2A) | Stable |
| §8 Examples | Hotel booking end-to-end example | Stable |

## Known Gaps (as of v0.3)

1. **`consent_token` structure** -- described as "cryptographic proof" but format, contents, and verification flow are undefined
2. **Session lifecycle** -- "ephemeral" and "short TTL" mentioned but no concrete lifetime, phases, or termination semantics
3. **Error handling** -- no defined error responses for expired sessions, invalid tokens, or malformed requests
4. **Rate limiting / abuse** -- no guidance on protecting the callback URL from abuse
5. **Versioning** -- no mechanism for protocol version negotiation between agents

## Decisions Made in v0.3

- **Feedback token over Local Agent PKI**: Lower adoption barrier; trust bootstraps from the handshake
- **No user ID in base protocol**: Privacy-first; identity handled via optional SSO hints
- **SSO via OIDC hints only**: ACT carries hints, IDP carries identity; two configurations (A: Global Agent is IDP, B: shared third-party IDP)
