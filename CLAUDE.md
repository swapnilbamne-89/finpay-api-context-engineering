# CLAUDE.md — FinPay API Integration Context

> This file is read automatically by Claude Code at the start of every session in this repository. It defines the project context, coding conventions, agent permissions, and guardrails for AI-assisted development on the FinPay API integration layer.

---

## Project Identity

**Repository:** `finpay-api` — Context engineering reference implementation for the FinPay payments API  
**Purpose:** Demonstrate LLM-native API documentation and agent integration patterns for fintech  
**Stack:** OpenAPI 3.1 · Python SDK (reference) · Node.js examples · Webhook handlers  
**Environment:** This repo contains ONLY test credentials. Live credentials are never committed.

---

## Quick Reference — Most Important Facts

1. **Base URL (test):** `https://api.sandbox.finpay.in/v2`
2. **All amounts in paise** (integer). ₹100 = `10000`. Never use floats for money.
3. **Always use Idempotency-Key** (UUIDv4) on POST requests for payments and payouts.
4. **Verify webhook signatures** before processing any webhook event. See `docs/webhooks.md`.
5. **Never log or commit API keys, secrets, or customer PAN/Aadhaar.**
6. **Test card:** `4111 1111 1111 1111` · Expiry: any future date · CVV: `123`
7. **Test UPI VPA:** `success@finpaytest` (always succeeds) · `fail@finpaytest` (always fails)

---

## Architecture Decisions (Do Not Override Without Discussion)

### Why Paise, Not Rupees
Indian payment networks (UPI, RBI clearing) operate in paise at the protocol level. Using integers eliminates floating-point rounding errors that can cause reconciliation failures and RBI compliance issues. This is not a style choice — it is a correctness requirement.

### Why Idempotency Keys Are Mandatory
Network failures between client and FinPay server can cause duplicate payment creation if retried without idempotency keys. In production, this would charge customers twice. All payment and payout creation calls must use `Idempotency-Key`.

### Why Webhooks Over Polling
The FinPay API is designed webhook-first. The payment state machine is asynchronous — a customer's UPI payment can take 30–120 seconds to confirm. Polling `/payments/{id}` under load violates rate limits and introduces unnecessary latency. Always use webhooks for state transitions.

---

## Coding Conventions

### Python
- Use `httpx` (async) or `requests` (sync) for HTTP calls. Not `urllib`.
- Money calculations: use `decimal.Decimal` or integer paise. Never `float`.
- Environment variables for credentials: `FINPAY_API_KEY`, `FINPAY_WEBHOOK_SECRET`. Load via `python-dotenv`.
- Type hints required on all function signatures.
- Error handling: catch `FinPayError` base class; check `error.retryable` before retrying.

```python
# CORRECT
amount_paise = int(Decimal("1500.50") * 100)  # = 150050

# WRONG — do not do this
amount_paise = 1500.50 * 100  # float precision error risk
```

### JavaScript / TypeScript
- Use `fetch` or `axios`. Prefer `axios` for interceptor-based retry logic.
- TypeScript strict mode (`strict: true` in tsconfig). No `any` types on FinPay response objects.
- Use the FinPay TypeScript SDK types from `@finpay/sdk` for all API objects.
- Money: use `bigint` or keep as integer paise. Never use `number` for amounts > 2^53.

### Webhook Handlers
```python
# ALWAYS verify before processing — no exceptions
def handle_webhook(request):
    signature = request.headers.get("X-FinPay-Signature")
    if not finpay.webhooks.verify(signature, request.body, WEBHOOK_SECRET):
        return Response(status=400)  # reject silently — do not leak reason
    event = finpay.webhooks.parse(request.body)
    # process event...
    return Response(status=200)  # return 200 IMMEDIATELY, process async
```

### Environment Configuration
```
# .env (never commit this file)
FINPAY_API_KEY=fpk_test_...
FINPAY_WEBHOOK_SECRET=fps_test_...
FINPAY_ENV=test

# .env.example (commit this)
FINPAY_API_KEY=fpk_test_YOUR_KEY_HERE
FINPAY_WEBHOOK_SECRET=fps_test_YOUR_SECRET_HERE
FINPAY_ENV=test
```

---

## What Claude Is Allowed To Do

- Write and modify code in `examples/`, `tests/`, `scripts/`, and `docs/`.
- Generate new OpenAPI path definitions following the existing schema patterns.
- Write unit tests for payment state machine logic, webhook verification, and error handling.
- Suggest refactors to improve idempotency, error handling, or retry logic.
- Generate client SDK stubs from the OpenAPI spec.
- Update `CHANGELOG.md` and version numbers following SemVer.
- Answer questions about FinPay API behaviour using `llms-full.txt` as the authoritative source.

---

## What Claude Must NOT Do

### Security-Critical Prohibitions
- **NEVER generate, suggest, or output real API keys, secrets, or tokens** — even as examples. Always use placeholder format: `fpk_test_YOUR_KEY_HERE`.
- **NEVER write code that logs payment amounts, customer PAN, Aadhaar, card numbers, or CVV** to console, files, or monitoring systems.
- **NEVER disable webhook signature verification**, even in test environments or "quick demo" code.
- **NEVER use `float` or `double` for monetary amounts** in any language.
- **NEVER store CVV** — in memory, database, logs, or anywhere. This is a PCI-DSS violation.

### Architecture Prohibitions
- **NEVER call live API endpoints** (`api.finpay.in`, not `api.sandbox.finpay.in`) from this repository. This is a reference/demo repo.
- **NEVER implement polling loops** on `/payments/{id}` as a substitute for webhooks.
- **NEVER commit `.env` files** or any file containing real credentials.
- **NEVER modify `llms.txt` or `llms-full.txt`** without updating the version timestamp and changelog entry.

### Compliance Prohibitions
- **NEVER write code that transmits customer PAN or Aadhaar** to non-FinPay endpoints.
- **NEVER suggest storing raw card numbers** in any database, cache, or log.
- **NEVER implement cross-border transactions** — FinPay is India-only (INR only).

---

## Ambiguity Resolution Protocol

When you encounter an ambiguous instruction, use this decision tree before acting:

```
1. Is this a security or compliance concern?
   → YES: Stop. Ask for explicit confirmation before proceeding.
   
2. Would this modify llms.txt, llms-full.txt, or CLAUDE.md?
   → YES: Stop. These are governed documents. Ask before modifying.

3. Is the instruction about live vs. test environment?
   → Assume TEST unless the user explicitly confirms LIVE intent in the same message.

4. Does the request involve a payment amount in rupees (not paise)?
   → Convert to paise. Note the conversion in a code comment.

5. Is the user asking about a FinPay feature that isn't in llms-full.txt?
   → Say so explicitly. Do not invent API behaviour. Suggest checking official docs.
```

---

## Key File Locations

| File | Purpose |
|------|---------|
| `llms.txt` | LLM index — start here for API overview |
| `llms-full.txt` | Full authoritative context — use for detailed behaviour |
| `openapi/finpay-openapi.yaml` | Machine-readable API spec |
| `docs/errors.md` | All error codes and retry guidance |
| `docs/webhooks.md` | Webhook signature verification and event catalogue |
| `context/disambiguation.md` | Concept disambiguation — check here before making assumptions |
| `context/guardrails.md` | Agent safety rules — read before any automated action |
| `mcp/finpay-mcp-spec.md` | MCP tool definitions for agent integration |

---

## Testing

```bash
# Run all tests
pytest tests/ -v

# Run webhook verification tests specifically
pytest tests/test_webhooks.py -v

# Run with test credentials loaded
source .env && pytest tests/ -v

# Lint
ruff check .

# Type check
mypy src/
```

Test accounts:
- Successful card payment: card `4111 1111 1111 1111`
- Failed payment (insufficient funds): card `4000 0000 0000 9995`
- 3DS required: card `4000 0027 6000 3184`
- Successful UPI: VPA `success@finpaytest`
- Failed UPI (bank error): VPA `fail@finpaytest`
- Successful payout: IFSC `FINP0000001`, Account `1234567890`

---

## Versioning & Changelog

This repo follows SemVer. Context files (`llms.txt`, `llms-full.txt`) are versioned separately as `context-vX.Y`.

When updating context files:
1. Increment `context-vX.Y` in the file header.
2. Add entry to `CHANGELOG.md` under `## Context Layer`.
3. Run `scripts/validate-context.py` to check for broken internal links.
4. Update `README.md` if the structure changes.

---

*This file is governed. Changes require PR review from the AI PM owner (Swapnil Bamne) and a changelog entry.*  
*Last updated: June 2026 · Context version: v1.0*
