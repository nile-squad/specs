# Operations

Part of the [Nylon Pay SDK Spec](./spec.md).

The SDK exposes ten operations and one utility:

| Operation | Pattern | Description |
|-----------|---------|-------------|
| `collectPayment` | Async (PaymentInstance) | Initiate a collection from a customer's phone or bank account |
| `collectPaymentAndResolve` | Sync (blocking) | Initiate a collection and block until terminal state |
| `makePayout` | Async (PaymentInstance) | Initiate a disbursement to a destination account |
| `makePayoutAndResolve` | Sync (blocking) | Initiate a disbursement and block until terminal state |
| `getStatus` | Sync | One-shot status check for a transaction by reference |
| `getTransaction` | Sync | Look up a full transaction record by id or reference |
| `listTransactions` | Sync | List transactions with optional filters including tags |
| `getTransactionsByTag` | Sync | Shorthand for filtering transactions by a single tag |
| `verifyPhone` | Sync | Pre-validate a phone number |
| `createInvoice` | Sync | Generate a hosted payment link with optional line items |
| `verifyWebhookSignature` | Utility | Verify HMAC signature of an incoming webhook payload |

### collectPayment

Initiates a payment collection. Returns a PaymentInstance that polls until the transaction reaches a terminal state.

Input shape:
- `amount`: positive integer in smallest currency unit (e.g., cents, shillings)
- `currency`: ISO 4217 currency code
- `customer`: `{ name, phoneNumber, email? }`. The `phoneNumber` is normalized
  automatically to international format (`256XXXXXXXXX`) by the SDK before the
  request leaves. Accepted formats: local (`0768499027`), international with `+`
  (`+256768499027`), international without `+` (`256768499027`), with or without
  spaces. See [Phone Number Normalization](./types.md#phone-number-normalization).
- `description`: human-readable narration
- `reference?`: merchant-supplied idempotency key (auto-generated if omitted). When supplied, it MUST be a valid UUID (see [Reference constraints](#reference-constraints)).
- `method?`: payment method: `"mobileMoney"` or `"bank"` (defaults to `"mobileMoney"`)
- `bank?`: required when `method` is `"bank"`: `{ accountNumber, bankName }`
- `tags?`: up to 10 labels attached to the transaction for filtering and reporting. See [Smart Tags](#smart-tags).
- `metadata?`: arbitrary key-value pairs attached to the transaction
- `testOutcome?`: sandbox-only forced outcome: `"success"` always succeeds, `"fail"` always fails. Omitted, the sandbox picks a result at random. The SDK MUST validate the value synchronously and raise a `validation` error for anything else; the server rejects `testOutcome` on live keys with a `validation` error.

Returns: `PaymentInstance`

#### Reference constraints

A supplied `reference` MUST be a **valid UUID** on every create operation
(`collectPayment`, `collectPaymentAndResolve`, `makePayout`, `makePayoutAndResolve`,
`createInvoice`). Any UUID version is accepted, so a v1, v4 or v5 value all pass.
Implementations MUST validate this **synchronously** at the call site (same as
`amount`) and raise a `validation` error. They MUST NOT defer it to a network
round-trip. An omitted `reference` is auto-generated, and implementations SHOULD
generate a UUID v4.

Provider identifier limits do not constrain what a merchant may pass here. Where a
payment provider bounds the length of its own transaction identifier, the backend
generates a separate internal value for that call and keeps the merchant reference
intact.

#### Reference uniqueness and replay

The reference is the transaction identity and the only idempotency mechanism.
There is no separate idempotency key.

- **One reference, one transaction.** Calling a create operation again with the
  same reference does NOT charge again: the server replays the existing
  transaction's current state, and the response carries `duplicate: true`.
- **A new transaction needs a new reference.** Same customer, same amount, same
  timing, none of it matters; a fresh reference always starts a fresh payment.
- A reference that is taken and cannot be replayed (it belongs to another
  account) fails with the `duplicate` error category. See
  [Error Categories](./errors.md). Retry with a new reference.
- Retrying a network failure (5xx/timeout) MUST reuse the same reference so the
  retry replays instead of double-charging.

### collectPaymentAndResolve

Initiates a payment collection and blocks until the transaction reaches a terminal state. Equivalent to calling `collectPayment` then `wait()`, but provided as a single operation for convenience in synchronous contexts. When the server's inline poll budget (~60s) ends still pending, the SDK continues client-side status polling until terminal (or merchant caps / `onDelayed: "return"`).

Input shape: same as `collectPayment`.

Returns: `Transaction` on success, error result on failure/cancellation/timeout.

### makePayout

Initiates a disbursement. Returns a PaymentInstance that polls until the payout reaches a terminal state.

Input shape:
- `amount`: positive integer
- `currency`: ISO 4217
- `customer`: `{ name, phoneNumber, email? }`. The `phoneNumber` is normalized
  automatically to international format (`256XXXXXXXXX`). Same accepted formats as
  `collectPayment`. See [Phone Number Normalization](./types.md#phone-number-normalization).
- `destination`: `{ accountHolderName, accountNumber, bankName?, phone? }`
- `description`: narration
- `reference?`: idempotency key; when supplied, MUST be a valid UUID (see [Reference constraints](#reference-constraints))
- `tags?`: up to 10 labels. See [Smart Tags](#smart-tags).
- `metadata?`: arbitrary key-value pairs
- `testOutcome?`: sandbox-only forced outcome, same semantics as `collectPayment`.

Returns: `PaymentInstance`

### makePayoutAndResolve

Initiates a disbursement and blocks until the payout reaches a terminal state. Equivalent to calling `makePayout` then `wait()`, but provided as a single operation for convenience in synchronous contexts. When the server's inline poll budget ends still pending, the SDK continues client-side status polling until terminal (or merchant caps / `onDelayed: "return"`).

Input shape: same as `makePayout`.

Returns: `Transaction` on success, error result on failure/cancellation/timeout.

### getStatus

One-shot status check. Does not poll. Returns the current transaction state.

Input shape:
- `reference`: transaction reference

Returns: `{ reference, status, amount, currency, updatedAt, delayed? }`

### getTransaction

Full transaction lookup.

Input shape:
- `id?`: transaction UUID
- `reference?`: merchant reference

At least one of `id` or `reference` is required.

Returns: full transaction record (see [Transaction Shape](./types.md#transaction-shape))

### listTransactions

Returns a paginated list of transactions for the authenticated account, with optional filters.

Input shape (`ListTransactionsInput`, all fields optional):
- `tags?`: array of tag strings. Uses **AND semantics**: only transactions carrying **all** listed tags are returned.
- `status?`: filter by status: `"pending"`, `"processing"`, `"on_hold"`, `"successful"`, `"failed"`, `"cancelled"`
- `type?`: filter by type: `"collection"`, `"payout"`, `"invoice"`
- `limit?`: results per page, 1–100 (default `20`)
- `offset?`: zero-based pagination offset (default `0`)
- `createdAfter?`: ISO 8601 datetime, earliest creation time (inclusive)
- `createdBefore?`: ISO 8601 datetime, latest creation time (inclusive)

Returns: `ListTransactionsResponse`, `{ transactions: TransactionSummary[], count, limit, offset, tags }`

`count` is the total number of matching transactions (useful for pagination). `tags` echoes the filter tags applied.

### getTransactionsByTag

Shorthand for filtering by a single tag. Accepts one required `tag` argument plus any `ListTransactionsInput` options except `tags`.

Equivalent to `listTransactions({ tags: [tag], ...options })`.

Returns: `ListTransactionsResponse`

### Smart Tags

Tags are short labels attached to a transaction at creation time. They persist on the transaction record and can be used to filter or group transactions by campaign, product, team, channel, or any merchant-defined dimension.

**Normalization:** applied by the backend at write time:
- Lowercased and whitespace-trimmed
- Characters outside `[a-z0-9\-_:.]` are rejected (tag is dropped)
- Max 50 characters per tag; longer tags are dropped
- Max 10 tags per transaction; extras beyond the first 10 are dropped
- Duplicates removed after normalization

**Reserved tags:** `"live"` and `"test"` are set automatically to mark the transaction mode. Passing either in `tags` has no effect.

**Filter semantics:** `listTransactions({ tags: ["a", "b"] })` returns only transactions that carry **both** `"a"` and `"b"`, not either.

### verifyPhone

Pre-validates a phone number with the payment provider. Returns the registered name on the account.

Input shape:
- `phoneNumber`: any accepted format (local `0XXXXXXXXX`, international `+256XXXXXXXXX`
  or `256XXXXXXXXX`, with or without spaces). The backend normalizes it to international
  format. See [Phone Number Normalization](./types.md#phone-number-normalization).
- `purpose?`: `"collection"` or `"payout"` (provider may route differently)

Returns: `{ phoneNumber, customerName, verified }`

### createInvoice

Generates a hosted invoice and emails it to the customer. The returned payment link directs the customer to a mobile-money checkout page.

Input shape:
- `amount`: positive integer in smallest currency unit
- `currency`: ISO 4217
- `customerEmail`: required; invoice is sent to this address
- `customerName?`: display name shown on the invoice
- `customerPhone?`: pre-fills the phone field on the payment page
- `description?`: invoice narration
- `dueDate?`: ISO 8601 date string (e.g. `"2025-12-31"`)
- `items?`: array of `{ name, quantity, unitPrice }` (max 50 items)
- `merchantReference?`: stored on the transaction for reconciliation
- `tags?`: up to 10 labels. See [Smart Tags](#smart-tags).
- `metadata?`: arbitrary key-value pairs

Returns: `{ id, invoiceNumber, paymentLink, amount, currency, status }`

The customer receives an email containing the `paymentLink`. Opening it shows a mobile-money payment page where the customer enters their phone number to complete the payment. A receipt email is sent automatically after a successful payment.

### verifyWebhookSignature

Standalone utility. Verifies that a webhook was genuinely sent by Nylon Pay and
is not a replay.

Input shape:
- `payload`: raw request body (string or bytes, depending on language)
- `signature`: value of the `x-nylon-signature` HTTP request header
- `secret`: the merchant's **webhook secret**, which is NOT the `apiSecret` used for
  request and response signing. They are separate credentials; using `apiSecret` here
  makes every webhook fail verification.
- `toleranceSeconds`: optional replay-protection window in seconds (default `300`).
  `0` means a tolerance of **zero seconds, maximum strictness**, and does NOT disable
  the check. Opting out requires the explicit `DISABLE_FRESHNESS_CHECK` sentinel
  (value `-1`); any other negative value is rejected (verification returns `false`).
  See [D20](./decision-records.md#d20-0-tolerance-means-strict-not-disabled) and
  invariant 23.

`verifyWebhookSignature` returns `false` for any verification failure, it NEVER
raises or throws, regardless of input (malformed signature, non-hex signature,
invalid UTF-8 in the payload, empty payload, unparseable JSON body, missing
timestamp). The HMAC is computed over the raw payload bytes before any JSON
parsing; if the HMAC verifies but the body cannot be parsed for the timestamp,
verification fails closed (returns `false`).

Returns: boolean, `true` only when **both** hold:
1. **Authenticity:** HMAC-SHA256 over the raw payload bytes equals `signature`.
2. **Freshness:** the `timestamp` field carried *inside the signed body* is
   within `toleranceSeconds` of now. Every Nylon Pay delivery (including retries)
   stamps and signs a current timestamp, so a captured `(body, signature)` pair
   goes stale and a replay is rejected, while legitimate delayed retries, each
   freshly stamped, still pass. The timestamp is read from the signed body, not
   from a header, so it cannot be refreshed without the secret. When
   `toleranceSeconds > 0` and the signed body carries no parseable timestamp,
   verification fails closed.

See [D16](./decision-records.md#d16-webhook-verification-is-replay-protected).

