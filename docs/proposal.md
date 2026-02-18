# Agent Context Transfer (ACT) Protocol v0.3

## 1 Introduction

The **Agent Context Transfer (ACT)** protocol defines a standardized mechanism for a **Global Agent** (e.g., a general-purpose AI assistant) to securely hand off a user’s conversation state, intent, and constraints to a **Local Agent** (e.g., a website-specific AI or service).

The goal of ACT is to eliminate the "cold start" problem when a user clicks a link from an AI assistant to a website, ensuring the website understands exactly what the user is looking for without requiring the user to repeat themselves. In contrast to automation-heavy protocols, ACT is specifically designed to **enhance the human-led browsing experience** by ensuring the website "knows" the nuances of the conversation the user was just having with their Global Agent, allowing for a seamless transition from dialogue to digital storefront or service.

## 2 Terminology

* **Global Agent:** The initiating AI platform (e.g., Gemini, ChatGPT, Claude).  
* **Local Agent:** The destination AI or automated service residing on a specific web domain.  
* **ACT Session:** A temporary, conversation-scoped link between the two agents.  
* **Context Pull:** The process by which a Local Agent fetches structured data from the Global Agent.

## 3 The Handoff (URL Parameters)

When a Global Agent directs a user to a website, it appends a minimal set of parameters to the destination URL. This provides the Local Agent with the "keys" needed to retrieve the full context.

### **Parameters**

| Parameter | Description | Example |
| :---- | :---- | :---- |
| act\_session\_id | A unique, ephemeral identifier for the conversation. | sess\_98765abc |
| act\_origin | The identifier/domain of the Global Agent. | agent.google.com |
| act\_callback\_url | The endpoint where the Local Agent fetches context. | https://api.global-ai.com/v1/act |

**Example URL:**

```
https://italy-eats.com/order?act_session_id=sess_98765abc&act_origin=agent.google.com&act_callback_url=https://api.global-ai.com/v1/act
```

**Implementation Note:** To protect user privacy, browsers and servers SHOULD NOT log act\_ prefixed parameters in plain-text server access logs.

## 4 Context Retrieval (The "Pull" Model)

Upon page load, the Local Agent initiates a server-to-server GET request to the `act_callback_url` using the `act_session_id`.

### 4.1 The Secure Handshake

Upon receiving a user via an ACT-enabled link, the Local Agent's server initiates a **HTTP GET** request to the `act_callback_url`.

* **Discovery**: The Local Agent extracts the `act_session_id` and `act_callback_url` from the URL parameters.  
* **Authentication**: The Local Agent identifies itself in the request header (e.g., via User-Agent or a domain-specific identifier).  
* **Authorization**: The Global Agent validates that the `act_session_id` is active, has not expired (short TTL), and was originally intended for the requesting `act_origin`.

**Example Context Response:**

```json
{ 
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
  "consent_token": "ct_v1_signed_9921", 
}
```

| Field | Type | Description |
| :---- | :---- | :---- |
| intent | String | A concise summary of what the user is trying to achieve. |
| preferences | Object | Key-value pairs of user desires (e.g., style, brand, urgency). |
| constraints | Object | Hard requirements that must be met (e.g., allergies, size, budget). |
| payload | Object | **Schema.org** compatible object, representing the constraints and preferences. |
| consent\_token | String | A cryptographic proof that the user authorized this specific data transfer. |

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
  "intent_match": "failure", 
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
| action | Enum | engagement (user interacting) or conversion (user completed intent). |
| intent\_match | Boolean | **True** if the Global Agent's context was accurate to the site's capability. **False** if the site couldn't fulfill the specific constraints (e.g., "We don't sell size 15"). |
| payload | Object | **Schema.org** compatible object, giving more details about the engagement / conversion |

## 6 Privacy & Consent

### 6.1 The Consent Orchestrator

The Global Agent serves as the primary **Consent Orchestrator** for the user journey. Unlike traditional web tracking where consent is often managed by invisible scripts, ACT makes consent explicit and centralized:

* **Role**: The Global Agent is responsible for auditing the user's conversation and identifying sensitive data (e.g., health, finance, or PII).  
* **Execution**: Before releasing an Intent Package via the callback URL, the Global Agent must ensure appropriate consent.  
  * **Implicit Intent:** General parameters (e.g., "blue sneakers") are consented to by the user's action of clicking the link.  
  * **Explicit Sensitive Data:** For constraints such as medical allergies or precise location, the Global Agent must trigger a UI prompt: *"Share your allergy profile with \[Local Agent\]?"*  
* **The Token as Certification**: When a Local Agent receives a context response, the existence of that data serves as a cryptographic certification that the Consent Orchestrator has verified the user's permission to share that specific information for that specific session.

### 6.2 The "Cookie-Free" Evolution

ACT replaces the "guessing" model of traditional web tracking with explicit, conversation-driven communication:

* **Intent over ID**: The Local Agent focuses strictly on the "what" (user intent) rather than the "who" (Personal Identifiable Information).  
* **Zero-Knowledge Start**: Because personalization is driven by a server-to-server pull, no tracking cookies are required to "remember" a user's search criteria. This eliminates the need for third-party tracking pixels to maintain state.  
* **Session De-identification**: The `act_session_id` is ephemeral. Once the conversation ends, the link between the Global Agent's user identity and the Local Agent's visitor is severed, preventing long-term profile building.

### 6.3 Security & Verification

To prevent malicious actors from spoofing user intent or preferences, Local Agents MUST verify the source of the context.

* **Trust Root**: Global Agents shall publish their public keys at a standardized location: `https://[origin]/.well-known/act-pubkey.json`.  
* **Verification**: The Context Payload response SHOULD be signed (e.g., using JWS), allowing the Local Agent to confirm that the intent data is authentic and originates from the stated Global Agent. This ensures that constraints (like food allergies) cannot be tampered with by a man-in-the-middle.

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

## 8 Examples

### Hotel Search

This end-to-end example follows a **User** looking for a specific **Hotel Stay** through a **Global Agent**, transitioning to a **Local Agent** (https://www.google.com/search?q=BoutiqueHotels.com).

```mermaid
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
    GlobalAgent-->>LocalAgent: Return JSON (Intent + Schema.org + Constraints)
    end

    LocalAgent->>Browser: Render page pre-filled with Suite, <br>Dates, and Accessibility filters.
    User->>Browser: Clicks "Confirm Reservation"

    rect rgb(245, 245, 245)
    Note right of LocalAgent: [Step 2] Feedback Loop (Push)
    LocalAgent->>GlobalAgent: POST https://api.agent.ai/act (Order Success)
    end
    
    LocalAgent->>Browser: Show Confirmation Page
```

![][image1]

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
  "consent_token": "sig_9901_verified"
}
```

---

#### 3 Payload: Feedback Loop (The Push)

The user successfully completes the booking. The Local Agent notifies the Global Agent so it can "congratulate" the user and update its own memory of the trip.

**Request to Global Agent:**

JSON

```
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

[image1]: <data:image/png;base64,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>