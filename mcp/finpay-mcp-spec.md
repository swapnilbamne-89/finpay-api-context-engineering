# FinPay MCP Tool Specification

> **Model Context Protocol (MCP) Tool Definitions for FinPay API**  
> Version: 1.0 | Base URL: `https://api.sandbox.finpay.in/v2` (test)

This document defines the MCP tool interface for autonomous agents integrating with the FinPay API. Tools are designed to be safe-by-default: read operations are freely available; write operations (creating payments, triggering payouts) require explicit confirmation signals.

---

## Design Principles

1. **Least privilege by default.** Each tool exposes the minimum scope needed for its stated purpose.
2. **Idempotency built in.** All write tools automatically generate and attach `Idempotency-Key`.
3. **Amounts always in paise.** Tools accept and return amounts in paise (integer). Agents must convert from rupees before calling.
4. **Errors are structured.** All error responses include `retryable: boolean` so agents can decide whether to retry.
5. **No live environment in this spec.** This spec targets the sandbox. Agents must not call live endpoints without explicit human confirmation.

---

## Tool Definitions

---

### Tool: `finpay_create_payment_order`

**Description:** Creates a new payment order. Call this when a customer is ready to pay. Returns an `order_id` and `payment_link` to redirect the customer.

**When to use:** User wants to initiate a payment collection from a customer.  
**When NOT to use:** Do not call this to check if a payment exists. Use `finpay_get_payment` instead.

**Inputs:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `amount` | integer | ✅ | Amount in paise (INR × 100). Min: 100. Max: 50000000. |
| `currency` | string | ✅ | Must be `"INR"`. No other currencies supported. |
| `receipt` | string | ✅ | Merchant's internal reference. Max 40 chars. |
| `description` | string | ❌ | Shown to customer. Max 255 chars. |
| `customer_name` | string | ❌ | Pre-fills payment form. |
| `customer_email` | string | ❌ | Pre-fills payment form. |
| `customer_phone` | string | ❌ | E.164 format, e.g. `+919876543210`. |
| `capture_mode` | string | ❌ | `"automatic"` (default) or `"manual"`. |
| `payment_methods` | array | ❌ | Restrict methods, e.g. `["upi", "card"]`. |
| `metadata` | object | ❌ | Merchant key-value pairs. Not processed by FinPay. |

**Outputs:**

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | string | `ord_...` — use this to track the payment |
| `payment_link` | string | URL to redirect customer to FinPay payment page |
| `status` | string | `"created"` on success |
| `expires_at` | string | ISO 8601. Payment link expires after 15 minutes. |

**Example call:**
```json
{
  "tool": "finpay_create_payment_order",
  "input": {
    "amount": 49900,
    "currency": "INR",
    "receipt": "order_june_001",
    "description": "Premium plan — June 2026",
    "customer_email": "user@example.com",
    "capture_mode": "automatic"
  }
}
```

**Safety notes:**
- This tool creates a real payment intent. Confirm with the user before calling in any automated workflow.
- Do NOT call repeatedly in a loop — use the returned `order_id` to check status instead.

---

### Tool: `finpay_get_payment`

**Description:** Retrieves the current status and details of a payment order.

**When to use:** After creating an order, to check if the customer has paid.  
**When NOT to use:** Do not poll this repeatedly — set up webhooks instead.

**Inputs:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `order_id` | string | ✅ | The `ord_...` ID returned from order creation. |

**Outputs:** Full payment order object including `status`, `amount`, `amount_captured`, `payment_method`, `customer`.

**Safety notes:** Read-only. Safe to call freely.

---

### Tool: `finpay_capture_payment`

**Description:** Captures an authorized payment. Only valid when `capture_mode` was `"manual"` and payment is in `authorized` state.

**Inputs:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `order_id` | string | ✅ | The `ord_...` ID to capture. |
| `amount` | integer | ❌ | Paise. If omitted, captures full authorized amount. If provided, enables partial capture. |

**Safety notes:**
- **Irreversible action** — captured funds will be settled to merchant. Confirm before calling.
- If `amount` < authorized amount, the remainder is automatically voided.

---

### Tool: `finpay_create_refund`

**Description:** Issues a refund for a captured payment.

**Inputs:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `order_id` | string | ✅ | The `ord_...` ID to refund. |
| `amount` | integer | ✅ | Paise. Partial refunds allowed. Cannot exceed captured amount minus prior refunds. |
| `reason` | string | ✅ | `"customer_request"` \| `"duplicate"` \| `"fraud"` \| `"other"` |

**Outputs:** Refund object with `refund_id` (`ref_...`), `status`, `amount`.

**Safety notes:**
- **Irreversible.** Confirm amount carefully before calling.
- For fraud reasons, also call `finpay_report_fraud` to notify FinPay's risk team.

---

### Tool: `finpay_create_payout`

**Description:** Disburses funds from the merchant's FinPay balance to a bank account or UPI VPA.

**Inputs:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `amount` | integer | ✅ | Paise. |
| `mode` | string | ✅ | `"imps"` \| `"neft"` \| `"upi"` \| `"rtgs"` |
| `purpose` | string | ✅ | `"vendor_payment"` \| `"salary"` \| `"refund"` \| `"cashback"` \| `"other"` |
| `recipient_name` | string | ✅ | Legal name of recipient. |
| `account_number` | string | ❌ | Required for IMPS/NEFT/RTGS. |
| `ifsc` | string | ❌ | Required for IMPS/NEFT/RTGS. 11-character IFSC code. |
| `vpa` | string | ❌ | Required for UPI. Format: `name@bankhandle`. |
| `reference` | string | ❌ | Merchant's internal reference for this payout. |

**Outputs:** Payout object with `payout_id` (`pyt_...`), `status`, `mode`.

**Safety notes:**
- **Financial action** — moves real money. Always confirm recipient details with user before calling.
- For amounts above ₹10,000, FinPay will perform penny-drop verification — payout may be delayed by 1–2 minutes.
- RTGS minimum: ₹2,00,000 (2000000 paise). Will error below this threshold.

---

### Tool: `finpay_list_payouts`

**Description:** Lists recent payouts with optional filters.

**Inputs:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `from_date` | string | ❌ | ISO 8601 date. Filter payouts created after this date. |
| `to_date` | string | ❌ | ISO 8601 date. |
| `status` | string | ❌ | Filter by status: `queued`, `processing`, `processed`, `failed`. |
| `limit` | integer | ❌ | Max results. Default 20. Max 100. |

**Safety notes:** Read-only. Safe to call freely.

---

### Tool: `finpay_report_fraud`

**Description:** Reports a suspected fraud event to FinPay's risk team. Required by RBI guidelines within 24 hours of detection.

**Inputs:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `order_id` | string | ✅ | The `ord_...` ID of the suspected fraudulent payment. |
| `fraud_type` | string | ✅ | `"card_not_present"` \| `"account_takeover"` \| `"friendly_fraud"` \| `"identity_theft"` \| `"other"` |
| `description` | string | ✅ | Free text description of the suspected fraud. Min 20 chars. |
| `reporter_reference` | string | ❌ | Merchant's internal case ID. |

**Safety notes:**
- Filing a fraud report initiates a FinPay investigation. Do not file frivolously.
- If `order_id` is already `refunded`, still file the report — required for RBI compliance.

---

## Agent Workflow Examples

### Pattern 1: Payment Collection Flow
```
1. finpay_create_payment_order(amount, receipt, customer_email)
   → redirect customer to payment_link
2. [Webhook: payment.captured arrives]
3. finpay_get_payment(order_id)
   → confirm status == "captured"
4. Fulfil order in merchant system
```

### Pattern 2: Vendor Payout After Sale
```
1. finpay_get_payment(order_id) → confirm "settled"
2. [Human confirms: "Pay vendor INR 1,200"]
3. finpay_create_payout(amount=120000, mode="upi", vpa="vendor@okaxis")
4. [Webhook: payout.processed arrives] → mark as disbursed
```

### Pattern 3: Fraud Refund Flow
```
1. Customer reports fraud
2. finpay_create_refund(order_id, amount=full, reason="fraud")
3. finpay_report_fraud(order_id, fraud_type="card_not_present", description="...")
4. Log case ID for compliance trail
```

---

## What Agents Must Not Do (MCP Safety Constraints)

These constraints must be enforced at the agent orchestration layer, not just documented:

- **No autonomous payouts above ₹10,000** without explicit human confirmation in the same conversational turn.
- **No bulk payout batches** without human review of the recipient list.
- **No fraud reports** without human confirmation — false reports have compliance consequences.
- **No looping on `finpay_get_payment`** — max 3 polling calls before pausing and notifying the user.
- **No calls to live API endpoints** from this reference implementation.

---

*MCP Spec Version: 1.0 | FinPay API Version: 2.1 | Last updated: June 2026*
