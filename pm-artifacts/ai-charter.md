# AI Product Charter: FinPay LLM Integration

**Document type:** AI Product Charter  
**Product:** FinPay API — LLM-Native Integration Layer  
**Version:** 1.0  
**Owner:** Swapnil Bamne — Senior AI Product Manager  
**Date:** June 2026  
**Review cycle:** Quarterly

---

## Purpose of This Charter

An AI Product Charter defines the boundaries, values, and governance principles for AI-assisted capabilities in a product. For FinPay's LLM integration layer, the charter answers: *What should AI agents be trusted to do autonomously with payment infrastructure, and what must remain under human control?*

This charter governs:
- All AI coding assistants using `CLAUDE.md` and the FinPay context layer
- All autonomous payment agents using the MCP tool definitions
- All RAG pipelines ingesting `llms.txt` / `llms-full.txt`

---

## 1. Charter Values

### 1.1 Financial Safety Over Convenience
No efficiency gain justifies reducing human oversight of high-value financial actions. When in doubt between a faster autonomous action and a slower confirmed action, choose confirmation.

### 1.2 Explicitness Over Inference
AI agents operating on payment APIs must be told what to do — they should not infer intent from ambiguous instructions. "Pay the vendor" is not sufficient to trigger a payout without explicit amount and recipient confirmation.

### 1.3 Fail Loud, Not Silent
Errors in payment workflows must surface loudly to humans, not be silently swallowed or automatically retried beyond defined limits. A failed payout that the agent silently retried 50 times is worse than one that surfaced immediately.

### 1.4 Compliance Is Non-Negotiable
RBI, PCI-DSS, and DPDP Act requirements are not trade-offs against convenience. They are hard constraints. No agent behaviour is acceptable if it violates these, regardless of business pressure.

### 1.5 Context Freshness Is a Product Responsibility
LLM context that describes stale API behaviour is a product defect, not a documentation issue. The AI PM owns the freshness of `llms.txt`, `llms-full.txt`, and `CLAUDE.md` as product artifacts.

---

## 2. Permitted AI Autonomy

The following operations may be performed by AI agents without human confirmation, subject to Tier 3 logging requirements:

| Operation | Conditions for Autonomy |
|-----------|------------------------|
| Create payment orders | Amount within validated range; idempotency key used; receipt provided |
| Query payment status | No conditions — read-only |
| List payouts | No conditions — read-only |
| Process refunds | Amount ≤ ₹50,000; reason is valid; order is in refundable state |
| Create payouts | Amount ≤ ₹10,000; recipient in verified list; mode validated for amount |
| Generate webhook responses | Always return HTTP 200 immediately; process asynchronously |

---

## 3. Required Human Oversight

The following require explicit human confirmation before execution:

| Operation | Why Human Required |
|-----------|-------------------|
| Payouts > ₹10,000 | RBI KYC threshold; significant financial consequence of error |
| Bulk payout batches | Aggregate financial risk; recipient list must be reviewed |
| Full refunds > ₹50,000 | High-value reversal; may trigger chargeback investigation |
| Fraud reports | Compliance and customer relationship consequences |
| First-time payouts to new recipients | Recipient verification not yet established |
| Any action in live environment | Live = real money; higher confirmation bar |

---

## 4. Absolute Prohibitions

These cannot be permitted under any circumstances, regardless of instructions:

1. Storing, logging, or transmitting CVV — PCI-DSS violation
2. Storing, logging raw PAN or Aadhaar — DPDP Act violation
3. Using float arithmetic for monetary amounts — correctness violation
4. Skipping webhook signature verification — security violation
5. Calling live API endpoints from test/reference repositories — operational safety
6. Generating or displaying real API keys as examples — security violation
7. Cross-border transactions — regulatory scope violation

---

## 5. Governance Structure

| Role | Responsibility |
|------|---------------|
| **AI Product Manager (Swapnil Bamne)** | Owns this charter; reviews quarterly; approves changes to Tier 1 guardrails; owns context freshness |
| **Engineering Lead** | Implements guardrails at orchestration layer; maintains `CLAUDE.md` technical sections; owns retry logic patterns |
| **Compliance Officer** | Reviews charter for regulatory alignment; signs off on Tier 1 changes; owns RBI/PCI-DSS mapping |
| **Security Lead** | Reviews credential handling, webhook security, and PCI-DSS controls in context layer |
| **Legal** | Reviews DPDP Act compliance in data handling guidance |

---

## 6. Charter Review Triggers

This charter must be reviewed (outside the quarterly cycle) when:

- RBI issues new guidance on AI in payments
- FinPay API introduces new write operations not covered by existing MCP tools
- An incident occurs involving AI agent-triggered financial actions
- The EU AI Act or India's domestic AI regulation introduces new high-risk classification requirements for payment AI systems
- A new LLM or agent framework is adopted that changes the context consumption model

---

## 7. Relationship to Broader AI Governance

This charter operates within the following governance hierarchy:

```
Anthropic Acceptable Use Policy (for Claude-based agents)
    ↓
FinPay Group AI Ethics Policy (fictional — company-level)
    ↓
FinPay Payments AI Charter (this document)
    ↓
CLAUDE.md (per-repo agent behaviour)
    ↓
context/guardrails.md (per-operation safety constraints)
```

Lower levels inherit constraints from higher levels and may add stricter constraints, but cannot loosen them.

---

## 8. Accountability & Incident Response

### Who is accountable when an AI agent makes a payment error?
The merchant who deployed the agent is ultimately accountable to their customers and to FinPay. FinPay's context layer reduces the probability of error but does not transfer accountability away from the deploying merchant.

### Incident response for agent-triggered payment errors:
1. **Immediate:** Human takes control of the agent; halt autonomous operations
2. **Within 1 hour:** Assess financial impact; initiate refund/reversal if possible
3. **Within 24 hours:** File fraud report with FinPay if funds were misdirected (RBI requirement)
4. **Within 7 days:** Root cause analysis; identify which guardrail was missing or bypassed
5. **Within 30 days:** Charter and guardrail update to prevent recurrence

---

## 9. Charter Acknowledgement

By deploying an AI agent or coding assistant using the FinPay context layer, the deploying organisation acknowledges:

- They have read and understood this charter and `context/guardrails.md`
- They accept responsibility for implementing Tier 1 and Tier 2 guardrails at their orchestration layer
- They will not instruct agents to bypass any absolute prohibition in Section 4
- They will review this charter when the review triggers in Section 6 are met

---

*This charter is a portfolio artifact demonstrating responsible AI product management applied to fintech. FinPay is a fictional company.*  
*Owner: Swapnil Bamne — Senior AI Product Manager | June 2026*
