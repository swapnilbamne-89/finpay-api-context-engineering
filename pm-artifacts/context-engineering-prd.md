# Product Requirements Document: FinPay LLM Context Layer

**Document type:** Product Requirements Document  
**Product:** FinPay API — LLM-Native Context Engineering Layer  
**Version:** 1.0  
**Status:** Approved  
**Author:** Swapnil Bamne — Senior AI Product Manager  
**Date:** June 2026  

---

## 1. Executive Summary

FinPay's existing REST API documentation was designed for human developers. As AI coding assistants (GitHub Copilot, Claude Code, Cursor) and autonomous payment agents become primary consumers of API documentation, the existing documentation structure creates measurable problems: hallucinated API behaviour, incorrect error handling, misuse of financial primitives, and compliance gaps in agent-generated code.

This PRD defines the **FinPay LLM Context Layer** — a structured set of context engineering artifacts (`llms.txt`, `llms-full.txt`, `CLAUDE.md`, MCP tool definitions, disambiguation guides, and agent guardrails) designed to make the FinPay API reliably consumable by LLMs and autonomous agents.

---

## 2. Problem Statement

### 2.1 The Documentation Gap

When a developer asks Claude or Copilot to "write a FinPay payment integration," the AI model draws on:
1. Its training data (which may include outdated FinPay docs or competitors' APIs)
2. General payments API patterns (correct principles, wrong specifics)
3. Whatever is in the current context window

Without a structured context layer, models frequently:
- Hallucinate endpoint paths that don't exist
- Use floating-point arithmetic for monetary amounts (a critical bug pattern)
- Confuse FinPay-specific concepts like `order` vs `payment` vs `transaction`
- Generate code that polls for payment status instead of using webhooks
- Omit webhook signature verification (a security vulnerability)
- Mix up `settlement` (FinPay→merchant) and `payout` (merchant→vendor)

### 2.2 The Agentic Gap

As payment workflows become more automated (AI agents processing refunds, triggering payouts, responding to fraud signals), the stakes of hallucination increase from "broken demo code" to "wrong amount transferred to wrong recipient." 

Existing documentation provides no:
- Explicit human-in-the-loop checkpoints for financial actions
- Tier-structured guardrails (what requires confirmation vs what is autonomous)
- Agent-specific error handling patterns (retry logic calibrated to `error.retryable`)
- MCP tool definitions for standardised agent-API integration

### 2.3 The Compliance Gap

Indian payments operate under RBI PA Guidelines, PCI-DSS 4.0, and the DPDP Act 2023. Agent-generated code that violates these (e.g., logging PAN, storing CVV, skipping fraud reporting) creates regulatory exposure that the existing documentation does not proactively address for AI consumers.

---

## 3. Goals & Success Metrics

### 3.1 Primary Goal
Reduce hallucination rate and compliance error rate in LLM-generated FinPay integration code.

### 3.2 OKRs

| Objective | Key Result | Target |
|-----------|-----------|--------|
| Reduce hallucination | % of LLM-generated code using correct endpoint paths | > 95% (vs estimated 70% baseline) |
| Improve correctness | % using integer paise (not float rupees) for amounts | 100% |
| Improve security | % including webhook signature verification | > 95% (vs estimated 55% baseline) |
| Reduce disambiguation errors | % correctly distinguishing order vs payment vs payout | > 90% |
| Enable agent safety | % of agent-generated payout code that includes human confirmation gate for > ₹10K | 100% |

*Measurement: Evaluated via standardised prompt set against the context-enriched vs baseline documentation. Quarterly review.*

### 3.3 Non-Goals
- Replacing the full human-readable FinPay developer documentation (that remains at docs.finpay.in)
- Adding new API capabilities — this is a documentation/context layer, not a feature build
- Covering non-Indian payment rails (cross-border out of scope)

---

## 4. User Personas

| Persona | Context Layer Use Case | Primary Artifact |
|---------|----------------------|-----------------|
| **AI Coding Assistant User** — Developer using Claude Code or Copilot to scaffold a FinPay integration | Needs accurate endpoint paths, correct data types, proper error handling patterns | `CLAUDE.md` + `llms-full.txt` |
| **RAG Pipeline Builder** — Platform team embedding FinPay docs into a RAG system for internal developer tools | Needs semantically dense, well-chunked, authoritative context | `llms-full.txt` |
| **LLM Crawler / Indexer** — AI service indexing FinPay docs for training or retrieval | Needs structured index with scope boundaries to prevent mis-indexing | `llms.txt` |
| **Payment Agent Developer** — Engineer building an autonomous payment orchestration agent | Needs MCP tool definitions, guardrails, agent workflow patterns | `mcp/finpay-mcp-spec.md` + `context/guardrails.md` |
| **AI PM / Technical Writer** — Portfolio reviewer assessing this project | Needs to understand the product decisions behind the context engineering choices | This PRD + `pm-artifacts/ai-charter.md` |

---

## 5. Feature Scope

### 5.1 In Scope — v1.0

| Artifact | Description | Priority |
|----------|-------------|----------|
| `llms.txt` | Structured LLM index following llmstxt.org standard. Summary, links, scope boundaries. | P0 |
| `llms-full.txt` | Full authoritative context document. All API behaviour, data models, error codes, compliance constraints. | P0 |
| `CLAUDE.md` | Claude Code context file. Project identity, coding conventions, permission model, guardrails. | P0 |
| `context/disambiguation.md` | Concept disambiguation to prevent hallucination on FinPay-specific terminology. | P1 |
| `context/guardrails.md` | Tiered agent safety constraints. Absolute prohibitions, confirmation gates, auditable autonomy. | P0 |
| `context/agent-patterns.md` | Reusable agentic workflow patterns with failure modes and guardrail checkpoints. | P1 |
| `mcp/finpay-mcp-spec.md` | MCP tool definitions for all write operations and key read operations. | P1 |
| `openapi/finpay-openapi.yaml` | Machine-readable API spec. Consumed by IDE tooling and SDK generators. | P0 |

### 5.2 Out of Scope — v1.0
- Automated context freshness checking (planned v1.5)
- Embedding pipeline integration (planned v2.0)
- Multi-language SDK generation from OpenAPI spec (separate initiative)
- Webhook simulator for agent testing (separate tooling project)

---

## 6. Design Decisions & Rationale

### Decision 1: Two-File LLM Context Strategy (llms.txt + llms-full.txt)
**Decision:** Maintain a concise index (`llms.txt`, ~50 lines) and a comprehensive document (`llms-full.txt`, ~500 lines) rather than a single file.  
**Rationale:** LLM context windows have cost and length constraints. For simple queries ("what's the base URL?"), the index is sufficient. For complex integration tasks, the full document is needed. RAG pipelines can choose their ingestion level based on use case.

### Decision 2: Tiered Guardrail Structure
**Decision:** Guardrails are organised into Tier 1 (absolute), Tier 2 (confirmation required), and Tier 3 (autonomous + logged) rather than a binary allow/deny.  
**Rationale:** Binary rules are either too restrictive (blocking legitimate automation) or too permissive (no nuance). Tiers enable genuine autonomy for low-risk operations while maintaining human oversight for high-stakes financial actions — consistent with the EU AI Act's risk-based approach.

### Decision 3: Paise-First Everywhere
**Decision:** All monetary amounts in the context layer are expressed in paise (integer), never rupees (float).  
**Rationale:** LLMs trained on financial documentation learn both patterns. Explicit paise-first language and a prohibition on float arithmetic in `CLAUDE.md` reduces the probability of generating float-based amount code to near zero.

### Decision 4: MCP Tool Definitions Are Safety-Scoped
**Decision:** Each MCP tool definition includes not just parameters but explicit "When NOT to use" guidance and safety notes.  
**Rationale:** Tool descriptions are consumed directly by agent orchestrators to decide when to invoke a tool. "When NOT to use" guidance prevents the most common misuse patterns — particularly using read tools in write contexts (e.g., polling `get_payment` instead of using webhooks).

### Decision 5: Disambiguation as a Standalone Document
**Decision:** Concept disambiguation is a dedicated document (`context/disambiguation.md`), not embedded in other docs.  
**Rationale:** Disambiguation information needs to be findable by both human reviewers (who want to understand the mental model) and RAG pipelines (which benefit from a dedicated semantic chunk for disambiguation queries). Embedding it in endpoint docs would fragment its utility.

---

## 7. Compliance Alignment

| Requirement | How Context Layer Addresses It |
|-------------|-------------------------------|
| PCI-DSS 4.0 — No CVV storage | Guardrail G2 (absolute prohibition); CLAUDE.md prohibition; noted in `llms-full.txt` Section 8 |
| PCI-DSS 4.0 — Audit logging | Tier 3 guardrails mandate logging for autonomous financial actions |
| RBI PA Guidelines — Fraud reporting | Fraud report pattern in `agent-patterns.md`; `finpay_report_fraud` MCP tool with timing requirements |
| DPDP Act 2023 — Data minimisation | CLAUDE.md prohibition on logging PAN/Aadhaar; `llms-full.txt` Section 8 covers tokenisation requirement |
| RBI — Human oversight of automated payments | Tier 1 and Tier 2 guardrails encode human-in-the-loop requirements |

---

## 8. Open Questions

| Question | Owner | Target |
|----------|-------|--------|
| Should `llms-full.txt` be chunked into per-section files for better RAG retrieval? Test semantic search precision. | AI PM | Q3 2026 |
| What is the optimal context window strategy for Claude Projects — full file injection vs retrieval? | Engineering | Q3 2026 |
| Should guardrail tiers be machine-readable (JSON schema) rather than prose, for enforcement at orchestration layer? | Engineering + AI PM | Q4 2026 |
| Freshness: How do we detect when `llms-full.txt` drifts from the live API spec? Automated diff needed. | Engineering | Q3 2026 |

---

*This PRD is a portfolio artifact demonstrating AI PM product thinking applied to context engineering. FinPay is a fictional company.*  
*Author: Swapnil Bamne | June 2026*
