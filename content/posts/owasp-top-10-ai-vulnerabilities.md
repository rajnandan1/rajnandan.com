+++
title = "OWASP Top 10 for LLMs: AI Vulnerabilities and How to Defend Against Them"
description = "A practical guide to the OWASP Top 10 for LLM applications, with examples of how AI systems fail in production and the controls that reduce real-world risk."
date = 2026-03-13
updated = 2026-03-13

[taxonomies]
tags = ["ai", "security", "llm", "owasp", "production"]

[extra]
toc = true
comment = false
+++

If you are putting LLMs into production, the main risk is no longer whether the demo works. The main risk is whether the system can be manipulated, over-trusted, or over-permissioned in ways that create a real security incident. That is why the [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) matters. First published in 2023 and updated for 2025, it turns vague AI anxiety into a concrete operating model for defending systems that reason over data, call tools, and influence downstream workflows.

The most important takeaway is simple: most AI incidents are not caused by one magical "AI exploit." They happen when ordinary security weaknesses meet LLM-specific behavior. A prompt becomes an instruction. A retrieved document becomes a hidden control channel. A model output gets treated like trusted code. A helpful assistant gets enough permissions to act like an admin.

This post walks through the 2025 OWASP Top 10 for LLM applications and, more importantly, what to do about each item in practice.

## The 10 Risks at a Glance

| OWASP item | What goes wrong | What to do first |
| --- | --- | --- |
| **LLM01 Prompt Injection** | The model treats attacker-controlled input as instructions | Add input and output filtering, isolate untrusted content, and test with adversarial prompts |
| **LLM02 Sensitive Information Disclosure** | The system leaks secrets, customer data, or internal knowledge | Sanitize data, restrict access, and inspect responses before release |
| **LLM03 Supply Chain** | Models, adapters, datasets, and platforms arrive from untrusted sources | Vet suppliers, verify provenance, scan dependencies, and patch aggressively |
| **LLM04 Data and Model Poisoning** | Training, fine-tuning, or RAG data is tampered with | Track data lineage, enforce change control, and validate against trusted sources |
| **LLM05 Improper Output Handling** | LLM output is executed or rendered without validation | Treat model output as untrusted input and apply context-aware encoding and validation |
| **LLM06 Excessive Agency** | The assistant can call too many tools with too much power | Minimize tool scope, minimize permissions, and require approval for high-risk actions |
| **LLM07 System Prompt Leakage** | Internal instructions reveal secrets or guardrail details | Keep secrets out of prompts and enforce controls outside the model |
| **LLM08 Vector and Embedding Weaknesses** | RAG stores leak data or retrieve poisoned context | Use permission-aware retrieval, validate documents, and log retrieval activity |
| **LLM09 Misinformation** | The system sounds right while being wrong | Ground answers, cross-check outputs, and keep humans in the loop for high-stakes decisions |
| **LLM10 Unbounded Consumption** | Attackers drive cost, latency, or service failure | Add rate limits, quotas, timeouts, and anomaly detection |

## 1. Prompt Injection Is Still the Front Door

Prompt injection remains the number one issue because LLMs are not good at separating trusted instructions from untrusted input. A user can type a malicious instruction directly, or the model can ingest it indirectly from a webpage, PDF, email, or knowledge base document.

That distinction matters:

- **Direct prompt injection** comes from the user's own input
- **Indirect prompt injection** is buried in external content the model reads on the user's behalf

In both cases, the model may ignore its original instructions and follow the attacker's agenda instead. That can lead to data leakage, unsafe guidance, or arbitrary action in connected systems.

### How to reduce prompt injection risk

- Keep system prompts explicit about role, scope, and refusal behavior
- Put an AI gateway or filtering layer in front of the model and on the response path
- Clearly separate trusted instructions from untrusted retrieved content
- Treat external documents as hostile until validated
- Pen test the application with adversarial prompts, including obfuscated or encoded variants

System prompts help, but they are not enough by themselves. The durable control is an external enforcement layer that inspects both the request and the response.

## 2. Sensitive Information Disclosure Is an AI-Specific Data Breach

LLMs are often trained on, connected to, or augmented with valuable data: customer records, health information, internal financials, source code, product plans, and proprietary workflows. Once that data is inside the system, the risk is not just accidental disclosure. It is also extraction over time.

One practical example is the **model inversion** or extraction pattern: an attacker repeatedly queries the system, records the responses, and slowly harvests information that should never have been exposed.

### How to reduce disclosure risk

- Sanitize training and retrieval data before it enters the model pipeline
- Mask, tokenize, or redact high-risk fields such as credentials, account numbers, and personal identifiers
- Apply least-privilege access controls to the model, the data sources, and the users
- Inspect outbound responses for sensitive patterns before returning them
- Publish clear data usage and retention rules so users know what not to submit

This is where conventional security discipline matters. Encryption, access control, data minimization, and configuration hygiene still apply. AI does not replace them.

## 3. Supply Chain Risk Is Bigger Than Dependencies

Traditional software teams already understand package risk. LLM systems expand that problem. Your AI supply chain now includes:

- base models
- fine-tuned models
- LoRA adapters
- embeddings and datasets
- inference frameworks
- plug-ins and tool integrations
- the infrastructure that runs all of the above

The difficult part is scale. Teams often consume models and artifacts they cannot realistically inspect by hand, especially from open repositories.

### How to reduce supply chain risk

- Only use models, datasets, and adapters from suppliers you can verify
- Check provenance, hashes, signatures, and release history
- Maintain an inventory of models, components, and licenses
- Scan dependencies and runtime platforms for known vulnerabilities
- Keep patching policies current for both software and model-serving infrastructure
- Red team third-party models in the context you will actually deploy

The practical lesson is straightforward: do not treat model downloads like neutral content. Treat them like executable supply chain inputs.

## 4. Data and Model Poisoning Corrupts the System at the Source

LLMs are only as trustworthy as the data and artifacts used to shape them. If training data, fine-tuning inputs, or retrieval sources are subtly manipulated, the resulting system may still appear normal while producing biased, unsafe, or strategically wrong answers.

Poisoning is especially dangerous because it can be quiet. A small amount of malicious content can alter outputs only when a specific trigger appears. That makes the system look healthy until the exact failure mode is activated.

### How to reduce poisoning risk

- Track where data came from and how it changed over time
- Require approval and change control for updates to models and knowledge sources
- Validate outputs against trusted references, especially in regulated domains
- Use anomaly detection and adversarial testing on training and retrieval pipelines
- Restrict who can modify model weights, fine-tuning data, and RAG corpora

If your team uses Retrieval-Augmented Generation, remember that the RAG corpus is part of the attack surface. Poisoned documents can become poisoned answers.

## 5. Improper Output Handling Turns a Bad Answer Into a Real Exploit

A common design mistake is to treat model output as safe because it came from "our assistant." That is backwards. The output of an LLM should be handled the same way you would handle untrusted user input.

This matters when model output is:

- rendered in a browser
- sent to an API
- converted into SQL
- written into shell commands
- inserted into file paths
- passed into automation tools

Without validation, you can turn prompt injection into cross-site scripting, SQL injection, remote code execution, or destructive backend actions.

### How to reduce output handling risk

- Treat every model response as untrusted data
- Validate structure and schema before downstream use
- Use parameterized queries for database operations
- Apply context-aware output encoding for HTML, Markdown, and JavaScript
- Enforce content security policies where browser rendering is involved
- Log anomalous outputs and block unsafe execution paths

The safe default is simple: never let raw LLM output execute anything important.

## 6. Excessive Agency Makes Small Errors Catastrophic

LLM systems become more dangerous when they can act, not just answer. The issue is not only whether the model hallucinates. It is also whether the model has enough power to turn a bad decision into a business, financial, or safety event.

Excessive agency typically appears in three forms:

1. **Excessive functionality** - the model can call tools it does not need
2. **Excessive permissions** - the connected tools can do far more than the task requires
3. **Excessive autonomy** - the system performs high-impact actions without human approval

### How to reduce agency risk

- Offer only the minimum set of tools to the model
- Replace open-ended tools with narrow task-specific operations
- Run tool actions in the user's real authorization context
- Enforce authorization in downstream systems, not just in prompts
- Require human confirmation for destructive or sensitive actions
- Rate limit tool execution so one bad chain of reasoning cannot spiral quickly

The more powerful the agent, the more deterministic the surrounding controls must be.

## 7. System Prompts Are Not a Secret Store

Many teams assume the system prompt is hidden and therefore safe. It is not. Attackers can often infer, extract, or approximate prompt contents simply by interacting with the model long enough.

The real problem is not that the prompt text becomes visible. The real problem is storing secrets or policy-critical logic in a place the model can leak or reinterpret.

### How to reduce system prompt leakage risk

- Never place credentials, tokens, connection strings, or private architecture details in prompts
- Keep permission checks and policy enforcement outside the model
- Use prompts for guidance, not as your primary security boundary
- Add output inspection that detects attempts to reveal internal instructions

If a control would fail the moment the prompt leaks, it was never a strong control.

## 8. Vector and Embedding Weaknesses Expand the RAG Attack Surface

RAG improves relevance, but it also adds a new security boundary: embeddings, vector stores, retrieval logic, and document ingestion pipelines.

That creates several practical risks:

- unauthorized retrieval across tenants or user classes
- poisoned documents that steer model behavior
- embedding inversion attacks that recover source information
- conflicting or stale context that degrades answer quality

### How to reduce vector and embedding risk

- Use permission-aware retrieval with strict logical separation between tenants and data classes
- Validate and classify documents before they enter the knowledge base
- Detect hidden or obfuscated content in uploaded files
- Log retrieval events so suspicious access patterns are visible
- Review how new retrieved context changes model behavior, not just factual accuracy

RAG is not just a relevance feature. It is a security-sensitive data pipeline.

## 9. Misinformation Is a Security Problem, Not Just a Quality Problem

LLMs can produce answers that sound credible while being false, unsupported, or dangerously incomplete. In low-stakes use cases that may be embarrassing. In legal, medical, financial, or security-sensitive workflows, it becomes liability.

Misinformation often comes from:

- hallucinations
- incomplete or conflicting context
- poisoned data
- overreliance by users who assume the system is authoritative

### How to reduce misinformation risk

- Ground responses in trusted sources through retrieval and citation patterns
- Add automatic validation for critical outputs
- Require human review in high-stakes domains
- Design the interface to show uncertainty, limitations, and provenance
- Train users to verify important answers instead of accepting fluent text at face value

The goal is not to eliminate every wrong answer. The goal is to stop wrong answers from quietly becoming decisions.

## 10. Unbounded Consumption Leads to Denial of Service and Denial of Wallet

LLMs are resource-intensive by default. That makes them vulnerable to abuse by oversized inputs, repeated requests, complex prompts, and extraction-style querying. The result may look like classic denial of service, but in AI systems it often shows up first as a cost problem.

This is why teams now use the phrase **denial of wallet**. An attacker may not need to crash the system. It may be enough to make it expensive enough to hurt.

### How to reduce consumption risk

- Limit request size, context length, and queued actions
- Apply rate limits, quotas, and per-user budgets
- Set timeouts and throttles for expensive operations
- Monitor token usage, latency spikes, and abnormal request patterns
- Degrade gracefully under load rather than failing completely
- Watch for repeated querying patterns that resemble extraction attempts

Cost controls are security controls when inference is expensive.

## A Practical Defense Stack for LLM Applications

If you want one implementation-oriented view, the controls below do the most work across the full Top 10:

### 1. AI gateway or firewall

Inspect prompts going in and responses coming out. Block obvious prompt injection, redact sensitive data, and enforce policy before the model or the user sees anything.

### 2. Least-privilege architecture

Limit what the model can access, what its tools can do, and what downstream identities are allowed to perform. Keep authorization outside the model.

### 3. Data hygiene and provenance

Know where training, fine-tuning, and RAG data came from. Validate it, version it, and control changes to it.

### 4. Output validation

Do not let the model write raw SQL, HTML, shell commands, or API parameters into real systems without deterministic validation.

### 5. Human approval for high-risk actions

If the action moves money, changes records, deletes data, sends external communications, or impacts health and safety, require human review.

### 6. Monitoring, logging, and anomaly detection

Log prompts, retrieval events, tool calls, and outcomes with enough structure to replay incidents and detect abuse patterns.

### 7. Adversarial testing

Regularly test with direct injections, indirect injections, poisoned content, malicious documents, and tool abuse scenarios. If you do not test these paths, attackers will.

## A Production Checklist

- [ ] Sanitize sensitive data before training, fine-tuning, or retrieval ingestion
- [ ] Treat prompts, retrieved content, and model output as untrusted by default
- [ ] Put approval gates around destructive or externally visible actions
- [ ] Enforce least privilege for tools, plug-ins, APIs, and vector stores
- [ ] Verify model, adapter, and dataset provenance before deployment
- [ ] Validate output before passing it to browsers, databases, shells, or business systems
- [ ] Apply quotas, rate limits, and timeouts to control denial-of-wallet risk
- [ ] Log enough detail to trace incidents across prompt, retrieval, tool, and response steps
- [ ] Red team the application, not just the base model

## Further Reading

If you want the original OWASP write-ups behind this summary, start here:

- [LLM01: Prompt Injection](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM01_PromptInjection.md)
- [LLM02: Sensitive Information Disclosure](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM02_SensitiveInformationDisclosure.md)
- [LLM03: Supply Chain](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM03_SupplyChain.md)
- [LLM04: Data and Model Poisoning](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM04_DataModelPoisoning.md)
- [LLM05: Improper Output Handling](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM05_ImproperOutputHandling.md)
- [LLM06: Excessive Agency](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM06_ExcessiveAgency.md)
- [LLM07: System Prompt Leakage](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM07_SystemPromptLeakage.md)
- [LLM08: Vector and Embedding Weaknesses](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM08_VectorAndEmbeddingWeaknesses.md)
- [LLM09: Misinformation](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM09_Misinformation.md)
- [LLM10: Unbounded Consumption](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications/raw/020595761a4b7b0c3f9cf01a0457b78f9f1e7f9c/2_0_vulns/LLM10_UnboundedConsumption.md)

## The Bottom Line

The OWASP Top 10 for LLMs is useful because it reframes AI security as engineering discipline. Prompt injection, data leakage, poisoned retrieval, untrusted output, and overpowered agents are not abstract research curiosities. They are production failure modes.

The teams that operate LLMs safely will be the teams that treat them neither as magic nor as ordinary software. They will treat them as probabilistic systems wrapped in deterministic controls.

That is the right mental model for building AI that is useful, secure, and still under your control.
