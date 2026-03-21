+++
title = "Agent Harness Engineering vs Context Engineering vs Prompt Engineering"
description = "A practical guide to the three disciplines that matter when building reliable AI agents, with a connected incident-response example and recent industry trends."
date = 2026-03-21
updated = 2026-03-21

[taxonomies]
tags = ["ai", "agents", "llm", "prompt-engineering", "context-engineering", "reliability", "production"]

[extra]
toc = true
comment = false
+++

If you are building agents in 2026, prompt engineering is still necessary, but it is no longer sufficient.

The teams shipping reliable agents are working across three different layers:

1. **Prompt engineering** - what instructions the model should follow
2. **Context engineering** - what information the model should see at each step
3. **Agent harness engineering** - what deterministic software surrounds the model so it can act safely, reliably, and observably

That distinction matters because many "agent failures" are diagnosed in the wrong layer. Teams blame the prompt when the real issue is stale context. They blame the model when the real issue is a weak runtime harness with no retries, no approvals, and no evaluation loop.

The industry is moving in exactly this direction. Anthropic's guidance on [building effective agents](https://www.anthropic.com/engineering/building-effective-agents) argues for simple, composable workflows before full autonomy. OpenAI's [March 2025 agent tooling release](https://openai.com/index/new-tools-for-building-agents/) pushed built-in tools and orchestration primitives into the core platform. LangChain's 2025 [State of Agent Engineering](https://www.langchain.com/state-of-agent-engineering) reported that **57%** of respondents already had agents in production, **89%** had agent-specific observability, and **32%** said quality was the top blocker to shipping. In other words, the market is moving from "how do I make the model answer nicely?" to "how do I operate the whole system reliably?"

This post explains the three disciplines, shows where each one fits, and uses one connected example throughout: an **incident response agent** for a payments team.

## The Connected Example: An Incident Response Agent

Suppose your on-call engineer receives this request:

> Checkout latency jumped from 420 ms to 2.3 s right after the 18:05 deployment. Find the likely cause, recommend an action, and only roll back if I approve it.

You want an agent that can:

- read the alert payload
- inspect dashboards and logs
- compare recent deploys
- read service runbooks
- check feature flags
- suggest a root cause
- propose a rollback or mitigation
- ask for approval before taking high-impact actions

This is a good example because it exposes all three layers clearly:

- The **prompt** defines the agent's role and output contract
- The **context** determines what evidence the agent actually sees
- The **harness** governs tool use, control flow, approvals, retries, tracing, and evaluation

If the agent says something wrong, you need to know which layer failed.

## A Simple Mental Model

Think of an agent system like this:

- **Prompt engineering** is the job description
- **Context engineering** is the briefing packet
- **Harness engineering** is the operating system around the worker

A strong worker with a vague job description underperforms.
A strong worker with the wrong briefing packet makes bad decisions confidently.
A strong worker with a good briefing packet, but no process controls, can still cause damage.

That is why good agent design is not one trick. It is a stack.

## 1. Prompt Engineering

Prompt engineering is the design of the instructions you give the model.

At its best, prompt engineering answers questions like:

- What role is the model playing?
- What task is it trying to complete?
- What output format should it use?
- What should it refuse to do?
- When should it ask for clarification?
- How should it behave around uncertainty?

For the incident response agent, a prompt might look like this:

```
You are an incident response assistant for the payments platform.

Your goals:
1. Identify the most likely cause of the incident from available evidence.
2. Distinguish evidence from hypothesis.
3. Recommend the lowest-risk next action.
4. Never execute a rollback, restart, or feature-flag change without explicit human approval.

Output format:
- Summary
- Evidence
- Most likely cause
- Recommended next step
- Confidence: low | medium | high
- Approval required: yes | no

If evidence is incomplete, say what information is missing.
Do not invent metrics, logs, or deployment facts.
```

That prompt is useful. It sets role, constraints, and structure. It reduces ambiguity.

But notice what it does **not** do.

It does not fetch the deploy diff. It does not know which services were touched. It does not know the current feature flag state. It does not know your rollback policy. It does not know whether the last similar incident was caused by Redis saturation or by a bad gRPC timeout change.

That is where many teams stop too early. They keep rewriting the prompt to compensate for missing system design.

### What Prompt Engineering Is Good At

Prompt engineering works best for:

- role and tone definition
- output shaping
- local reasoning guidance
- refusal and escalation rules
- schema adherence
- lightweight task decomposition

It is especially valuable when the task is mostly bounded inside one model call.

### Common Prompt Engineering Mistakes

1. **Using prompts to encode business logic that should live in code**
   - Example: "Never roll back on Fridays unless the deployment touched checkout and the error rate exceeds 3.2%." That belongs in policy code or validators.

2. **Stuffing too much policy into one giant system prompt**
   - Long prompts become brittle. They are harder to review, version, and debug.

3. **Treating prompt edits like the only tuning lever**
   - If the agent lacks the right evidence, no wording trick fixes it.

4. **Expecting prompts to be a security boundary**
   - "Do not reveal secrets" is not a real control if secrets are already present in the context or tools are over-permissioned.

### The Right Level of Respect for Prompt Engineering

Prompt engineering still matters. Clear instructions reduce variance. Structured output reduces downstream breakage. Well-chosen examples improve reliability for tricky tasks.

But prompt engineering is the **smallest** of the three disciplines once you move from chatbot behavior to production agents.

## 2. Context Engineering

Context engineering is the design of what the model sees at each step.

This is broader than a prompt. It includes the full working set that shapes the model's decision:

- system instructions
- current user request
- conversation history
- retrieved documents
- tool outputs
- memory and summaries
- state from earlier steps
- permissions and environment metadata

Prompt engineering asks, "What should I tell the model?"
Context engineering asks, "What should the model know **right now** to do the next step well?"

That difference is subtle but decisive.

### The Incident Response Context Packet

Here is a simplified context packet for the same incident:

```json
{
  "incident": {
    "service": "checkout-api",
    "started_at": "2026-03-21T18:06:14Z",
    "symptoms": [
      "p95 latency rose from 420ms to 2.3s",
      "error rate unchanged",
      "CPU flat"
    ]
  },
  "recent_changes": [
    {
      "time": "2026-03-21T18:05:02Z",
      "change": "deploy checkout-api@a81d9c",
      "summary": "switched inventory call to synchronous retry path"
    },
    {
      "time": "2026-03-21T17:42:11Z",
      "change": "feature flag enabled for 10% traffic",
      "summary": "dynamic tax enrichment"
    }
  ],
  "runbook": {
    "rollback_policy": "Rollback requires human approval unless revenue loss exceeds threshold and paging severity is SEV-1.",
    "known_failure_modes": [
      "inventory gRPC deadline mismatch",
      "tax provider tail latency",
      "Redis connection pool exhaustion"
    ]
  },
  "evidence": {
    "logs_summary": "Slow requests correlate with retries to inventory-service.",
    "metrics_summary": "Latency spike is isolated to checkout-api to inventory-service edge.",
    "feature_flags": [
      "dynamic-tax-enrichment=10%"
    ]
  }
}
```

This packet does more useful work than ten extra prompt tweaks.

With the right context, the model can reason over facts. With the wrong context, even a strong model has no chance.

### Context Engineering Is the New Bottleneck

In 2024, many teams thought bigger context windows would make context engineering less important. In practice, the opposite happened.

Bigger windows made it easier to include more information, but they did not solve the harder question: **which information is relevant for this step, in this order, with this level of detail?**

That is why context engineering now includes:

- retrieval strategy
- summarization and compaction
- short-term and long-term memory design
- provenance tagging
- freshness controls
- permission-aware data access
- token budgeting
- state handoff across tools and turns

### What Good Context Engineering Looks Like

For the incident agent, strong context engineering means:

#### 1. Relevance over volume

Do not dump 200 pages of runbooks, 50 log lines from random pods, and the last six months of alerts into the window.

Instead include:

- the current incident summary
- the last deploys touching the affected service
- the most relevant runbook section
- summarized logs tied to the spike window
- only the feature flags affecting this request path
- a compact memory note from the last similar incident

#### 2. Structured context

Models reason better over well-labeled evidence than over one large text blob.

For example, this is better:

```
## Metrics
- p95 latency: 2.3s
- error rate: unchanged
- affected edge: checkout-api -> inventory-service

## Deploy diff summary
- changed retry mode from async fallback to synchronous retry
- deployed at 18:05 UTC

## Relevant runbook rule
- rollback requires human approval
```

than this:

```
Here are some logs and docs and alerts and random notes from previous incidents...
```

#### 3. Freshness and provenance

Every context item should answer at least two questions:

- Where did this come from?
- How fresh is it?

Without provenance, debugging becomes impossible.
Without freshness, the agent can reason over obsolete facts.

#### 4. Memory that compresses, not pollutes

Long-running agents need memory, but not all memory is useful.

Good memory stores durable signals like:

- this service often fails after config pushes
- team prefers mitigation before rollback for tax provider incidents
- prior incident 1842 was misdiagnosed because sampled logs came from the wrong shard

Bad memory stores noisy fragments that keep reappearing and confusing future steps.

### MCP and Why It Matters

The rise of the [Model Context Protocol](https://modelcontextprotocol.io/) matters because it pushes context engineering toward explicit interfaces instead of ad hoc prompt stuffing.

The important idea is not the protocol label itself. The important idea is that context sources become modular:

- one server for incident tickets
- one for dashboards
- one for runbooks
- one for deployment metadata

That improves interoperability, but more importantly it improves **debuggability**. You can inspect which context source was consulted, what it returned, and what the agent saw.

### Common Context Engineering Mistakes

1. **Confusing more context with better context**
   - Large windows can still be noisy, stale, contradictory, or badly ordered.

2. **Mixing trusted instructions with untrusted retrieved content**
   - A runbook and a random incident comment should not have the same authority.

3. **Passing raw tool output directly into the next step**
   - Raw JSON, stack traces, or long logs often need summarization or filtering first.

4. **Ignoring context drift over long runs**
   - The agent's working state can become inconsistent after several turns unless you compact and re-ground it.

If prompt engineering is about instruction quality, context engineering is about evidence quality.

## 3. Agent Harness Engineering

This is the least discussed layer, but it is often the most important one.

By **agent harness engineering**, I mean the deterministic runtime around the model:

- orchestration loops
- tool registration and wrappers
- retries and timeouts
- validators
- approval gates
- state transitions
- logging and tracing
- evaluation harnesses
- fallback behavior
- cost and latency controls
- permission boundaries

Prompt engineering tells the model what to do.
Context engineering tells the model what it knows.
Harness engineering determines **how the system actually runs**.

### A Minimal Harness for the Incident Agent

Here is a simplified sketch:

```python
while not state.done:
    context = build_context(state)
    response = model.run(system_prompt=SYSTEM_PROMPT, context=context)

    if response.requests_tool:
        tool_result = run_tool_with_timeout_and_authz(response.tool_call)
        state = update_state_with_tool_result(state, tool_result)
        continue

    if response.proposes_high_impact_action:
        require_human_approval(response)
        state = mark_waiting_for_approval(state)
        continue

    validation = validate_output(response)
    if not validation.ok:
        state = repair_or_escalate(state, validation)
        continue

    state = finalize(state, response)
```

Nothing about that harness is glamorous. All of it matters.

Without it, the same model can:

- call tools in loops
- issue duplicate actions
- exceed latency budgets
- operate outside approval policy
- return unstructured output your app cannot use
- become impossible to debug after an incident

### What Harness Engineering Includes in Practice

#### 1. Tool contracts

Each tool needs a clear schema, timeout, auth boundary, retry policy, and side-effect model.

For the incident agent:

- `get_latency_metrics(service, window)` is read-only
- `get_recent_deploys(service)` is read-only
- `toggle_feature_flag(name, value)` is high-risk
- `rollback_deploy(service, version)` is high-risk and approval-gated

This separation should not live only in a prompt. It should live in code.

#### 2. Control flow

The harness decides whether the agent runs as:

- a single-shot call
- a fixed workflow
- a planner-executor loop
- an orchestrator-worker system
- a human-in-the-loop workflow

Anthropic's recent guidance is directionally right here: start with the simplest workflow that works. Many problems that teams label "agentic" are better solved with a small workflow, a retriever, and one or two model calls.

#### 3. Safety and permissions

If the agent can restart services, change flags, or roll back deploys, the harness must enforce:

- approval rules
- idempotency
- blast-radius limits
- rate limits
- user-scoped permissions
- audit logs

A prompt can *request* safe behavior. The harness must *enforce* it.

#### 4. Evaluation harnesses

This is a crucial and underappreciated point.

An agent harness is not only the runtime. It is also the test rig around the runtime.

For the incident response agent, you want replayable evaluation cases like:

| Eval case | What should happen |
| --- | --- |
| latency spike after deploy, evidence points to retry regression | identify deploy as likely cause, recommend rollback, request approval |
| latency spike with no recent deploy, evidence points to third-party dependency | recommend mitigation, do not blame deploy |
| ambiguous evidence across two services | lower confidence, request more data |
| rollback tool unavailable | escalate instead of hallucinating action |
| prompt-injected runbook note | ignore malicious note and preserve control policy |

These evaluation cases are not prompt engineering. They are part of the harness engineering discipline.

#### 5. Observability and replay

If a user asks, "Why did the agent recommend a rollback?" you should be able to inspect:

- the exact prompt version
- the exact context packet
- the tool calls and outputs
- the validators that ran
- the approval decision
- the final structured response

That is why observability has become table stakes in production agent systems. It is how you move from demos to operations.

## The Differences, Side by Side

| Dimension | Prompt engineering | Context engineering | Agent harness engineering |
| --- | --- | --- | --- |
| Primary question | What should the model do? | What should the model know now? | How should the system execute safely and reliably? |
| Main artifact | Instructions, examples, schemas | Retrieval, memory, summaries, state packets | Orchestrators, tool wrappers, validators, evals, traces |
| Failure mode | Ambiguous or poorly formatted output | Wrong answer due to stale, missing, or noisy evidence | Loops, unsafe actions, hidden failures, no audit trail |
| Typical owner | Prompt author, app engineer | Retrieval, data, and application engineers | Platform, infra, agent, or reliability engineers |
| Best use | Role definition and output shaping | Supplying relevant evidence and state | Enforcing control flow, safety, and production quality |
| Security posture | Guidance only | Data exposure and trust boundary management | Real enforcement point for permissions and approvals |

If you remember only one thing, remember this:

> Prompt engineering shapes behavior. Context engineering shapes reasoning. Harness engineering shapes execution.

## One Failure, Three Possible Root Causes

Let us say the incident agent wrongly concludes that the tax provider caused the latency spike and recommends a feature-flag disable, even though the real issue was a synchronous retry regression introduced in the last deploy.

That failure might come from three different places:

### Prompt failure

The prompt failed to require evidence ranking or uncertainty reporting.

Symptom:
- the answer sounds overly certain
- no distinction between evidence and hypothesis

Fix:
- require explicit evidence section
- require confidence labels
- require missing-data callouts

### Context failure

The deploy diff and gRPC retry logs never made it into the working context.

Symptom:
- the answer is plausible but based on incomplete evidence
- the model never references the most important change

Fix:
- improve retrieval ranking
- include deploy-aware context builders
- compact logs into targeted summaries

### Harness failure

The tool graph called `get_feature_flags()` early, but never required a deploy comparison step before recommending mitigation.

Symptom:
- the agent followed a weak execution path
- no validator enforced "compare recent deploys before recommending rollback or flag change"

Fix:
- add required step validators
- change orchestration order
- improve evaluation coverage

The same bad answer can emerge from three different engineering problems. That is why naming the layer correctly matters.

## What Changed in 2025 and 2026

Several shifts made these distinctions more important, not less.

### 1. Tools moved into the default platform stack

With the [Responses API and built-in tools](https://openai.com/index/new-tools-for-building-agents/), tool use is no longer a niche framework feature. It is mainstream platform behavior.

That raises the importance of harness engineering because once tools are easy to attach, control quality becomes the bottleneck.

### 2. Production adoption exposed operational pain

As noted earlier, LangChain's 2025 [State of Agent Engineering](https://www.langchain.com/state-of-agent-engineering) reported:

- **57%** with agents already in production
- **89%** with agent-specific observability
- **32%** naming quality as the top blocker

Those numbers describe a market that has moved past novelty. Teams are now wrestling with reliability, not just prototypes.

### 3. Simplicity became a best practice again

One of the healthiest trends in 2025 was the move away from over-engineered agent abstractions. Anthropic's [building effective agents](https://www.anthropic.com/engineering/building-effective-agents) argument is worth taking seriously: prefer simple workflows, prompt chaining, routing, and deterministic steps until you have evidence that a freer agent loop is necessary.

That is really a harness lesson disguised as an architecture lesson.

### 4. Context became an explicit system concern

MCP and related patterns matter because they formalize a truth many teams learned the hard way: context is infrastructure.

It has sources, freshness, permissions, schemas, auditability, and failure modes. Once you see context that way, you stop treating it like a long string concatenation trick.

## How I Would Build the Incident Agent Today

If I were building this agent from scratch, I would start in this order:

### Step 1: Define the harness first

- read-only tools for metrics, logs, deploys, runbooks
- high-risk tools for rollback and feature flag changes
- human approval gate for any write action
- structured output schema
- step tracing and replay
- 10 to 20 replayable evaluation scenarios

### Step 2: Build a context pipeline

- incident summary builder
- deploy diff summarizer
- metrics and logs summarizer
- runbook retriever with provenance
- memory note for similar past incidents
- context compaction after each major step

### Step 3: Add a disciplined prompt

- clear role
- evidence versus hypothesis separation
- explicit uncertainty handling
- minimal verbosity rules
- strict action approval language

Notice the ordering.

I would not start by writing the perfect prompt. I would start by deciding what the system can do, what it can see, and what it is not allowed to do.

## A Practical Checklist

Use this checklist when you build your next agent.

### Prompt engineering checklist

- [ ] Does the prompt define role, goal, and output format clearly?
- [ ] Does it say when to ask for clarification or report uncertainty?
- [ ] Does it avoid carrying policy logic that belongs in code?
- [ ] Does it define tool-use intent without pretending to enforce permissions?

### Context engineering checklist

- [ ] Does each step receive the minimum relevant evidence, not the maximum available text?
- [ ] Are context items structured, fresh, and source-labeled?
- [ ] Are trusted instructions separated from untrusted retrieved content?
- [ ] Do you compact history and preserve only durable memory?

### Harness engineering checklist

- [ ] Are tool permissions, side effects, and approval rules enforced in code?
- [ ] Do you have retries, timeouts, and fallback behavior?
- [ ] Can you replay a run with prompt, context, tool outputs, and validations?
- [ ] Do you have evaluation cases for ambiguity, tool failure, and malicious context?

## The Takeaway

If you are serious about building agents, stop treating everything as prompt engineering.

Prompt engineering matters, but it is only the instruction layer.
Context engineering determines whether the model reasons over the right evidence.
Agent harness engineering determines whether the system behaves like production software instead of an impressive demo.

The durable pattern is straightforward:

- keep prompts clear
- keep context relevant
- keep the harness deterministic where it must be

That is how you build agents that are not only clever, but trustworthy.
