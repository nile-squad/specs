# Operations

Part of the [Nylon Pay SDK Spec](./spec.md).

The SDK exposes eight operations and one utility:

| Operation | Pattern | Description |
|-----------|---------|-------------|
| `collectPayment` | Async (PaymentInstance) | Initiate a collection from a customer's phone or bank account |
| `collectPaymentAndResolve` | Sync (blocking) | Initiate a collection and block until terminal state |
| `makePayout` | Async (PaymentInstance) | Initiate a disbursement to a destination account |
| `makePayoutAndResolve` | Sync (blocking) | Initiate a disbursement and block until terminal state |
| `getStatus` | Sync | One-shot status check for a transaction by reference |
| `getTransaction` | Sync | Look up a full transaction record by id or reference |
| `verifyPhone` | Sync | Pre-validate a phone number |
| `createInvoice` | Sync | Generate a hosted payment link with optional line items |
| `verifyWebhookSignature` | Utility | Verify HMAC signature of an incoming webhook payload |

### collectPayment

Initiates a payment collection. Returns a PaymentInstance that polls until the transaction reaches a terminal state.

Input shape:
- `amount` — positive integer in smallest currency unit (e.g., cents, shillings)
- `currency` — ISO 4217 currency code
- `customer` — `{ name, phoneNumber, email? }`
- `description` — human-readable narration
- `reference?` — merchant-supplied idempotency key (auto-generated if omitted). When supplied, it MUST be 13–15 characters (see [Reference constraints](#reference-constraints)).
- `method?` — payment method: `"mobileMoney"` or `"bank"` (defaults to `"mobileMoney"`)
- `bank?` — required when `method` is `"bank"`: `{ accountNumber, bankName }`
- `metadata?` — arbitrary key-value pairs attached to the transaction

Returns: `PaymentInstance`

#### Reference constraints

A supplied `reference` MUST be **13–15 characters** on every create operation
(`collectPayment`, `collectPaymentAndResolve`, `makePayout`, `makePayoutAndResolve`,
`createInvoice`). The backend echoes the reference verbatim as the provider's
`merchantTransactionId`, which is bounded to 13–15 characters; an out-of-range
reference is rejected. Implementations MUST validate this **synchronously** at the
call site (same as `amount`) and raise a `validation` error — they MUST NOT defer it
to a network round-trip. An omitted `reference` is auto-generated as a 15-character
value and always satisfies the constraint. Common pitfall: passing a 36-character
UUID order id — hash or truncate it to ≤15 characters first.

#### Reference uniqueness and replay

The reference is the transaction identity and the only idempotency mechanism —
there is no separate idempotency key.

- **One reference, one transaction.** Calling a create operation again with the
  same reference does NOT charge again: the server replays the existing
  transaction's current state, and the response carries `duplicate: true`.
- **A new transaction needs a new reference.** Same customer, same amount, same
  timing — none of it matters; a fresh reference always starts a fresh payment.
- A reference that is taken and cannot be replayed (it belongs to another
  account) fails with the `duplicate` error category — see
  [Error Categories](./errors.md). Retry with a new reference.
- Retrying a network failure (5xx/timeout) MUST reuse the same reference so the
  retry replays instead of double-charging.

### collectPaymentAndResolve

Initiates a payment collection and blocks until the transaction reaches a terminal state. Equivalent to calling `collectPayment` then `wait()`, but provided as a single operation for convenience in synchronous contexts.

Input shape: same as `collectPayment`.

Returns: `Transaction` on success, error result on failure/cancellation/timeout.

### makePayout

Initiates a disbursement. Returns a PaymentInstance that polls until the payout reaches a terminal state.

Input shape:
- `amount` — positive integer
- `currency` — ISO 4217
- `customer` — `{ name, phoneNumber, email? }`
- `destination` — `{ accountHolderName, accountNumber, bankName?, phone? }`
- `description` — narration
- `reference?` — idempotency key; when supplied, MUST be 13–15 characters (see [Reference constraints](#reference-constraints))
- `metadata?` — arbitrary key-value pairs

Returns: `PaymentInstance`

### makePayoutAndResolve

Initiates a disbursement and blocks until the payout reaches a terminal state. Equivalent to calling `makePayout` then `wait()`, but provided as a single operation for convenience in synchronous contexts.

Input shape: same as `makePayout`.

Returns: `Transaction` on success, error result on failure/cancellation/timeout.

### getStatus

One-shot status check. Does not poll. Returns the current transaction state.

Input shape:
- `reference` — transaction reference

Returns: `{ reference, status, amount, currency, updatedAt }`

### getTransaction

Full transaction lookup.

Input shape:
- `id?` — transaction UUID
- `reference?` — merchant reference

At least one of `id` or `reference` is required.

Returns: full transaction record (see [Transaction Shape](./types.md#transaction-shape))

### verifyPhone

Pre-validates a phone number with the payment provider. Returns the registered name on the account.

Input shape:
- `phoneNumber` — E.164 or local format
- `purpose?` — `"collection"` or `"payout"` (provider may route differently)

Returns: `{ phoneNumber, customerName, verified }`

### createInvoice

Generates a hosted payment link. The link renders a payment page where the customer completes the transaction.

Input shape:
- `amount` — positive integer
- `currency` — ISO 4217
- `description` — invoice narration
- `items?` — array of `{ name, quantity, unitPrice }` (max 50 items)
- `redirectUrl?` — URL to redirect customer after payment
- `reference?` — idempotency key; when supplied, MUST be 13–15 characters (see [Reference constraints](#reference-constraints))
- `metadata?` — arbitrary key-value pairs

Returns: `{ id, url, token, expiresAt, status }`

**Hosted Payment Page:**

The generated `url` points to a hosted payment page where the customer completes the transaction. The page supports:

- **Mobile Money** — customer enters phone number, receives payment prompt on phone
- **Card** — customer enters card details, may be redirected to card issuer for 3D Secure
- **Bank Transfer** — customer selects bank and enters account number

**Line Items:**

When `items` are provided, the payment page displays an itemized breakdown (name, quantity, unit price) alongside the total amount. This gives customers visibility into what they're paying for.

**Redirect Behavior:**

When `redirectUrl` is provided, the customer is redirected to that URL after successful payment. For card payments, the customer may first be redirected to the card issuer's 3D Secure page, then to your `redirectUrl` after verification.

**Payment URL Format:**

The `url` field follows the pattern: `{frontendUrl}/pay?token={token}`

The token is a unique identifier for the payment link. Tokens expire after 24 hours by default.

### verifyWebhookSignature

Standalone utility. Verifies that a webhook was genuinely sent by Nylon Pay and
is not a replay.

Input shape:
- `payload` — raw request body (string or bytes, depending on language)
- `signature` — signature from the webhook header
- `secret` — merchant's webhook secret
- `toleranceSeconds` — optional replay-protection window (default `300`). Set to
  `0` to disable the freshness check.

Returns: boolean — `true` only when **both** hold:
1. **Authenticity** — HMAC-SHA256 over the raw payload bytes equals `signature`.
2. **Freshness** — the `timestamp` field carried *inside the signed body* is
   within `toleranceSeconds` of now. Every Nylon Pay delivery (including retries)
   stamps and signs a current timestamp, so a captured `(body, signature)` pair
   goes stale and a replay is rejected, while legitimate delayed retries — each
   freshly stamped — still pass. The timestamp is read from the signed body, not
   from a header, so it cannot be refreshed without the secret. When
   `toleranceSeconds > 0` and the signed body carries no parseable timestamp,
   verification fails closed.

See [D16](./decision-records.md#d16-webhook-verification-is-replay-protected).

