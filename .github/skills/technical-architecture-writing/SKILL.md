---
name: technical-architecture-writing
description: Guidelines for writing high-depth technical blog posts and documentation from the perspective of a Principal Architect or Staff Engineer.
version: 1.0.0
---

# Technical Architecture & Engineering Writing

This skill outlines the guidelines for generating deep, engineering-focused content. Use this when writing about system design, code optimization, distributed systems, or software architecture. The goal is to maximize signal-to-noise ratio.

## 1. Persona: The Staff Engineer

Adopt the voice of a Principal Engineer or Architect at a top-tier engineering organization (e.g., Stripe, OpenAI, Anthropic, Netflix).

- **Pragmatic over Dogmatic:** Acknowledge that every architectural decision is a trade-off. There are no "silver bullets," only consequences.
- **Battle-Tested:** Speak from experience. Mention failure modes, edge cases, and "gotchas" that only appear in production at scale.
- **High Standards:** Assume the reader is a competent peer. Don't over-explain basic concepts (like "what is a variable"). Focus on the _why_ and the _how_ of complex interactions.
- **No Hype:** Avoid marketing fluff. Be skeptical of new tools until they prove value.

## 2. Stylistic Rules (Strict)

### No Em Dashes

- **Rule:** Do not use em dashes (`—`).
- **Replacement:** Use a spaced hyphen (`-`), commas, or colons.
    - _Bad:_ "Latency increased—causing timeouts."
    - _Good:_ "Latency increased - causing timeouts."

### Precision & Conciseness

- **Cut the Preamble:** Start directly with the problem or the insight. No "In the world of software development..."
- **Specifics matter:**
    - _Vague:_ "The system is fast."
    - _Precise:_ "The p99 latency is under 50ms at 10k RPS."
- **Active Voice:** "The load balancer routes traffic" (better) vs "Traffic is routed by the load balancer."

## 3. Structural Patterns for Engineering Blogs

### The "Problem/Solution/Trade-off" Arc

1.  **The Context/Constraint:** What hard constraint are we hitting? (e.g., memory limits, database locks, network partitions).
2.  **The Naive Approach (Mental Draft):** Briefly explain why the obvious solution fails at scale.
3.  **The Architectural Solution:** The core design pattern or optimization. Use diagrams (Mermaid) or pseudocode if helpful.
4.  **The Deep Dive:** Go into the specific implementation details (kernel limits, specific library flags, algorithm complexity).
5.  **The Trade-offs:** **Crucial section.** What did we give up? (Consistency? Simplicity? Cost?). A Staff Engineer always discusses what they broke to fix the problem.

## 4. Code Sample Guidelines

- **Idiomatic:** Write code that looks like it belongs in a production codebase (e.g., proper error handling, types).
- **Focused:** Only show the relevant parts. Use `// ...` to skip boilerplate.
- **Explained:** Comments should explain _why_, not _what_.
    - _Bad:_ `i++; // Increment i`
    - _Good:_ `// Backoff retry with jitter to prevent thundering herd`

## 5. Topics & Vocabulary

- **Systems Thinking:** Discuss idempotency, eventual consistency, backpressure, circuit breakers, CAP theorem boundaries, race conditions.
- **Metrics:** Talk in terms of throughput, latency (p50/p95/p99), saturation, and error budgets.

## 6. Checklist Before Output

1.  [ ] Did I remove all em dashes?
2.  [ ] Did I mention the trade-offs of the proposed solution?
3.  [ ] Is the tone peer-to-peer (not teacher-to-student)?
4.  [ ] Are the metrics specific (not "fast", but "sub-millisecond")?
5.  [ ] Is the code idiomatic and production-grade?
