# FinPay Agent Workflow Patterns

> Reusable patterns for common agentic workflows involving the FinPay API. Each pattern documents the happy path, failure modes, and guardrail checkpoints. These are reference implementations — adapt to your specific orchestration framework.

---

## Pattern 1: Checkout Payment Collection

**Scenario:** A customer checks out from a merchant's platform. The agent creates a payment order, redirects the customer, waits for confirmation, and triggers fulfillment.

```
TRIGGER: Customer clicks "Pay Now"
ACTOR: Backend agent / webhook handler

Step 1: Validate order
  - Verify cart total in paise (integer)
  - Validate: 100 ≤ amount ≤ 50,000,000
  - Generate idempotency key (UUIDv4)

Step 2: Create payment order
  → finpay_create_payment_order(amount, receipt, customer_email)
  ← Returns: order_id, payment_link

Step 3: Store order mapping
  - Save {merchant_order_id → finpay_order_id} in your database
  - Set status: PENDING_PAYMENT

Step 4: Redirect customer
  - Redirect to payment_link
  - Link expires in 15 minutes — do not store link, store order_id

Step 5: Receive webhook
  → Event: payment.captured
  - Verify signature (MANDATORY — see guardrail G3)
  - Match finpay_order_id to merchant_order_id
  - Confirm status == "captured" AND amount matches

Step 6: Fulfil order
  - Update order status: PAID
  - Trigger fulfillment (ship, provision, grant access)
  - Send confirmation to customer

FAILURE MODES:
  - payment.failed webhook: Update status to PAYMENT_FAILED. Allow retry.
  - No webhook after 15 min: Query finpay_get_payment(order_id) — max 3 times.
    If still pending, alert human.
  - Duplicate webhook: Idempotency check on event_id — safe to ignore duplicates.
```

**Guardrail checkpoints:**
- Step 1: Amount validation (anti-pattern: no bounds check)
- Step 5: Signature verification (guardrail G3)
- Step 5: Never fulfil on payment.authorized alone — wait for payment.captured

---

## Pattern 2: Vendor Payout After Settlement

**Scenario:** After collecting payment from a customer, the merchant pays a marketplace vendor their share.

```
TRIGGER: payment.settled webhook received
ACTOR: Marketplace settlement agent

Step 1: Calculate vendor share
  - Retrieve platform fee percentage from config
  - vendor_amount_paise = captured_amount - platform_fee
  - Validate: vendor_amount_paise > 0

Step 2: Check merchant balance
  → GET /accounts/balance
  - Confirm available_balance >= vendor_amount_paise
  - If insufficient: queue payout for next settlement cycle

Step 3: GUARDRAIL CHECK
  - If vendor_amount_paise > 1,000,000 (₹10,000): STOP → human confirmation required
  - If recipient not in verified vendor list: STOP → human confirmation required
  - If recipient verified AND amount ≤ ₹10,000: proceed autonomously

Step 4: Select payout mode
  - amount ≤ 100,000 paise (₹1,000): UPI (fastest, no minimum)
  - 100,001–10,000,000 paise: IMPS (immediate, higher limit)
  - > 10,000,000 paise: NEFT or RTGS

Step 5: Create payout
  → finpay_create_payout(amount, mode, purpose="vendor_payment", vpa or account)
  ← Returns: payout_id

Step 6: Track payout
  - Store {vendor_id → payout_id} in payout ledger
  - Set status: PAYOUT_PROCESSING

Step 7: Receive webhook
  → Event: payout.processed
  - Update vendor ledger: PAYOUT_COMPLETE
  - Notify vendor of payment

FAILURE MODES:
  - payout.failed: Check error code
    - payout_upi_vpa_invalid: Alert human — do not retry automatically
    - payout_recipient_bank_error: Retry with IMPS instead of UPI
    - server_error: Retry with exponential backoff (max 3 attempts)
  - payout.reversed: Funds returned — log, alert human, reinitiate after verification
```

---

## Pattern 3: Automated Refund Handling

**Scenario:** Customer contacts support requesting a refund. Agent validates eligibility and processes refund.

```
TRIGGER: Customer support ticket / refund request
ACTOR: Support automation agent

Step 1: Retrieve payment details
  → finpay_get_payment(order_id)
  - Verify status is "captured" or "settled" (not "failed" or "refunded")
  - Calculate max refundable: amount_captured - amount_refunded

Step 2: Validate refund eligibility
  - Refund window: within 180 days of capture (FinPay hard limit)
  - Requested amount ≤ max_refundable
  - Refund reason is valid

Step 3: GUARDRAIL CHECK
  - If requested_amount == amount_captured AND amount_captured > 5,000,000 (₹50,000):
    → STOP: Require human confirmation
  - Otherwise: proceed

Step 4: Process refund
  → finpay_create_refund(order_id, amount, reason)
  ← Returns: refund_id

Step 5: Update records
  - Log: {ticket_id, order_id, refund_id, amount, reason, timestamp}
  - Update customer record: REFUND_INITIATED

Step 6: Notify customer
  - "Your refund of ₹X has been initiated. 
     UPI: 1 business day. Card: 5–7 business days."

Step 7: Receive confirmation (optional)
  → Event: payment.refunded (webhook)
  - Update status: REFUND_CONFIRMED

FAILURE MODES:
  - payment_invalid_state: Payment not in refundable state — check status, explain to customer
  - validation_amount_exceeds_captured: Requested more than captured — recalculate and retry
  - server_error: Retry once; if fails again, escalate to human agent
```

---

## Pattern 4: Fraud Detection and Response

**Scenario:** Fraud detection system flags a payment. Agent executes the fraud response workflow.

```
TRIGGER: Internal fraud score exceeds threshold
ACTOR: Fraud response agent

Step 1: Retrieve payment
  → finpay_get_payment(order_id)
  - Record: status, amount, payment_method, customer details

Step 2: Assess refund requirement
  - If status == "captured" or "settled": refund required
  - If status == "authorized": void required
  - If status == "processing": monitor for final state

Step 3: ALWAYS STOP HERE — human confirmation required for fraud response
  - Present to human:
    - order_id, amount, payment_method, fraud signal details
    - Proposed action: [refund/void/monitor]
  - Wait for explicit confirmation

Step 4 (if confirmed): Execute refund
  → finpay_create_refund(order_id, amount=full, reason="fraud")

Step 5 (if confirmed): File fraud report
  → finpay_report_fraud(order_id, fraud_type, description)
  ← Returns: report_id

Step 6: Log for compliance
  - Record: {order_id, refund_id, report_id, fraud_type, timestamp, confirming_human}
  - RBI requires this trail for FMR reporting

Step 7: Block customer (if policy requires)
  - This is a merchant-side action — not a FinPay API call
  - Flag customer_id in merchant's risk system

NOTE: Never automate Step 3. Fraud response has compliance and customer relationship 
implications that require human judgment. An automated false-positive response 
(refunding a legitimate payment) creates a chargeback risk.
```

---

## Pattern 5: Retry Logic Template

Reusable retry wrapper for any FinPay API call.

```python
import time
import uuid

def finpay_call_with_retry(api_func, *args, max_attempts=5, **kwargs):
    """
    Retry a FinPay API call with exponential backoff.
    Only retries on retryable errors.
    Always uses idempotency key.
    """
    idempotency_key = str(uuid.uuid4())
    
    for attempt in range(max_attempts):
        try:
            result = api_func(*args, idempotency_key=idempotency_key, **kwargs)
            return result
            
        except FinPayError as e:
            # Check retryability FIRST — do not retry non-retryable errors
            if not e.retryable:
                raise  # Surface to caller immediately
            
            if attempt == max_attempts - 1:
                # Exhausted retries — surface to human
                raise FinPayMaxRetriesError(
                    f"Failed after {max_attempts} attempts: {e.code}"
                ) from e
            
            # Exponential backoff: 1s, 4s, 16s, 64s
            wait = 4 ** attempt
            time.sleep(wait)
    
    # Should not reach here
    raise RuntimeError("Unexpected retry loop exit")
```

**Key properties of this pattern:**
- Single idempotency key across all retries — ensures FinPay treats all retries as the same request
- Checks `error.retryable` before retrying — respects `payment_failed_card_declined` as non-retryable
- Exponential backoff — avoids hammering the API during incidents
- Raises distinct error after max attempts — enables human escalation

---

*Pattern Version: 1.0 | Last updated: June 2026 | Maintained by: Swapnil Bamne — AI Product Manager*
