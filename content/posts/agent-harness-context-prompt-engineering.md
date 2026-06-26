+++
title = "Agent Harness Engineering vs Context Engineering vs Prompt Engineering"
description = "Reliable agents come from three separate jobs: prompt engineering, context engineering, and the harness around the model. One incident-response agent shows where each fits and how to tell which layer broke."
date = 2026-03-21
updated = 2026-06-26

[taxonomies]
tags = ["ai", "agents", "llm", "prompt-engineering", "context-engineering", "reliability", "production"]

[extra]
toc = true
comment = false
+++

I build agents for a living, and most weeks someone hands me a "prompt problem" that turns out to be something else. The wording was fine. The model never saw the right data, or the runtime let it do something it should never have been allowed to do.

A reliable agent comes from three separate jobs, and the prompt is one of them:

1. **Prompt engineering**: the instructions the model follows.
2. **Context engineering**: the information the model sees at each step.
3. **Agent harness engineering**: the deterministic code around the model that lets it act safely and leaves an audit trail.

Most agent failures get diagnosed in the wrong layer. A team rewrites the prompt for a week when the context was stale. They swap models when the harness had no retries, no approval gate, and no eval loop. You fix the wrong thing and the bug stays.

The industry has been drifting this way for a while. Anthropic's guidance on [building effective agents](https://www.anthropic.com/engineering/building-effective-agents) pushes simple, composable workflows before full autonomy. OpenAI's [March 2025 agent tooling release](https://openai.com/index/new-tools-for-building-agents/) baked tools and orchestration into the core platform. LangChain's 2025 [State of Agent Engineering](https://www.langchain.com/state-of-agent-engineering) surveyed practitioners and found **57%** already running agents in production, **89%** with agent-specific observability, and **32%** naming quality as the top blocker to shipping. Those are not demo numbers. Teams are fighting reliability now.

One example runs through the whole post: an incident-response agent for a payments team.

## The connected example: an incident response agent

Your on-call engineer gets this:

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

This one task touches all three layers:

- the **prompt** defines the agent's role and output contract
- the **context** determines what evidence the agent sees
- the **harness** governs tool use, control flow, approvals, retries, tracing, and evaluation

When the agent says something wrong, you need to know which layer failed.

## A simple mental model

Picture the agent as a worker you just hired.

- Prompt engineering is the job description.
- Context engineering is the briefing packet on their desk.
- Harness engineering is the process around the work: approvals, logs, the rule that says nobody ships to prod alone.

Give a strong engineer a vague job description and they drift. Give them a sharp one but the wrong briefing packet and they make confident, wrong calls. Give them both and no process around the work, and they can still take down prod by themselves. Good agent design works as a stack. Miss a layer and the other two cannot cover for it.

## 1. Prompt engineering

Prompt engineering is how you write the instructions.

A good prompt answers:

- What role is the model playing?
- What task is it trying to complete?
- What output format should it use?
- What should it refuse to do?
- When should it ask for clarification?
- How should it behave around uncertainty?

For the incident agent, a prompt might look like this:

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

That prompt does real work. Role, constraints, output shape, all set.

It also does nothing about the part you need next. It can't fetch the deploy diff, name the services that changed, read the current flag state, or recall that the last lookalike incident was a gRPC timeout and not Redis saturation. No wording trick adds a fact the prompt never had. Plenty of teams stall right here, rewriting the prompt over and over to patch a hole that lives in the system, not the sentence.

### What prompt engineering is good at

Prompt engineering works best for:

- role and tone definition
- output shaping
- local reasoning guidance
- refusal and escalation rules
- schema adherence
- lightweight task decomposition

It earns its keep when the task fits inside one model call.

### Common prompt engineering mistakes

1. **Using prompts to encode business logic that should live in code**
   - Example: "Never roll back on Fridays unless the deployment touched checkout and the error rate exceeds 3.2%." That belongs in policy code or validators.

2. **Stuffing too much policy into one giant system prompt**
   - Long prompts turn brittle. They get harder to review, version, and debug.

3. **Treating prompt edits like the only tuning lever**
   - If the agent lacks the right evidence, no wording trick fixes it.

4. **Expecting prompts to be a security boundary**
   - "Do not reveal secrets" is not a real control when the secrets already sit in the context or the tools are over-permissioned.

### Where prompt engineering stops

It still matters. Clear instructions cut variance, structured output keeps your downstream parser from choking, and a couple of well-chosen examples carry a tricky task. Once you move from chatbot behavior to a production agent, though, it is the smallest of the three jobs.

## 2. Context engineering

Context engineering is the design of what the model sees at each step.

It covers the whole working set the model decides from:

- system instructions
- current user request
- conversation history
- retrieved documents
- tool outputs
- memory and summaries
- state from earlier steps
- permissions and environment metadata

A prompt asks what to tell the model. Context asks what the model needs to know right now, for this step, to do it well.

### The incident response context packet

A trimmed context packet for the same incident:

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

This packet beats ten more prompt tweaks. Hand the model these facts and it can reason. Starve it and the best model on the market falls back to guessing.

### Context engineering is the new bottleneck

Back in 2024 the bet was that bigger context windows would make this easy. They made it harder.

A bigger window just lets you pour more in. It does nothing for the real question: which facts matter for this step, in what order, at what level of detail? So context engineering now covers:

- retrieval strategy
- summarization and compaction
- short-term and long-term memory design
- provenance tagging
- freshness controls
- permission-aware data access
- token budgeting
- state handoff across tools and turns

### What good context engineering looks like

For the incident agent, strong context engineering means a few things.

#### 1. Relevance over volume

Do not dump 200 pages of runbooks, 50 log lines from random pods, and six months of alerts into the window. Include:

- the current incident summary
- the last deploys touching the affected service
- the most relevant runbook section
- summarized logs tied to the spike window
- only the feature flags on this request path
- a compact memory note from the last similar incident

#### 2. Structured context

A model reasons better over labeled evidence than over one wall of text. This is better:

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

Every context item should answer two questions:

- Where did this come from?
- How fresh is it?

Skip provenance and you can't debug a wrong answer. Skip freshness and the agent reasons over yesterday's truth.

#### 4. Memory that compresses, not pollutes

Long-running agents need memory. Most of what they could store is noise. Keep the durable signals:

- this service often fails after config pushes
- team prefers mitigation before rollback for tax provider incidents
- prior incident 1842 was misdiagnosed because sampled logs came from the wrong shard

Drop the noisy fragments that keep resurfacing and confusing later steps.

### MCP and why it matters

The [Model Context Protocol](https://modelcontextprotocol.io/) matters because it turns context sources into explicit interfaces instead of ad hoc string-stuffing.

Forget the protocol branding. The point is that each source becomes its own module:

- one server for incident tickets
- one for dashboards
- one for runbooks
- one for deployment metadata

You get interoperability, and better than that, you can debug. Inspect which source you hit, what it returned, and what the agent saw.

### Common context engineering mistakes

1. **Confusing more context with better context**
   - A large window can still be noisy, stale, contradictory, or badly ordered.

2. **Mixing trusted instructions with untrusted retrieved content**
   - A runbook and a random incident comment should not carry the same authority.

3. **Passing raw tool output straight into the next step**
   - Raw JSON, stack traces, and long logs usually need summarizing or filtering first.

4. **Ignoring context drift over long runs**
   - The working state goes inconsistent after a few turns unless you compact and re-ground it.

Prompt engineering is instruction quality. Context engineering is evidence quality.

## 3. Agent harness engineering

This is the layer nobody talks about, and the one that decides whether you sleep through the night.

By agent harness engineering I mean the deterministic runtime around the model:

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

The prompt and the context shape what the model says. The harness decides what the system is allowed to do with it.

### A minimal harness for the incident agent

A simplified sketch:

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

None of that is glamorous. Drop any one piece and the next incident finds it. Without the harness, the same model will call tools in loops, fire duplicate actions, blow past your latency budget, act outside approval policy, hand your app unstructured output it can't use, and leave you nothing to debug afterward.

### What harness engineering includes in practice

#### 1. Tool contracts

Each tool needs a schema, a timeout, an auth boundary, a retry policy, and a side-effect model. For the incident agent:

- `get_latency_metrics(service, window)` is read-only
- `get_recent_deploys(service)` is read-only
- `toggle_feature_flag(name, value)` is high-risk
- `rollback_deploy(service, version)` is high-risk and approval-gated

That split belongs in code, not in a line of the prompt.

#### 2. Control flow

The harness decides whether the agent runs as:

- a single-shot call
- a fixed workflow
- a planner-executor loop
- an orchestrator-worker system
- a human-in-the-loop workflow

Anthropic's guidance gets this right: start with the simplest workflow that works. Plenty of problems teams label "agentic" come apart with a small workflow, a retriever, and one or two model calls.

#### 3. Safety and permissions

If the agent can restart services, change flags, or roll back deploys, the harness has to enforce:

- approval rules
- idempotency
- blast-radius limits
- rate limits
- user-scoped permissions
- audit logs

A prompt can ask for safe behavior. Only the harness enforces it.

#### 4. Evaluation harnesses

The harness includes the test rig, not just the live loop. For the incident agent, you want replayable eval cases:

| Eval case | What should happen |
| --- | --- |
| latency spike after deploy, evidence points to retry regression | identify deploy as likely cause, recommend rollback, request approval |
| latency spike with no recent deploy, evidence points to third-party dependency | recommend mitigation, do not blame deploy |
| ambiguous evidence across two services | lower confidence, request more data |
| rollback tool unavailable | escalate instead of hallucinating action |
| prompt-injected runbook note | ignore the malicious note and hold the control policy |

Those cases are harness work, not prompt work.

#### 5. Observability and replay

When a user asks "Why did the agent recommend a rollback?" you should be able to pull up:

- the exact prompt version
- the exact context packet
- the tool calls and outputs
- the validators that ran
- the approval decision
- the final structured response

Observability is table stakes now. It is the gap between a demo and a system you can run on-call.

## The differences, side by side

| Dimension | Prompt engineering | Context engineering | Agent harness engineering |
| --- | --- | --- | --- |
| Primary question | What should the model do? | What should the model know now? | How should the system execute safely and reliably? |
| Main artifact | Instructions, examples, schemas | Retrieval, memory, summaries, state packets | Orchestrators, tool wrappers, validators, evals, traces |
| Failure mode | Ambiguous or poorly formatted output | Wrong answer due to stale, missing, or noisy evidence | Loops, unsafe actions, hidden failures, no audit trail |
| Typical owner | Prompt author, app engineer | Retrieval, data, and application engineers | Platform, infra, agent, or reliability engineers |
| Best use | Role definition and output shaping | Supplying relevant evidence and state | Enforcing control flow, safety, and production quality |
| Security posture | Guidance only | Data exposure and trust boundary management | Real enforcement point for permissions and approvals |

Three jobs, three outcomes. The prompt shapes what the model says, the context shapes what it reasons over, the harness shapes what the system does.

## One failure, three possible root causes

Say the agent blames the tax provider for the latency spike and recommends disabling a feature flag, when the real cause was a synchronous retry regression in the last deploy. That same wrong answer can come from three different places.

### Prompt failure

The prompt never asked for evidence ranking or uncertainty.

Symptom:
- the answer sounds far too certain
- no line between evidence and hypothesis

Fix:
- require an explicit evidence section
- require confidence labels
- require missing-data callouts

### Context failure

The deploy diff and the gRPC retry logs never reached the working context.

Symptom:
- the answer is plausible but built on incomplete evidence
- the model never references the most important change

Fix:
- improve retrieval ranking
- add deploy-aware context builders
- compact logs into targeted summaries

### Harness failure

The tool graph called `get_feature_flags()` early but never forced a deploy comparison before recommending mitigation.

Symptom:
- the agent followed a weak execution path
- no validator enforced "compare recent deploys before recommending a rollback or flag change"

Fix:
- add required-step validators
- change the orchestration order
- improve eval coverage

One wrong answer, three root causes. Name the layer wrong and you fix the wrong thing.

## What changed in 2025 and 2026

A few shifts over the last two years made these distinctions matter more.

### 1. Tools moved into the default platform stack

With the [Responses API and built-in tools](https://openai.com/index/new-tools-for-building-agents/), tool use is a default now, not a framework add-on. Once attaching a tool is trivial, the bottleneck moves to controlling what those tools do. That is harness work.

### 2. Production adoption exposed operational pain

LangChain's 2025 [State of Agent Engineering](https://www.langchain.com/state-of-agent-engineering) reported:

- **57%** with agents already in production
- **89%** with agent-specific observability
- **32%** naming quality as the top blocker

Those teams are past novelty and deep into reliability.

### 3. Simplicity became a best practice again

The best trend of 2025 was teams walking back over-engineered agent abstractions. Anthropic's [building effective agents](https://www.anthropic.com/engineering/building-effective-agents) argument holds up: prefer simple workflows, prompt chaining, routing, and deterministic steps until you have evidence that a freer agent loop earns its complexity. That is a harness lesson wearing an architecture costume.

### 4. Context became an explicit system concern

MCP and the patterns around it formalize something many teams learned the hard way: context is infrastructure. It has sources, freshness, permissions, schemas, auditability, and failure modes. See it that way and you stop treating it like a long string-concatenation trick.

## How I would build the incident agent today

Starting from scratch, I would go in this order.

### Step 1: define the harness first

- read-only tools for metrics, logs, deploys, runbooks
- high-risk tools for rollback and feature-flag changes
- a human approval gate for any write action
- a structured output schema
- step tracing and replay
- 10 to 20 replayable evaluation scenarios

### Step 2: build a context pipeline

- incident summary builder
- deploy diff summarizer
- metrics and logs summarizer
- runbook retriever with provenance
- memory note for similar past incidents
- context compaction after each major step

### Step 3: add a disciplined prompt

- clear role
- evidence versus hypothesis separation
- explicit uncertainty handling
- minimal verbosity rules
- strict action-approval language

The order is the point. I do not open with the perfect prompt. I decide what the system can do, what it can see, and what it must never do without a human. The prompt comes last.

## A practical checklist

Run this on your next agent.

### Prompt engineering checklist

- [ ] Does the prompt define role, goal, and output format clearly?
- [ ] Does it say when to ask for clarification or report uncertainty?
- [ ] Does it keep out policy logic that belongs in code?
- [ ] Does it state tool-use intent without pretending to enforce permissions?

### Context engineering checklist

- [ ] Does each step get the minimum relevant evidence, not the maximum available text?
- [ ] Are context items structured, fresh, and source-labeled?
- [ ] Are trusted instructions kept apart from untrusted retrieved content?
- [ ] Do you compact history and keep only durable memory?

### Harness engineering checklist

- [ ] Are tool permissions, side effects, and approval rules enforced in code?
- [ ] Do you have retries, timeouts, and fallback behavior?
- [ ] Can you replay a run with prompt, context, tool outputs, and validations?
- [ ] Do you have eval cases for ambiguity, tool failure, and malicious context?

## The takeaway

If you are serious about agents, stop treating everything as prompt engineering.

The prompt is the instruction layer. The context decides whether the model reasons over the right evidence. The harness decides whether the thing behaves like software you can run on-call or a demo that wows once and pages you at 2am.

The pattern that lasts:

- keep prompts clear
- keep context relevant
- keep the harness deterministic where it has to be

Get those three right and the agent stops being a demo you babysit and starts being software you can hand a real action.
