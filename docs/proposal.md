# Agent Context Transfer (ACT) Protocol v0.5

## 1 Introduction

The **Agent Context Transfer (ACT)** protocol defines a standardized mechanism for a **Global Agent** (e.g., a general-purpose AI assistant) to securely hand off a user’s conversation state, intent, and constraints to a **Local Agent** (e.g., a website-specific AI or service).

The goal of ACT is to eliminate the "cold start" problem when a user clicks a link from an AI assistant to a website, ensuring the website understands exactly what the user is looking for without requiring the user to repeat themselves. In contrast to automation-heavy protocols, ACT is specifically designed to **enhance the human-led browsing experience** by ensuring the website "knows" the nuances of the conversation the user was just having with their Global Agent, allowing for a seamless transition from dialogue to digital storefront or service.

A core design principle is **zero-setup adoption**: any website can participate in ACT without pre-registration, API key exchange, or shared secrets with Global Agents. The protocol relies on public-key infrastructure and short-lived, unguessable session tokens rather than bilateral agreements. This simplicity comes with deliberate trade-offs in caller authentication (see §4.1), which are documented throughout the spec.

## 2 Terminology

* **Global Agent:** The initiating AI platform (e.g., Gemini, ChatGPT, Claude).  
* **Local Agent:** The destination AI or automated service residing on a specific web domain.  
* **ACT Session:** A temporary, conversation-scoped link between the two agents. Progresses through three phases: **Created** (link generated, awaiting context pull), **Active** (context pulled, feedback accepted), and **Closed** (expired or revoked).  
* **Context Pull:** The process by which a Local Agent fetches structured data from the Global Agent.

## 3 The Handoff (URL Parameters)

When a Global Agent directs a user to a website, it appends a minimal set of parameters to the destination URL. This provides the Local Agent with the "keys" needed to retrieve the full context.

### **Parameters**

| Parameter | Description | Example |
| :---- | :---- | :---- |
| act\_session\_id | A unique, ephemeral identifier for the conversation. | sess\_98765abc |
| act\_origin | The identifier/domain of the Global Agent. | agent.google.com |
| act\_callback\_url | The endpoint where the Local Agent fetches context. | https://api.global-ai.com/v1/act |
| act\_version | Protocol major version. Absent defaults to `1`. | 1 |

**Example URL:**

```
https://italy-eats.com/order?act_session_id=sess_98765abc&act_origin=agent.google.com&act_callback_url=https://api.global-ai.com/v1/act&act_version=1
```

**Implementation Note:** To protect user privacy, browsers and servers SHOULD NOT log act\_ prefixed parameters in plain-text server access logs.

## 4 Context Retrieval (The "Pull" Model)

Upon page load, the Local Agent initiates a server-to-server GET request to the `act_callback_url` using the `act_session_id`.

### 4.1 The Secure Handshake

Upon receiving a user via an ACT-enabled link, the Local Agent's server initiates a **HTTP GET** request to the `act_callback_url`.

* **Discovery**: The Local Agent extracts the `act_session_id` and `act_callback_url` from the URL parameters.  
* **Authorization**: The `act_session_id` itself serves as the authorization token. The Global Agent validates that it is active and has not expired (see §4.2 Session Lifecycle).  
* **Trust model**: Because this is a server-to-server call, the Global Agent cannot cryptographically verify the caller's identity. The protocol relies on the **unguessable-URL pattern**: session IDs MUST be cryptographically random (minimum 128 bits of entropy) and short-lived, making interception or guessing infeasible. Implementations requiring stronger caller authentication MAY layer on mutual TLS or pre-registered API keys.

**Implementation Note:** The Global Agent SHOULD rate-limit requests per session — one successful context pull is expected (allow a small retry window), and only a handful of feedback events per session. This limits the impact of leaked session IDs.

**Example Context Response:**

```json
{ 
  "act_session_id": "sess_98765abc",
  "intent": "quick italian meal", 
  "preferences": { 
    "style": "authentic", 
    "delivery": "express" 
  }, 
  "constraints": { 
    "dietary": ["no_fish", "nut_free"], 
    "max_price": 50.00 
  }, 
  "payload": { 
    "@context": "https://schema.org", 
    "@type": "FoodOrder", 
    "description": "Authentic Italian meal with express delivery",     
    "priceCurrency": "USD", 
    "maximumPayloadPrice": 50.00, 
    "orderLocation": { 
      "@type": "OrderAction", 
      "deliveryMethod": "http://schema.org/OnSitePickup" 
    }, 
    "orderedItem": { 
      "@type": "MenuItem", 
      "suitableForDiet": [ "https://schema.org/NoFishDiet", "https://schema.org/NutFreeDiet" ], 
      "cuisine": "Italian" 
      } 
    },
  "consent_token": "eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJhZ2VudC5nb29nbGUuY29tIiwic3ViIjoic2Vzc185ODc2NWFiYyIsImlhdCI6MTc1MDAwMDAwMCwiY2F0ZWdvcmllcyI6WyJkaWV0YXJ5IiwiYnVkZ2V0Il0sImNvbnNlbnRfdHlwZSI6ImV4cGxpY2l0In0.signature",
  "feedback_token": "ft_hmac_abc123def456"
}
```

| Field | Type | Description |
| :---- | :---- | :---- |
| act\_session\_id | String | Echo of the session ID, allowing the Local Agent to confirm the correct context was returned. |
| intent | String | A concise summary of what the user is trying to achieve. |
| preferences | Object | Key-value pairs of user desires (e.g., style, brand, urgency). |
| constraints | Object | Hard requirements that must be met (e.g., allergies, size, budget). |
| payload | Object | **Schema.org** compatible object, representing the constraints and preferences. |
| consent\_token | String (JWT) | A JWS-signed token certifying the user consented to share the included data categories (see §6.1). |
| feedback\_token | String | A session-scoped token the Local Agent MUST include in all Feedback Loop requests (see §5.3). |
| identity\_hints | Object | (Optional) SSO hints for seamless authentication when an existing SSO relationship exists (see §6.4). |

**Note on `payload` vs. `intent`/`preferences`/`constraints`:** The `intent`, `preferences`, and `constraints` fields are human-readable, unstructured summaries. The `payload` is the machine-readable Schema.org representation of the same data. They are complementary — the Local Agent MAY use either or both depending on its capabilities. When both are present, the `payload` is authoritative.

### 4.2 Session Lifecycle

An ACT session progresses through three phases:

```mermaid
stateDiagram-v2
    [*] --> Created: Global Agent generates link
    Created --> Active: Local Agent pulls context
    Created --> Expired: Handoff window elapses
    Active --> Closed: Session TTL expires
    Active --> Closed: Global Agent revokes
    Closed --> [*]
```

| Phase | Duration | Behavior |
| :---- | :---- | :---- |
| **Created** | Link generated → context pulled (or handoff window expires) | Only the context GET is accepted. Feedback POSTs are rejected. |
| **Active** | Context pulled → session TTL or explicit revocation | Context GET returns cached response. Feedback POSTs are accepted. |
| **Closed** | Terminal | All requests return `410 Gone`. |

**Recommended TTLs:**

| Parameter | Value | Rationale |
| :---- | :---- | :---- |
| Handoff window | 5 minutes | User should land on the site shortly after clicking the link. |
| Session TTL | 60 minutes | Covers browsing, comparison, and conversion. |

Global Agents MAY adjust these values. Use cases vary -- a quick food order needs less time than a hotel booking.

**Termination:**
* **Timeout**: The session auto-expires after the session TTL.
* **Explicit revocation**: The Global Agent can close a session at any time (e.g., the user ends the conversation). Subsequent requests receive `410 Gone`.
* **Graceful degradation**: When a Local Agent receives `410 Gone` on a feedback POST, it SHOULD stop sending feedback. The user's browsing experience is unaffected -- only the agent-to-agent link is severed.

**Response codes:**

| Scenario | HTTP Status |
| :---- | :---- |
| Valid context pull | `200 OK` |
| Valid feedback POST | `200 OK` |
| Unknown or invalid session ID | `404 Not Found` |
| Session expired or revoked | `410 Gone` |
| Invalid or missing feedback token | `401 Unauthorized` |

## 5 The Feedback Loop

The feedback loop allows the Local Agent to notify the Global Agent of significant milestones reached during the session. This data is used to verify "Intent Fulfillment" and to inform the Global Agent's future routing decisions (reputation/quality scoring).

### 5.1 Endpoint Discovery

The Local Agent MUST use the `act_callback_url` provided in the initial handoff. The feedback request is an **HTTP POST** to this URL.

### 5.2 Request Body (Schema)

```json
{ 
  "act_session_id": "sess_98765abc", 
  "description": "User found the item, but requested size (11W) is currently out of stock.", 
  "action": "engagement", 
  "intent_match": false, 
  "payload": { 
    "@context": "https://schema.org", 
    "@type": "ItemAvailability", 
    "availability": "https://schema.org/OutOfStock" 
  } 
}
```

| Field | Type | Description |
| :---- | :---- | :---- |
| act\_session\_id | String | The unique ID provided in the initial handoff. |
| description | String | Natural language summary for the Global Agent to explain the status to the user. |
| action | Enum | `engagement` (user interacting), `conversion` (user completed intent), or `abandonment` (user left without completing intent). |
| intent\_match | Boolean | **True** if the Global Agent's context was accurate to the site's capability. **False** if the site couldn't fulfill the specific constraints (e.g., "We don't sell size 15"). |
| payload | Object | **Schema.org** compatible object, giving more details about the engagement / conversion |

### 5.3 Feedback Authentication

The Local Agent MUST include the `feedback_token` (received during Context Retrieval, §4) in the `Authorization` header of every feedback POST:

```
Authorization: Bearer ft_hmac_abc123def456
```

The Global Agent MUST reject any feedback request where:
* The `feedback_token` is missing or invalid.
* The `feedback_token` does not match the `act_session_id` in the request body.
* The session has expired.

This ensures that only the Local Agent that successfully retrieved the context can submit feedback, preventing spoofed conversion or engagement signals from poisoning the Global Agent's routing and reputation data.

### 5.4 Response

The Global Agent SHOULD return `200 OK` with a JSON acknowledgment:

```json
{ "status": "accepted" }
```

Error responses follow the status codes defined in §4.2.

### 5.5 Multiple Events

The Local Agent MAY send multiple feedback POSTs during a single session (e.g., an `engagement` event followed by a `conversion`). Each event is independent. The Global Agent SHOULD accept all valid events until the session closes.

Similarly, the Local Agent MAY pull context multiple times during the Active phase (e.g., for retries or multi-page architectures). The Global Agent returns the same cached response.

## 6 Privacy & Consent

### 6.1 The Consent Orchestrator

The Global Agent serves as the primary **Consent Orchestrator** for the user journey. Unlike traditional web tracking where consent is often managed by invisible scripts, ACT makes consent explicit and centralized:

* **Role**: The Global Agent is responsible for auditing the user's conversation and identifying sensitive data (e.g., health, finance, or PII).  
* **Execution**: Before releasing an Intent Package via the callback URL, the Global Agent must ensure appropriate consent.  
  * **Implicit Intent:** General parameters (e.g., "blue sneakers") are consented to by the user's action of clicking the link.  
  * **Explicit Sensitive Data:** For constraints such as medical allergies or precise location, the Global Agent must trigger a UI prompt: *"Share your allergy profile with \[Local Agent\]?"*  
* **The Token as Certification**: When a Local Agent receives a context response, the `consent_token` serves as cryptographic certification that the Consent Orchestrator has verified the user's permission to share that specific information for that specific session.

#### Consent Token Structure

The `consent_token` is a **JWS-signed JWT** issued by the Global Agent. It is not round-tripped -- it flows one direction (Global Agent → Local Agent) as proof that travels with the data.

**JWT Claims:**

| Claim         | Type     | Description |
| :----         | :----    | :---- |
| iss           | String   | Global Agent origin (matches `act_origin`). |
| sub           | String   | The `act_session_id` this consent is bound to. |
| iat           | Number   | Issued-at timestamp (Unix epoch). |
| categories    | String[] | Data categories the user consented to share (e.g., `["dietary", "budget", "accessibility"]`). |
| consent\_type | String   | `implicit` (user clicked the link) or `explicit` (user was prompted). |

**Example decoded payload:**

```json
{
  "iss": "agent.google.com",
  "sub": "sess_98765abc",
  "iat": 1750000000,
  "categories": ["dietary", "budget"],
  "consent_type": "explicit"
}
```

**Verification**: The Local Agent validates the JWS signature using the Global Agent's public key from `https://[act_origin]/.well-known/act-jwks.json` (same keys used for context verification in §6.3). Keys can be cached.

**Storage**: The Local Agent MAY store the signed `consent_token` as a compliance receipt, demonstrating that data was received with verified user consent. The token is self-contained and can be verified offline at any later point.

### 6.2 The "Cookie-Free" Evolution

ACT replaces the "guessing" model of traditional web tracking with explicit, conversation-driven communication:

* **Intent over ID**: The Local Agent focuses strictly on the "what" (user intent) rather than the "who" (Personal Identifiable Information).  
* **Zero-Knowledge Start**: Because personalization is driven by a server-to-server pull, no tracking cookies are required to "remember" a user's search criteria. This eliminates the need for third-party tracking pixels to maintain state.  
* **Session De-identification**: The `act_session_id` is ephemeral. Once the conversation ends, the link between the Global Agent's user identity and the Local Agent's visitor is severed, preventing long-term profile building.

### 6.3 Security & Verification

ACT requires bidirectional trust: the Local Agent must trust the context it receives, and the Global Agent must trust the feedback it receives.

#### Context Authenticity (Global → Local)

To prevent malicious actors from spoofing user intent or preferences, Local Agents MUST verify the source of the context.

* **Trust Root**: Global Agents shall publish their public keys as a **JWK Set** (RFC 7517) at a standardized location: `https://[act_origin]/.well-known/act-jwks.json`. The JWK Set supports key rotation via the `kid` (Key ID) field.  
* **Signing**: The Global Agent SHOULD sign the Context Response using JWS Compact Serialization. The response body remains clean JSON; the signature is sent in an `ACT-Signature` response header. The header includes the `kid` so the Local Agent knows which key to verify with.  
* **Verification**: The Local Agent validates the `ACT-Signature` against the Global Agent's public key (fetched from the JWK Set endpoint, cacheable). This confirms that the intent data is authentic and has not been tampered with in transit. This is particularly important for safety-critical constraints (e.g., food allergies, accessibility needs).

#### Feedback Authenticity (Local → Global)

To prevent spoofed feedback (fake conversions, poisoned reputation scores), the Global Agent MUST verify that feedback originates from the Local Agent that received the context.

* **Feedback Token**: The Global Agent issues a cryptographically signed, session-scoped `feedback_token` as part of the Context Response (§4). Only the entity that successfully pulled the context possesses this token.
* **Verification**: The Global Agent validates the token signature and confirms it is bound to the `act_session_id` before accepting any feedback.
* **Why not Local Agent PKI?**: Requiring every website to publish signing keys would raise the adoption barrier. The feedback token bootstraps trust from the existing handshake -- no additional infrastructure is needed from the Local Agent side.

#### Link Integrity

The `act_callback_url` and `act_origin` parameters are visible in the URL. If a man-in-the-middle modifies both to point to a malicious server, the Local Agent would pull context from the attacker and validate against the attacker's keys. This is not ACT-specific -- it is a general link-integrity problem that applies to any URL. Mitigations:

* The Global Agent serves links over HTTPS, preventing in-transit tampering.
* The user sees the destination domain in the browser address bar (unchanged by ACT parameters).
* Local Agents SHOULD maintain an allowlist of trusted `act_origin` domains and reject context from unknown origins.

### 6.4 Identity & User IDs

ACT intentionally excludes user identifiers from the protocol. The context payload carries **intent**, not **identity**.

* **No cross-site tracking**: A protocol-level user ID would allow Local Agents to correlate users across sites, enabling the same cross-site profiling that ACT is designed to avoid.
* **Ephemeral sessions**: The `act_session_id` is short-lived and single-use. Adding a persistent user ID would defeat this property.
* **Existing login flows**: If a Local Agent needs to identify a returning customer, the user logs in through the site's normal authentication -- separate from ACT.
* **Scoped opt-in data**: When the user explicitly wants to share identity-adjacent information (e.g., a loyalty program number), this can be included as an optional field in `preferences` with the user's consent, rather than as a protocol-level identifier.

#### SSO Identity Hints (Optional Extension)

When the Global Agent and Local Agent share an existing SSO relationship, ACT can carry **identity hints** that enable seamless authentication -- without ACT itself becoming an identity protocol. The user MUST explicitly consent to sharing identity (per §6.1, this is "Explicit Sensitive Data").

The Context Response MAY include an optional `identity_hints` object:

```json
"identity_hints": {
  "protocol": "oidc",
  "idp_issuer": "https://accounts.google.com",
  "login_hint": "opaque_hint_abc123",
  "id_token_hint": "eyJhbGciOiJSUzI1NiIs..."
}
```

| Field           | Type   | Description |
| :----           | :----  | :---- |
| protocol        | String | Authentication protocol. Currently only `oidc` is defined. |
| idp\_issuer     | String | The OIDC Issuer URL of the Identity Provider. |
| login\_hint     | String | An opaque hint the IDP can use to identify the user. MUST NOT contain plaintext PII. |
| id\_token\_hint | String | (Optional) A signed OIDC ID Token the Local Agent can use for silent authentication. |

**Configuration A -- Global Agent is the IDP:**
When the Global Agent provider is the Local Agent's SSO Identity Provider (e.g., user is on Gemini, site uses "Sign in with Google"), the Global Agent can issue an `id_token_hint` directly. The Local Agent passes this to the Global Agent's OIDC authorization endpoint to silently authenticate the user -- no popup or redirect required. This does not create a new tracking vector since the identity relationship already exists on both sides.

**Configuration B -- Shared third-party IDP:**
When both agents use the same external IDP (e.g., both use "Sign in with Apple"), the Global Agent includes the `idp_issuer` and a `login_hint`. The Local Agent initiates its own OIDC flow with that IDP using the hint. The IDP handles the user mapping internally. Note that OIDC uses **pairwise subject identifiers** -- the same user gets different `sub` values at different relying parties -- so the Global Agent cannot and does not pass a cross-party user ID. ACT carries the hint, the IDP carries the identity.

**Privacy invariants:**
* ACT never carries a resolved user identity -- only hints for an external IDP to resolve.
* The Local Agent authenticates the user through its own OIDC flow, not through ACT.
* If the SSO flow fails or the user declines, the ACT session continues normally without identity -- intent transfer is unaffected.

## 7 Design Alternatives & Decisions

During the development of this spec, the following alternatives were explored and rejected:

* **Raw Data in URL (Rejected):** Initially considered passing Base64 encoded context in the URL. This was rejected due to URL length limits (2KB in many browsers), lack of security (data exposure in logs), and the inability to update context dynamically.  
* **Mutual Secret Pre-sharing (Rejected):** Requiring every website to have a pre-shared API key with every Global Agent is not scalable. We moved to a **Public Key Infrastructure (PKI)** model to allow any Local Agent to verify any Global Agent.  
* **Client-side POST Handoff (Rejected):** Using a form POST to redirect the user is disruptive to the user experience and often blocked by modern browser security settings. The "Pull" model is more robust and allows for asynchronous data retrieval.

### Comparison with Existing Methods

The ACT protocol occupies a unique space between general web browsing and deep application integration.

| Feature | ACT Protocol | WebMCP (Model Context Protocol) | A2A (Agent-to-Agent) |
| :---- | :---- | :---- | :---- |
| **User Experience** | **Visible & Interactive.** The user clicks a link and browses the site. | **Headless/Background.** The agent performs actions on the user's behalf. | **Delegated.** One agent gives a task to another; the user may not be present. |
| **Control** | The user remains in control of the browser. | Global Agent controls the browser/DOM. | Local Agent controls the task execution. |
| **Data Flow** | **Context Transfer.** The site is "pre-filled" with intent. | **Direct Manipulation.** The agent clicks buttons and scrapes data. | **API Orchestration.** Structured data exchange between services. |
| **Privacy** | **Consent-Based.** The user chooses to click and share. | **Permission-Based.** The user gives Agent access to "act as me." | **Contract-Based.** Secure handshake between two systems. |
| **Adoption Barrier** | **Low.** Works with existing HTTP infrastructure. | **Medium.** Requires specific webMCP enabled browsers. | **High.** Requires complex multi-agent orchestration. |

**Key Differentiator:** 

**WebMCP** (and similar Model Context Protocols) essentially treats the website as a "headless tool" for the agent to manipulate directly via automation (RPAs or browser-control layers).

In contrast, **ACT** is about **enhancing the human-led browsing experience** by ensuring the website "knows" what the human and the Global Agent were just discussing.

### Future Considerations

* **ACT Discovery**: A mechanism for Local Agents to advertise ACT support (e.g., `/.well-known/act.json`) would allow Global Agents to prioritize ACT-enabled sites. Currently, ACT degrades gracefully -- non-ACT sites simply ignore the `act_` parameters, so discovery is not required for the protocol to function.

## 8 Examples

### Hotel Search

This end-to-end example follows a **User** looking for a specific **Hotel Stay** through a **Global Agent**, transitioning to a **Local Agent** (https://www.google.com/search?q=BoutiqueHotels.com).

```mermaid
%%{init: {
  &#39;theme&#39;: &#39;base&#39;,
  &#39;themeVariables&#39;: {
    &#39;primaryColor&#39;: &#39;#f6f8fa&#39;,
    &#39;primaryTextColor&#39;: &#39;#24292f&#39;,
    &#39;primaryBorderColor&#39;: &#39;#d0d7de&#39;,
    &#39;lineColor&#39;: &#39;#8c959f&#39;,
    &#39;secondaryColor&#39;: &#39;#ffffff&#39;,
    &#39;tertiaryColor&#39;: &#39;#f6f8fa&#39;
  }
}}%%
sequenceDiagram
    participant User
    participant GlobalAgent as Global Agent (AI)
    participant Browser as Hotel Deluxe site
    participant LocalAgent as Local Agent (Hotel Server)

    User->>GlobalAgent: "Find a boutique hotel in Paris for June 12-15. <br>I need a suite with an Eiffel Tower view, <br>wheelchair accessible, under $600/night."
    GlobalAgent->>User: "I found the perfect Suite at Hotel-De-Luxe. <br>[View Details & Book]"
    
    User->>Browser: Click Link
    Note over Browser: URL: hotel-deluxe.com/book?act_session_id=paris-442&<br>act_callback_url=https://api.agent.ai/act
    
    Browser->>LocalAgent: GET /book (Initial Request)
    
    rect rgb(240, 248, 255)
    Note right of LocalAgent: [Step 1] Context Retrieval (Pull)
    LocalAgent->>GlobalAgent: GET https://api.agent.ai/act?session_id=paris-442
    GlobalAgent-->>LocalAgent: JSON (Intent + Constraints + consent_token + feedback_token)
    Note right of LocalAgent: Session: Created → Active
    end

    Note right of LocalAgent: Verify consent_token JWT signature
    LocalAgent->>Browser: Render page pre-filled with Suite, <br>Dates, and Accessibility filters.
    User->>Browser: Clicks "Confirm Reservation"

    rect rgb(245, 245, 245)
    Note right of LocalAgent: [Step 2] Feedback Loop (Push)
    LocalAgent->>GlobalAgent: POST https://api.agent.ai/act<br>Authorization: Bearer feedback_token
    end
    
    LocalAgent->>Browser: Show Confirmation Page
```

#### 1 User search in Global Agent

Our user is searching for a boutique hotel in Paris for June 12-15. They are looking for a suite with an Eiffel Tower view, wheelchair accessible, under $600/night.

The global agent finds one or more options, and the user clicks on one of the options, a link to the Hotel Deluxe site to book a room. The link is an ACT link \- regular link with the act params.

#### 2 Payload: Context Retrieval (The Pull)

The Hotel Deluxe (Local Agent server) receives the `act_session_id` and calls the Global Agent to understand the user needs and constraints \- and learns about the "Eiffel Tower view" preference and the "Accessibility" needs.

The Hotel Deluxe site then shows the user the right room options that meet the user preferences and constraints.

**Response from Global Agent:**

JSON

```
{
  "act_session_id": "paris-442",
  "intent": "book_boutique_hotel_paris",
  "payload": {
    "@context": "https://schema.org",
    "@type": "LodgingReservation",
    "checkinDate": "2026-06-12",
    "checkoutDate": "2026-06-15",
    "numAdults": 2,
    "lodgingUnitType": {
      "@type": "QualitativeValue",
      "name": "Suite",
      "additionalProperty": {
        "@type": "PropertyValue",
        "name": "View",
        "value": "Eiffel Tower"
      }
    }
  },
  "constraints": {
    "accessibility": ["wheelchair_accessible"],
    "max_price_per_night": 600,
    "currency": "USD"
  },
  "preferences": {
    "vibe": "boutique",
    "location_priority": "central"
  },
  "consent_token": "eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJhcGkuYWdlbnQuYWkiLCJzdWIiOiJwYXJpcy00NDIiLCJpYXQiOjE3NTAwMDAwMDAsImNhdGVnb3JpZXMiOlsiYWNjZXNzaWJpbGl0eSIsImJ1ZGdldCIsImxvZGdpbmciXSwiY29uc2VudF90eXBlIjoiZXhwbGljaXQifQ.signature",
  "feedback_token": "ft_hmac_paris442_x7k9"
}
```

The `consent_token` decodes to:

```json
{
  "iss": "api.agent.ai",
  "sub": "paris-442",
  "iat": 1750000000,
  "categories": ["accessibility", "budget", "lodging"],
  "consent_type": "explicit"
}
```

The Local Agent verifies the JWT signature against the Global Agent's public key at `https://api.agent.ai/.well-known/act-jwks.json` and stores the token as a compliance receipt. The `consent_type: "explicit"` indicates the user was prompted before sharing accessibility needs.

---

#### 3 Payload: Feedback Loop (The Push)

The user successfully completes the booking. The Local Agent notifies the Global Agent so it can "congratulate" the user and update its own memory of the trip.

**Request to Global Agent:**

```
POST https://api.agent.ai/act
Authorization: Bearer ft_hmac_paris442_x7k9
```

```json
{
  "act_session_id": "paris-442",
  "action": "conversion",
  "intent_match": true,
  "description": "User successfully booked the Eiffel Tower Suite for June 12-15.",
  "payload": {
    "@context": "https://schema.org",
    "@type": "LodgingReservation",
    "reservationNumber": "FR-77821",
    "reservationStatus": "https://schema.org/Confirmed",
    "totalPrice": "1750.00",
    "priceCurrency": "USD",
    "underName": {
      "@type": "Person",
      "name": "User Name"
    }
  }
}
```

---

#### 4 Why this works for both Agents

* **The Local Agent** didn't have to ask the user "What dates?" or "Do you need a suite?" It received the `LodgingReservation` schema and instantly filtered its database to show only the matching rooms.  
* **The Global Agent** received a structured `reservationNumber` back. If the user later says to the AI, *"When is my Paris check-in?"*, the Global Agent already has the answer in its logs because of the Feedback Loop.

