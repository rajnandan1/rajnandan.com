+++
title = "Stripe's Agentic Commerce Protocol: Building the Rails for AI-Powered Commerce"
description = "A deep dive into Stripe's new Agentic Commerce Protocol (ACP) - how it works, its architecture, and what it means for the future of AI agents making purchases on behalf of users."
date = 2025-11-28
updated = 2025-11-28

[taxonomies]
tags = ["stripe", "ai", "agents", "payments", "mcp"]

[extra]
toc = true
comment = false
+++

Stripe recently announced the Agentic Commerce Protocol (ACP), a new standard designed to enable AI agents to discover merchants, negotiate terms, and complete purchases on behalf of users. This is a significant step toward a future where your AI assistant doesn't just recommend products—it actually buys them for you.

## The Problem ACP Solves

Today's AI agents can do a lot. They can browse the web, summarize content, write code, and even interact with APIs. But when it comes to making purchases, they hit a wall. Here's why:

1. **Trust**: How does a merchant know the AI agent has permission to spend a user's money?
2. **Discovery**: How does an agent find merchants that can fulfill a specific request?
3. **Negotiation**: How do agents and merchants agree on terms, pricing, and fulfillment?
4. **Compliance**: How do you handle authentication, fraud prevention, and regulatory requirements?

ACP addresses all of these by providing a standardized protocol that sits between AI agents, merchants, and payment infrastructure.

## Architecture Overview

ACP introduces three key participants in every transaction:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   AI Agent  │────▶│     ACP     │────▶│  Merchant   │
│  (Claude,   │     │  Protocol   │     │   (Seller)  │
│   GPT, etc) │◀────│   Layer     │◀────│             │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       │                   ▼                   │
       │           ┌─────────────┐            │
       └──────────▶│   Stripe    │◀───────────┘
                   │  (Payment   │
                   │   Rails)    │
                   └─────────────┘
```

### The Agent

The AI agent acts on behalf of the user. It could be a general-purpose assistant like Claude or a specialized agent for travel, shopping, or business procurement. The agent needs to:

- Understand user intent
- Discover relevant merchants via ACP
- Present options and get user confirmation
- Execute the transaction

### The Protocol Layer

ACP defines a standard way for agents and merchants to communicate. This includes:

- **Discovery endpoints**: Merchants register their capabilities, inventory, and terms
- **Negotiation schemas**: Structured formats for quotes, offers, and counter-offers
- **Transaction lifecycle**: States for pending, confirmed, fulfilled, and disputed transactions
- **Permission scopes**: Granular control over what an agent can do

### The Merchant

Merchants expose their catalog and business logic through ACP-compliant endpoints. They define:

- What products/services are available
- Pricing and availability rules
- Fulfillment options
- Refund and cancellation policies

## How a Transaction Works

Let's walk through a real example: booking a restaurant reservation.

### Step 1: User Intent
```
User: "Book me a table for 4 at an Italian restaurant 
       near downtown SF this Saturday at 7pm"
```

### Step 2: Agent Discovery

The agent queries ACP's discovery layer:

```json
{
  "intent": "restaurant_reservation",
  "constraints": {
    "cuisine": "italian",
    "location": {
      "near": "downtown San Francisco",
      "radius_km": 2
    },
    "party_size": 4,
    "datetime": "2024-11-30T19:00:00-08:00"
  },
  "user_id": "usr_abc123",
  "agent_id": "agent_xyz789"
}
```

### Step 3: Merchant Responses

Multiple merchants respond with availability:

```json
{
  "merchant_id": "merch_italian_place",
  "offer": {
    "offer_id": "off_12345",
    "description": "Table for 4 at Trattoria Milano",
    "datetime": "2024-11-30T19:00:00-08:00",
    "deposit_required": {
      "amount": 2500,
      "currency": "usd"
    },
    "cancellation_policy": "free_until_24h_before",
    "valid_until": "2024-11-28T23:59:59-08:00"
  }
}
```

### Step 4: User Confirmation

The agent presents options to the user and gets confirmation. This is a critical step—ACP requires explicit user consent for transactions above certain thresholds or outside pre-authorized categories.

### Step 5: Transaction Execution

Once confirmed, the agent sends a transaction request:

```json
{
  "action": "accept_offer",
  "offer_id": "off_12345",
  "payment_method": "pm_card_visa",
  "user_authorization": {
    "token": "auth_tok_xxx",
    "scope": ["restaurant_reservation"],
    "max_amount": 5000
  }
}
```

### Step 6: Fulfillment

The merchant confirms the reservation and ACP tracks the fulfillment state. If something goes wrong, there's a standardized dispute resolution process.

## The Technical Bits

### Permission Model

ACP uses a hierarchical permission system. Users grant agents specific scopes:

```json
{
  "agent_permissions": {
    "agent_id": "agent_xyz789",
    "scopes": [
      {
        "category": "food_and_dining",
        "max_transaction": 10000,
        "max_daily": 50000,
        "requires_confirmation": false
      },
      {
        "category": "travel",
        "max_transaction": 100000,
        "requires_confirmation": true
      }
    ],
    "expires_at": "2025-11-28T00:00:00Z"
  }
}
```

This means you could let your agent book lunch without asking, but require confirmation for flight bookings.

### MCP Integration

ACP is designed to work with Anthropic's Model Context Protocol (MCP). In fact, Stripe has released an MCP server that exposes ACP functionality as tools that any MCP-compatible agent can use.

```typescript
// Example MCP tool definition for ACP
{
  name: "acp_discover_merchants",
  description: "Find merchants that can fulfill a purchase intent",
  parameters: {
    intent: { type: "string" },
    constraints: { type: "object" },
    location: { type: "object", optional: true }
  }
}
```

This means if you're building an agent using Claude, you can add ACP capabilities by connecting to Stripe's MCP server.

### Idempotency and State Management

E-commerce is messy. Networks fail, retries happen, and state gets out of sync. ACP handles this with:

- **Idempotency keys**: Every transaction request includes a unique key. Retrying with the same key won't create duplicate charges.
- **State machine**: Transactions follow a strict state machine with well-defined transitions
- **Webhooks**: Merchants and agents receive real-time updates about state changes

```
PENDING ──▶ CONFIRMED ──▶ FULFILLED
   │            │             │
   ▼            ▼             ▼
CANCELLED   REFUNDED      DISPUTED
```

## Why This Matters

### For Developers

If you're building AI agents, ACP gives you a standardized way to add commerce capabilities without building payment infrastructure from scratch. You focus on the agent logic; Stripe handles the money movement.

### For Merchants

You get access to a new distribution channel. As AI agents become more prevalent, being discoverable and transactable via ACP could be like being searchable on Google in the early 2000s.

### For Users

The promise is convenience with control. Your agent can handle routine purchases automatically while still requiring your approval for significant decisions.

## Getting Started

Stripe's ACP is currently in beta. To start experimenting:

1. **Sign up for the beta** at stripe.com/acp
2. **Install the MCP server** if you're building MCP-compatible agents:
   ```bash
   npm install @stripe/acp-mcp-server
   ```
3. **Read the spec**: The protocol specification is open and available for review

## The Bigger Picture

ACP is part of a broader trend toward AI agents that can take real-world actions. We're moving from AI that *tells* you what to do to AI that *does* things for you.

But with great power comes great responsibility. The permission model, user confirmation requirements, and dispute resolution mechanisms in ACP reflect an understanding that autonomous spending needs guardrails.

The interesting question isn't whether AI agents will make purchases for us—they will. The question is how we build the infrastructure to make this safe, trustworthy, and beneficial for everyone involved.

Stripe's bet is that whoever builds the standard rails for agentic commerce will be well-positioned in this future. With ACP, they're making that bet concrete.

---

*Disclaimer: This analysis is based on publicly available information about Stripe's ACP announcement. Implementation details may vary as the protocol evolves.*
