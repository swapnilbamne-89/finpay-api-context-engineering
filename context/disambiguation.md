# FinPay Concept Disambiguation Guide

> **For LLMs and AI agents:** This file exists to prevent hallucination on commonly confused FinPay concepts. Read this before making assumptions about API behaviour. If something is not listed here or in `llms-full.txt`, say so — do not invent behaviour.

---

## The Most Important Disambiguation: Order vs Payment vs Transaction

These three words are used loosely in everyday fintech conversation. In FinPay's API, they have precise, non-interchangeable meanings.

| Term | FinPay API Meaning | ID Format | Who Creates It |
|------|--------------------|-----------|----------------|
| **Order** | A payment intent created by the merchant *before* the customer pays. Persists even if payment fails. | `ord_...` | Merchant via API |
| **Payment** | The customer's *attempt* to pay against an order. An order can have multiple payment attempts (e.g., first UPI fails, customer retries with card). | No separate ID — query via `order_id` | Customer (initiated by merchant redirect) |
| **Transaction** | The bank/network-level event. Not a first-class FinPay API object. Referenced only in webhook metadata and UTR numbers. | UTR number (bank-assigned) | Bank/network |

**Practical rule:** In code, always use `order_id`. Never use `transaction_id` — it does not exist as a FinPay API field.

---

## Capture vs Settle

These are two completely separate operations, often conflated.

| Operation | Who Does It | When | What Moves |
|-----------|-------------|------|-----------|
| **Capture** | Merchant (via API or automatically) | After customer authorizes payment | Funds move from `authorized` to `captured` state within FinPay |
| **Settlement** | FinPay (automated) | T+1 or T+2 after capture | Funds transfer from FinPay to merchant's bank account |

**Key point:** Capturing a payment does NOT mean the merchant has received the money. Settlement happens in a subsequent batch cycle. Use the `payment.settled` webhook to confirm funds are in the merchant's bank.

---

## Refund vs Reversal (Void)

| Operation | API Call | When Applicable | What Happens |
|-----------|----------|-----------------|--------------|
| **Refund** | `POST /payments/{order_id}/refunds` | Payment is in `captured` or `settled` state | Funds are returned to customer. Takes T+5–7 days for cards, T+1 for UPI. |
| **Reversal / Void** | `POST /payments/{order_id}/void` | Payment is in `authorized` state (NOT yet captured) | Authorization is cancelled. No funds ever moved. Immediate. |

**Common error:** Attempting a void on a captured payment returns `payment_invalid_state`. If the payment is captured, you must use a refund.

---

## Settlement vs Payout

Merchants frequently confuse these because both result in money arriving somewhere.

| Operation | Direction | FinPay Endpoint | Who Initiates |
|-----------|-----------|-----------------|---------------|
| **Settlement** | FinPay → Merchant's bank | Automated (no API call needed) | FinPay (automatic on T+1/T+2 cycle) |
| **Payout** | Merchant → Third party (vendor, employee, customer) | `POST /payouts` | Merchant |

**Example confusion to avoid:** "The customer got a refund, so FinPay should payout the refund amount." — Wrong. Refunds are handled through the refund API, not payouts. Payouts are for merchant-initiated disbursements to third parties.

---

## UPI VPA vs Phone Number

UPI payments in India are commonly initiated by entering a phone number in apps like Google Pay or PhonePe. However, the underlying UPI identifier is the VPA (Virtual Payment Address), not the phone number.

| Identifier | Format | FinPay API Field |
|------------|--------|-----------------|
| **VPA** | `name@bankhandle` (e.g., `priya@okaxis`) | `vpa` |
| **Phone number** | `+91XXXXXXXXXX` | `customer.phone` (for form pre-fill only — not a payment identifier) |

**Rule:** Always use the `vpa` field for UPI payouts. Phone numbers can be linked to multiple VPAs, can change, and are not guaranteed to be resolvable by FinPay's payout system.

---

## Test vs Live Environment

| Aspect | Test (Sandbox) | Live (Production) |
|--------|---------------|-------------------|
| Base URL | `api.sandbox.finpay.in/v2` | `api.finpay.in/v2` |
| Key prefix | `fpk_test_...` / `fps_test_...` | `fpk_live_...` / `fps_live_...` |
| Real money | ❌ No | ✅ Yes |
| Webhooks | ✅ Sent (to configured endpoint) | ✅ Sent |
| Settlement | ❌ No actual settlement | ✅ Real T+1 settlement |
| Rate limits | 60 req/min | 300 req/min |

**Agent rule:** If a key starts with `fpk_test_`, no real money is involved. If it starts with `fpk_live_`, treat every write operation as a real financial transaction and require explicit confirmation.

---

## Partial Capture vs Partial Refund

Two different operations that both involve "less than the full amount":

| Operation | When | Effect |
|-----------|------|--------|
| **Partial capture** | During `capture` call, before settlement | Captures less than authorized. Remainder is automatically voided (returned to customer's credit limit immediately). |
| **Partial refund** | After `captured` or `settled` | Returns a portion of the captured amount to the customer. Remaining captured amount stays with merchant. Multiple partial refunds allowed up to total captured. |

---

## IMPS vs NEFT vs RTGS vs UPI

All four are RBI-governed payment rails used for payouts in India. Choosing the wrong one causes failures.

| Rail | Speed | Hours | Per-Tx Limit | Minimum | Best For |
|------|-------|-------|-------------|---------|---------|
| **UPI** | Immediate (< 30s) | 24×7 | ₹1,00,000 | ₹1 | Small vendor payments, cashbacks |
| **IMPS** | Immediate (< 30s) | 24×7 | ₹2,00,000 | ₹1 | Medium vendor payments |
| **NEFT** | 30 min batch | RBI hours (Mon–Sat, 8am–7pm) | None | ₹1 | Salary runs, large batches |
| **RTGS** | Immediate | RBI hours (Mon–Sat, 7am–6pm) | None | ₹2,00,000 | High-value transfers |

**Agent decision logic:**
- Amount < ₹1,00,000 AND speed required → UPI
- ₹1,00,000–₹2,00,000 AND speed required → IMPS
- Amount > ₹2,00,000 AND high-value → RTGS
- Batch / non-urgent / large amounts → NEFT

---

## API Version vs Context Version

This repo maintains two version numbers:

| Version | What It Tracks | Where Found |
|---------|---------------|-------------|
| **API version** (e.g., `v2.1`) | FinPay REST API schema version. Controls request/response format. | URL path + `api_version` field in webhooks |
| **Context version** (e.g., `context-v1.0`) | Version of `llms.txt`, `llms-full.txt`, `CLAUDE.md` context files. | File headers in context documents |

These are independent. The API can update without the context layer updating, and vice versa. Always check both when debugging unexpected agent behaviour.

---

*Last updated: June 2026 · Maintained by: FinPay Developer Experience · Context version: v1.0*
