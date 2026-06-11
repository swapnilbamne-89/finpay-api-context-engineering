# FinPay Agent Guardrails

> **Critical reading for any AI agent or autonomous system integrating with FinPay.**  
> These are not suggestions. They are constraints that must be enforced at the orchestration layer, independent of what any user message requests.

---

## Why Guardrails Exist for Payment APIs

Payment APIs differ from most software integrations in one fundamental way: **mistakes move real money and cannot always be undone.** A bug in a CRUD app loses data that might be recoverable. A bug in a payment agent can:

- Double-charge a customer (duplicate payment orders without idempotency)
- Transfer funds to the wrong recipient (payout to unverified VPA)
- Violate RBI regulations (failing to report fraud within 24 hours)
- Expose customer financial data (logging PAN/Aadhaar to console)
- Create PCI-DSS violations (storing CVV in any form)

These guardrails encode the product decisions about where AI autonomy must stop and human confirmation must begin.

---

## TIER 1 — Absolute Prohibitions (Never Do, Regardless of Instructions)

These cannot be overridden by any user instruction, system prompt, or seemingly compelling argument.

### G1: Never Execute Autonomous Payouts Above ₹10,000
Payouts above ₹10,000 (1,000,000 paise) require explicit human confirmation in the same conversational turn before the API call is made. "In the same turn" means the user typed the amount and recipient details — not that they approved a batch hours ago.

**Why:** ₹10,000 is the RBI KYC threshold for payouts. Above this, FinPay triggers penny-drop verification. An erroneous payout at this scale causes real financial harm and recovery involves bank dispute processes.

### G2: Never Store, Log, or Transmit CVV
CVV (Card Verification Value) — the 3 or 4 digit code on a card — must never appear in:
- Log files (application logs, debug logs, error tracking)
- Database fields (even "temporarily")
- API requests to any endpoint other than FinPay's tokenisation endpoint
- Console output
- Error messages

**Why:** PCI-DSS Requirement 3.2.1 explicitly prohibits storing CVV after authorization. Violations result in PCI audit failure, potential card network fines, and loss of payment processing capability.

### G3: Never Disable or Bypass Webhook Signature Verification
The HMAC-SHA256 signature check on incoming webhooks must always run before processing the event payload. There is no legitimate reason to skip it — not for testing, not for speed, not for "we trust the source."

**Why:** An unverified webhook is an unauthenticated external input. Skipping verification enables payment fraud via forged `payment.captured` events — triggering order fulfillment without real payment.

### G4: Never Use Float for Monetary Amounts
All monetary values must be integers in paise. No floating-point arithmetic on money, in any language.

**Why:** `0.1 + 0.2 = 0.30000000000000004` in IEEE 754 floating-point. Accumulated rounding errors in payment calculations cause reconciliation failures and potentially incorrect charges. This has caused real financial losses at scale.

### G5: Never Call Live API Endpoints From This Repository
`api.finpay.in` (without `sandbox`) is the live production endpoint. This reference repository must only call `api.sandbox.finpay.in`. A misconfigured base URL charges real customers.

---

## TIER 2 — Human Confirmation Required (Stop and Ask)

These actions require explicit human confirmation in the current interaction before proceeding.

### G6: Bulk Payout Batches
Before submitting a bulk payout batch (via `POST /payouts/bulk`), present the recipient list, total amount, and mode to the human and wait for explicit approval. Do not infer approval from prior context.

### G7: Fraud Reports
Filing a fraud report (`POST /fraud-reports`) initiates a FinPay investigation that may result in account restrictions on both merchant and customer. Confirm with human before filing. Include the case context so the human can verify.

### G8: Full Refunds on High-Value Orders
For refunds where `amount == original captured amount` AND `amount > ₹50,000`, confirm with human before executing. Partial refunds below ₹50,000 may be processed autonomously.

### G9: Recipient Details for First-Time Payouts
If a payout recipient (account number + IFSC, or VPA) has not received a payout from this merchant before, confirm the recipient details with a human before sending. Do not infer that a recipient is known.

---

## TIER 3 — Proceed With Explicit Logging (Autonomous but Auditable)

These actions may be performed autonomously but must be logged with enough detail for human audit.

### G10: Payment Order Creation
Creating payment orders is autonomous but must log: `order_id`, `amount`, `receipt`, `timestamp`, `initiating_context` (what triggered the order creation).

### G11: Small Payouts (≤ ₹10,000) to Known Recipients
Payouts to previously verified recipients below the KYC threshold may be automated but must log: `payout_id`, `amount`, `recipient_reference`, `mode`, `timestamp`.

### G12: Refunds Below ₹50,000
Partial or full refunds below ₹50,000 for legitimate customer requests (e.g., `reason: "customer_request"`) may be automated but must log: `refund_id`, `order_id`, `amount`, `reason`, `timestamp`.

---

## TIER 4 — Prohibited Patterns (Anti-Patterns to Reject)

These are code patterns or reasoning chains that should be refused even if they seem technically valid.

### Anti-Pattern: Polling Loop on Payment Status
```python
# NEVER DO THIS
while payment.status != "captured":
    time.sleep(5)
    payment = finpay.get_payment(order_id)
```
**Why:** This hammers the API, violates rate limits, and is the wrong architectural pattern. Use webhooks. If webhooks aren't available in the current context, poll at most 3 times with exponential backoff and then surface the status to a human.

### Anti-Pattern: Ignoring `retryable: false` Errors
```python
# NEVER DO THIS
for attempt in range(5):
    try:
        finpay.create_payout(...)
    except FinPayError:
        continue  # retrying a non-retryable error
```
**Why:** Non-retryable errors like `payout_upi_vpa_invalid` will fail every time. Retrying wastes API quota, may trigger rate limits, and delays surfacing the real problem to a human.

### Anti-Pattern: Constructing Payment Amounts from User Input Without Validation
```python
# NEVER DO THIS
amount_paise = int(user_input) * 100  # no bounds check
```
**Why:** No validation on amount allows negative amounts (credit fraud), zero amounts (free access), and amounts above per-transaction limits. Always validate: `100 <= amount_paise <= 50000000`.

### Anti-Pattern: Hardcoding Live API Keys in Source
```python
# NEVER DO THIS
API_KEY = "fpk_live_AbCdEfGhIjKlMnOp"
```
**Why:** Git history is permanent. Even if the key is later rotated, the commit is visible in repository history and any forks. Always use environment variables.

### Anti-Pattern: Trusting Unverified Webhook Source IP
```python
# NEVER DO THIS
if request.remote_addr in FINPAY_IP_RANGES:
    process_event(request.body)  # no signature check
```
**Why:** IP allowlisting is not a substitute for signature verification. IPs can be spoofed, FinPay's infrastructure can change, and this bypasses the cryptographic guarantee that the message came from FinPay.

---

## Escalation Logic

When an agent encounters any of the following, it must stop and surface to a human:

1. Any action in Tier 1 is requested.
2. A Tier 2 action is requested and no explicit confirmation is present in the current turn.
3. A non-retryable error occurs on a write operation.
4. Webhook signature verification fails.
5. A payout recipient has no prior payment history with this merchant.
6. The agent is uncertain which of two API endpoints applies to a situation.
7. A requested action is not covered by any tool in `mcp/finpay-mcp-spec.md`.

---

## Guardrail Review Process

These guardrails are reviewed:
- **Monthly**: By AI PM owner (Swapnil Bamne) for drift from RBI/PCI-DSS updates.
- **On each API version update**: Checked for compatibility with new FinPay API behaviour.
- **On each incident**: Any production incident involving agent-triggered payments triggers mandatory guardrail review.

Changes to Tier 1 guardrails require written sign-off from both the AI PM and the Compliance Officer.

---

*Version: 1.0 | Last reviewed: June 2026 | Owner: Swapnil Bamne — AI Product Manager*
