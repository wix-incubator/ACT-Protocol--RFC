# ACT Protocol — Agent Context Transfer

ACT is an open protocol that lets AI assistants hand off conversation context to websites, so users don't have to repeat themselves.

When a user clicks a link from a Global Agent (e.g., ChatGPT, Gemini, Claude) to a website, the site's Local Agent can pull the conversation context — intent, preferences, constraints — and deliver a personalized experience from the first page load. No cookies, no tracking, no cold start.

## How It Works

```
User → Global Agent: "Find me a wheelchair-accessible hotel in Paris, under $600/night"
Global Agent → User: "Here's Hotel Deluxe — [View & Book]"
User clicks link → Hotel Deluxe site loads
Hotel Deluxe server → Global Agent: pulls context (intent, constraints, consent token)
Hotel Deluxe → User: page pre-filled with matching suites, dates, accessibility filters
```

The protocol uses a **pull model** — the website fetches context server-to-server, no data is embedded in the URL. Sessions are ephemeral, consent is explicit, and no persistent user identifiers are exchanged.

## Specification

The full proposal is at [`docs/proposal.md`](docs/proposal.md) (currently v0.4).

**Key sections:**

| Section | Topic |
|---|---|
| §3 | Handoff — URL parameters that start a session |
| §4 | Context Retrieval — the pull model and session lifecycle |
| §5 | Feedback Loop — Local Agent reports outcomes back |
| §6 | Privacy & Consent — consent tokens, security, identity |

## Design Logs

Discussion, trade-off analysis, and rationale for proposal changes live in [`design-log/`](design-log/). See the [index](design-log/index.md) for a catalog.

## Status

This is a draft specification. Feedback and contributions are welcome.

## License

TBD
