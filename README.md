# FinPay API — Context Engineering for LLM-Native Fintech Integrations

> **Portfolio Project · AI Product Manager — Swapnil Bamne**  
> Demonstrates: Context Engineering · LLM.txt · CLAUDE.md · Agent-Friendly API Documentation · MCP Tool Specification

---

## What This Project Is

**FinPay** is a fictional Indian payments API (think Razorpay-meets-Stripe) designed from the ground up to be **LLM-native** — meaning AI coding assistants, autonomous agents, and RAG pipelines can consume its documentation and surface the right information with minimal hallucination and maximum precision.

This repo is not just API docs. It is a **context engineering showcase** — demonstrating the craft of structuring information so that large language models can reason about it reliably.

---

## Why Context Engineering Matters for AI PMs

Most APIs were designed for human developers. Docs are sprawling, contextually ambiguous, and full of prose that search engines love but LLMs trip over. As AI agents take over more of the integration layer — writing SDK calls, handling errors, orchestrating multi-step workflows — **the quality of your context layer becomes a product decision, not a documentation task.**

This project answers: *What does a fintech API look like if you design its documentation for LLMs first?*

---

## Repository Structure

```
finpay-api/
├── README.md                        ← You are here
├── llms.txt                         ← LLM.txt index (Anthropic/llmstxt standard)
├── llms-full.txt                    ← Full expanded context for deep RAG ingestion
├── CLAUDE.md                        ← Claude Code context file (agent behaviour rules)
├── openapi/
│   └── finpay-openapi.yaml          ← OpenAPI 3.1 spec (machine-readable)
├── docs/
│   ├── quickstart.md                ← Human-readable quickstart
│   ├── authentication.md            ← Auth & token management
│   ├── payments.md                  ← Core payments endpoints
│   ├── payouts.md                   ← Payout & settlement endpoints
│   ├── webhooks.md                  ← Webhook event handling
│   ├── errors.md                    ← Error codes & recovery patterns
│   └── compliance.md                ← RBI / PCI-DSS / DPDP compliance notes
├── mcp/
│   └── finpay-mcp-spec.md           ← MCP tool specification for agent integration
├── context/
│   ├── agent-patterns.md            ← Common agentic workflow patterns
│   ├── disambiguation.md            ← Concept disambiguation for LLMs
│   └── guardrails.md                ← What agents must NEVER do with this API
└── pm-artifacts/
    ├── context-engineering-prd.md   ← PRD: why this context layer was built
    └── ai-charter.md                ← AI Product Charter for FinPay LLM integration
```

---

## Key Artifacts

| Artifact | Purpose | AI PM Skill Demonstrated |
|----------|---------|--------------------------|
| `llms.txt` | Structured index for LLM crawlers and RAG pipelines | Context architecture, information hierarchy |
| `llms-full.txt` | Full expanded context blob | RAG chunking strategy, semantic density |
| `CLAUDE.md` | Claude Code / agent behaviour contract | Agentic product thinking, guardrails design |
| `mcp/finpay-mcp-spec.md` | MCP tool definitions for autonomous agents | Emerging standards, tool-use product design |
| `context/guardrails.md` | What agents must not do | Responsible AI, risk management |
| `pm-artifacts/context-engineering-prd.md` | Product rationale | Full PM craft |

---

## The LLM.txt Standard

`llms.txt` follows the [llmstxt.org](https://llmstxt.org) specification — a proposed standard (analogous to `robots.txt`) for telling LLMs what a site contains and how to navigate it. It provides:

- A concise summary of the API
- Structured links to key documentation sections
- Disambiguation hints to reduce hallucination
- Scope boundaries (what this API does NOT do)

---

## The CLAUDE.md Standard

`CLAUDE.md` follows the convention established by Anthropic's Claude Code product. When placed at the root of a repository, Claude reads it automatically at the start of every session. It functions as a **persistent system prompt for the repo** — encoding:

- Project context and constraints
- Coding conventions and API patterns
- What the agent is and is not allowed to do
- Escalation logic for ambiguous decisions

---

## PM Reflection: What I Designed and Why

Context engineering is an emerging discipline that sits at the intersection of technical writing, information architecture, and AI product management. The decisions made in this repo — how to chunk documentation, what to put in `llms.txt` vs `llms-full.txt`, what guardrails to encode in `CLAUDE.md`, how to structure MCP tools — are all product decisions with measurable downstream impact on agent accuracy and safety.

See [`pm-artifacts/context-engineering-prd.md`](pm-artifacts/context-engineering-prd.md) for the full product rationale.

---

## Tech Stack (Reference Implementation)

| Component | Technology |
|-----------|-----------|
| API spec | OpenAPI 3.1 (YAML) |
| Auth | OAuth 2.0 + API key (dual mode) |
| Webhooks | HMAC-SHA256 signed payloads |
| LLM context | llms.txt / llms-full.txt |
| Agent interface | MCP (Model Context Protocol) |
| Compliance | RBI PPI Guidelines · PCI-DSS 4.0 · DPDP Act 2023 |

---

## Author

**Swapnil Bamne** — Senior AI Product Manager & Technical Writer  
13+ years experience · Fintech · Defense & Aerospace · IoT · AI Governance  
[LinkedIn](https://linkedin.com/in/Swapnil_Bamne) · [GitHub](https://github.com/swapnilbamne)

*This is a portfolio project. FinPay is a fictional company. All API endpoints, keys, and data are illustrative.*
