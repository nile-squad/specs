# Nylon Pay SDK Spec

**Version:** 1.0.0

> Canonical, language-agnostic specification for the Nylon Pay SDK. Implement it
> in any language; the [TypeScript SDK](https://github.com/nile-squad/nylonpay-ts)
> is the reference implementation.

## Purpose

The Nylon Pay SDK is the merchant's programmatic interface to the payment platform. It provides a consistent API across multiple languages for collecting payments, making payouts, verifying phones, creating invoices, checking transaction status, and verifying webhooks. The SDK abstracts the transport protocol, HMAC signing, polling mechanics, and error handling so merchants interact with payment operations — not infrastructure.

**Target languages:** TypeScript, Python, Rust, Go, C#, PHP, Kotlin, Elixir.

All SDKs are server-side. Client-side packages (browser, mobile) are a future scope.

## Principles

1. **Functional core, idiomatic shell.** Every SDK exposes a factory function that returns an object/struct of functions. Prefer functional patterns (plain objects, closures, composition) in every language. Classes are permitted only when the language lacks idiomatic functional alternatives — the end-user DX must remain identical regardless of internal implementation.

2. **Same names, same shapes, same events.** Function names, parameter names, event names, and status values are consistent across all implementations. A merchant reading TypeScript docs can write Python code without relearning the API. Only casing conventions differ per language (`collectPayment` vs `collect_payment` vs `CollectPayment`).

3. **Named parameters everywhere.** Every operation takes a single configuration object/struct/dict. No positional arguments beyond the factory. This makes the API self-documenting and resistant to parameter ordering bugs.

4. **Signing is invisible.** HMAC-SHA256 request signing, nonce generation, fingerprinting, and response verification happen automatically inside the transport layer. The merchant never touches crypto primitives.

5. **Async operations return a PaymentInstance.** Long-running operations (collections, payouts) return an event-driven instance with pubsub (`on`/`once`/`off`) and a blocking resolver (`wait`). The instance owns its polling lifecycle and stops on terminal states.

6. **Fail loudly on misconfiguration, gracefully on runtime errors.** A missing or malformed API key/secret throws at initialization. Network failures, timeouts, and provider rejections return error results. The boundary is: programmer error vs. operational failure.

7. **Idempotency is automatic but overridable.** Every payment operation generates a unique idempotency key by default. Merchants supply their own for retry-safe operations. The SDK never sends the same key twice unintentionally.

8. **Stateless between calls.** The SDK holds no session, connection pool, or persistent state between operations. The only stateful construct is the PaymentInstance, which is explicitly scoped to one transaction and disposable.

9. **Server-side only.** The SDK runs in server environments. It uses OS-level fingerprinting and server-grade crypto primitives.

10. **Spec is the ceiling.** No SDK implementation adds operations, parameters, events, or behavior beyond what this spec defines. If a feature is needed, the spec updates first — then implementations follow.

## Decision Records

### D1: PubSub is a first-class primitive

- **Decision** — Every SDK implementation includes a pubsub system for PaymentInstance events. Languages without native event emitters implement a minimal equivalent.
- **Context** — Async payment operations need status transition notifications. Polling in a loop is error-prone and wasteful.
- **Alternatives considered** — (a) Callbacks only (single handler per event). Rejected: multiple consumers need to react to the same event. (b) Channels/streams only (Go/Rust native). Rejected: inconsistent with other languages, harder for merchants switching between SDKs.
- **Rationale** — `on`/`once`/`off` is the universal event pattern. Every target language can implement it. Go uses goroutine-safe handler registration. Rust uses trait-based handlers. Elixir uses message passing. The API surface is identical.
- **Tradeoffs** — Some languages carry synchronization overhead for thread-safe pubsub. Acceptable — payment operations are I/O-bound, not CPU-bound.

### D2: Payouts use the same async primitive as collections

- **Decision** — `makePayout` returns a PaymentInstance with the same event-driven interface as `collectPayment`.
- **Context** — Collections and payouts are both async operations that resolve over time. Merchants need the same DX for both.
- **Alternatives considered** — (a) Return a sync result for payouts and require manual `getStatus` calls. Rejected: inconsistent DX, forces polling logic onto every merchant. (b) A different async primitive for payouts. Rejected: two async patterns is worse than one.
- **Rationale** — One async primitive for all long-running operations. The PaymentInstance polls internally using the returned reference. Merchants learn one pattern and apply it everywhere.
- **Tradeoffs** — Payout confirmations may be slower than collections depending on the provider. Merchants should use webhooks as the authoritative source for payout completion.

### D3: Webhook event types are defined in the SDK spec

- **Decision** — The SDK spec defines the webhook event catalog and verification utility.
- **Context** — Merchants need to know what events to expect and how to verify them. The verification function is an SDK export, and event types are part of the SDK's type surface.
- **Rationale** — Webhooks are the server-push complement to the SDK's client-pull model. They belong together so merchants have one document for the full integration surface.
- **Tradeoffs** — The spec is slightly larger. Offset by keeping all merchant-facing contracts in one place.

### D4: Fingerprint uses OS-level metadata

- **Decision** — The SDK generates a server fingerprint from stable OS and runtime metadata.
- **Context** — The signing protocol includes a fingerprint in the signature payload for server-side device identification and anomaly detection.
- **Alternatives considered** — (a) No fingerprint. Rejected: the server uses it for anomaly detection. (b) Random per-process ID. Rejected: loses device identification value.
- **Rationale** — OS and runtime metadata is stable per-process, unique per-server, and available in all target languages without external dependencies.
- **Tradeoffs** — Hostname may change in containerized environments. Acceptable: the fingerprint is one signal among many, not a sole identifier.

### D5: POST-only action-based transport

- **Decision** — All SDK requests are POST to `{baseUrl}/{action}` with JSON body.
- **Context** — The backend uses action-based routing. The SDK mirrors this architecture directly.
- **Rationale** — Action-based transport eliminates a mapping layer between SDK and server. The SDK abstracts URLs entirely — merchants call functions, not endpoints.
- **Tradeoffs** — Merchants familiar with REST may find action-based URLs unfamiliar. Offset by the SDK hiding URLs behind function calls.

### D6: Classes permitted only when functional is not idiomatic

- **Decision** — SDK implementations prefer functional patterns. Classes are allowed only when the language lacks idiomatic functional alternatives, and the end-user API must remain identical to functional implementations.
- **Context** — Some target languages (C#, Kotlin, PHP) have class-centric ecosystems where pure functional patterns are awkward or non-idiomatic.
- **Alternatives considered** — (a) Ban classes entirely. Rejected: forces unnatural patterns in some languages. (b) Allow classes freely. Rejected: leads to inconsistent DX across languages.
- **Rationale** — The merchant's experience is the invariant. Whether the SDK internally uses closures, structs, or classes is an implementation detail. What matters is that `nylon.collectPayment({ ... })` looks and behaves the same everywhere.
- **Tradeoffs** — Reviewers must verify that class-based implementations expose the same API surface as functional ones.

### D7: Card payments via hosted page only

- **Decision** — Card payments are only available through `createInvoice` (hosted payment page). The SDK does not accept card details directly.
- **Context** — Collecting card data (card number, CVV, expiry) imposes PCI-DSS compliance obligations on merchants. Most merchants should not handle raw card data.
- **Alternatives considered** — Exposing card fields in `collectPayment` with tokenization. Rejected: shifts PCI scope to the merchant's server, increases integration complexity, and creates a security surface that most merchants are not equipped to manage.
- **Rationale** — The hosted payment page handles card collection, 3DS, and PCI compliance on behalf of the merchant. Mobile money and bank transfers — which don't carry PCI obligations — are supported directly in the SDK.
- **Tradeoffs** — Merchants who want a fully embedded card checkout experience must use the hosted page redirect. This is an acceptable constraint given the security implications.

### D8: Provider routing is server-side, not merchant-facing

- **Decision** — The SDK does not expose provider selection, provider names, or provider references. The backend routes transactions to the appropriate provider based on merchant configuration, routing rules, and gateway health.
- **Context** — Nylon Pay operates as a payment gateway that abstracts provider complexity from merchants. Merchants specify what they want to do (collect, pay out) — not how or through whom.
- **Alternatives considered** — (a) Allow merchants to specify a preferred provider. Rejected: couples merchants to provider-specific integrations, defeats the gateway abstraction. (b) Return provider name in transaction responses. Rejected: leaks internal routing, creates support burden when providers change.
- **Rationale** — Provider routing is an internal concern. The transaction response contains only what the merchant needs: status, amount, reference, failure reason. Provider details are sanitized out.
- **Tradeoffs** — Merchants cannot debug provider-specific issues directly. This is by design — provider issues are Nylon Pay's responsibility, not the merchant's.

### D9: Resolve variants for async operations

- **Decision** — `collectPaymentAndResolve` and `makePayoutAndResolve` are provided alongside their async counterparts. They block until the transaction reaches a terminal state and return the transaction directly.
- **Context** — Some integration patterns (CLI tools, serverless functions, simple scripts) don't benefit from event-driven polling. They need a synchronous "send and wait" operation.
- **Alternatives considered** — (a) Only provide async + `wait()`. Rejected: forces every synchronous caller to chain two calls. (b) Make all operations synchronous by default. Rejected: loses the event-driven pattern that most server integrations need.
- **Rationale** — Two patterns for two use cases. Event-driven for long-lived server processes. Blocking resolve for simple integrations. Both use the same underlying transport and polling — the resolve variant is a convenience wrapper.
- **Tradeoffs** — Slightly larger API surface. Offset by the clarity of having one function per use case instead of requiring composition.

### D10: Lifecycle hooks for cross-cutting concerns

- **Decision** — The SDK exposes optional `beforeCollect`, `afterCollect`, `beforePayout`, and `afterPayout` hooks registered once at initialization via `NylonPayConfig.hooks`.
- **Context** — Merchants need to intercept payment calls for logging, analytics, metadata enrichment, and audit trails without wrapping every SDK call in their own middleware. Before we had hooks, merchants had to wrap every `collectPayment` call manually — leading to duplicated logic and missed edge cases (e.g., the error path not being instrumented).
- **Alternatives considered** — (a) Middleware stack (array of functions applied in sequence). Rejected: unnecessary complexity when a single hook per event is sufficient; arrays imply ordering and composition semantics that create confusion. (b) Event emitter on the SDK instance. Rejected: event emitters are appropriate for repeated events, not single-shot lifecycle points; `afterCollect` fires exactly once per call, not on a recurring bus.
- **Rationale** — One hook per lifecycle point. `before*` hooks allow payload enrichment (adding metadata, normalizing fields). `after*` hooks allow observability (logging, analytics) regardless of outcome. Hooks run synchronously in the call chain — `before*` is awaited before transport, `after*` is awaited before returning to the caller.
- **Tradeoffs** — Hook errors propagate to the caller, which means a broken hook breaks the payment call. This is intentional: silent hook failures would create hard-to-debug divergence between what the hook logged and what actually happened.

## Operations

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
- `reference?` — merchant-supplied idempotency key (auto-generated if omitted)
- `method?` — payment method: `"mobileMoney"` or `"bank"` (defaults to `"mobileMoney"`)
- `bank?` — required when `method` is `"bank"`: `{ accountNumber, bankName }`
- `metadata?` — arbitrary key-value pairs attached to the transaction

Returns: `PaymentInstance`

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
- `reference?` — idempotency key
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

Returns: full transaction record (see [Transaction Shape](#transaction-shape))

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
- `reference?` — idempotency key
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

Standalone utility. Verifies that a webhook payload was signed by Nylon Pay.

Input shape:
- `payload` — raw request body (string or bytes, depending on language)
- `signature` — signature from webhook header
- `secret` — merchant's webhook secret

Returns: boolean

## PaymentInstance Contract

Every async operation returns a PaymentInstance with this interface:

**Properties:**
- `reference` — the transaction reference (read-only)
- `status` — current transaction status, null until first poll resolves (read-only)

**Methods:**
- `on(event, handler)` — register a handler for an event. Returns the instance (chainable).
- `once(event, handler)` — register a handler that fires at most once. Returns the instance.
- `off(event, handler)` — remove a handler. Returns the instance.
- `wait()` — blocks until a terminal state. Resolves/returns the transaction on success. Rejects/errors on failure, cancellation, or timeout.

**Events:**
- `processing` — provider acknowledged, transaction in progress
- `success` — transaction completed successfully (terminal, polling stops)
- `failed` — transaction failed (terminal, polling stops)
- `cancelled` — transaction cancelled (terminal, polling stops)
- `error` — non-transaction error: network failure, reference mismatch, polling timeout (terminal, polling stops)

**Polling behavior:**
- Polls `getStatus` at configurable interval (default 2s)
- Stops on any terminal event
- Stops after max attempts (default 150) or max duration (default 5 minutes)
- Reference mismatch between initiation and poll response emits `error` and stops
- Network errors during polling emit `error` and stop (after retries exhausted)

## Type Definitions

TypeScript types are used below as the reference notation. Each language defines equivalent types using its own type system. The shapes, field names, and value constraints are identical across all implementations.

```typescript
type TransactionStatus =
  | "pending"
  | "processing"
  | "successful"
  | "failed"
  | "cancelled";

type TransactionType =
  | "collection"
  | "payout"
  | "transfer"
  | "escrow"
  | "refund"
  | "reversal"
  | "charge"
  | "chargeback";

type PaymentMethod = "mobileMoney" | "bank";

type TransactionMode = "test" | "live";

type PaymentEvent =
  | "processing"
  | "success"
  | "failed"
  | "cancelled"
  | "error";

type WebhookEventType =
  | "collection.completed"
  | "collection.failed"
  | "payout.completed"
  | "payout.failed"
  | "payout.reversed"
  | "refund.completed"
  | "chargeback.received";

type Currency = "USD" | "EUR" | "GBP" | "KES" | "UGX" | "TZS" | "RWF";

type SdkHooks = {
  beforeCollect?: (
    input: CollectPaymentInput,
  ) => CollectPaymentInput | void | Promise<CollectPaymentInput | void>;
  afterCollect?: (
    result: Result<{ reference: string; status: string }, string>,
    input: CollectPaymentInput,
  ) => void | Promise<void>;
  beforePayout?: (
    input: MakePayoutInput,
  ) => MakePayoutInput | void | Promise<MakePayoutInput | void>;
  afterPayout?: (
    result: Result<{ reference: string; status: string }, string>,
    input: MakePayoutInput,
  ) => void | Promise<void>;
};

type NylonPayConfig = {
  apiKey: string;
  apiSecret: string;
  baseUrl?: string;
  timeoutMs?: number;
  maxRetries?: number;
  maxPollIntervalMs?: number;
  maxPollDurationMs?: number;
  maxPollAttempts?: number;
  hooks?: SdkHooks;
};

type Customer = {
  name: string;
  phoneNumber: string;
  email?: string;
};

type Destination = {
  accountHolderName: string;
  accountNumber: string;
  bankName?: string;
  phone?: string;
};

type InvoiceItem = {
  name: string;
  quantity: number;
  unitPrice: number;
};

type BankDetails = {
  accountNumber: string;
  bankName: string;
};

type CollectPaymentInput = {
  amount: number;
  currency: Currency;
  customer: Customer;
  description: string;
  reference?: string;
  method?: PaymentMethod;
  bank?: BankDetails;
  metadata?: Record<string, string>;
};

type MakePayoutInput = {
  amount: number;
  currency: Currency;
  customer: Customer;
  destination: Destination;
  description: string;
  reference?: string;
  metadata?: Record<string, string>;
};

type GetStatusInput = {
  reference: string;
};

type GetTransactionInput = {
  id?: string;
  reference?: string;
};

type VerifyPhoneInput = {
  phoneNumber: string;
  purpose?: "collection" | "payout";
};

type CreateInvoiceInput = {
  amount: number;
  currency: Currency;
  description: string;
  items?: InvoiceItem[];
  redirectUrl?: string;
  reference?: string;
  metadata?: Record<string, string>;
};

type VerifyWebhookInput = {
  payload: string | Uint8Array;
  signature: string;
  secret: string;
};

type Transaction = {
  id: string;
  reference: string;
  amount: number;
  currency: Currency;
  status: TransactionStatus;
  type: TransactionType;
  method: PaymentMethod;
  description: string;
  phone: string;
  email: string | null;
  failureReason: string | null;
  metadata: Record<string, string>;
  mode: TransactionMode;
  createdAt: string;
  updatedAt: string;
};

type StatusResponse = {
  reference: string;
  status: TransactionStatus;
  amount: number;
  currency: Currency;
  updatedAt: string;
};

type PhoneVerification = {
  phoneNumber: string;
  customerName: string;
  verified: boolean;
};

type InvoiceResponse = {
  id: string;
  url: string;
  token: string;
  expiresAt: string;
  status: "pending";
};

type WebhookPayload = {
  event: WebhookEventType;
  data: Transaction;
  timestamp: string;
  signature: string;
};
```

## Transaction Shape

The transaction record returned by `getTransaction`, `wait()`, and event handlers. See `Transaction` type above for the full shape.

## Webhook Event Catalog

Events are delivered as POST requests to the merchant's configured webhook URL. Each event has a `type` field identifying the event and a `data` field containing the relevant transaction record.

| Event Type | Trigger |
|------------|---------|
| `collection.completed` | A collection transaction reaches `successful` status |
| `collection.failed` | A collection transaction reaches `failed` status |
| `payout.completed` | A payout transaction reaches `successful` status |
| `payout.failed` | A payout transaction reaches `failed` status |
| `payout.reversed` | A failed payout is reversed by the system |
| `refund.completed` | A refund transaction is processed successfully |
| `chargeback.received` | A chargeback is initiated by a bank or provider |

**Webhook payload shape:** See `WebhookPayload` type above.

**Delivery guarantees:**
- At-least-once delivery with exponential backoff retries
- Merchants must respond with 2xx within the timeout window
- Duplicate delivery is possible — merchants must be idempotent (use `reference` for deduplication)

## Transport Contract

The Nylon Pay backend uses Nile.js action-based routing. All SDK requests target a single endpoint and carry structured payloads that identify the service and action being invoked.

### Endpoint

All requests are `POST` to `{baseUrl}/services`.

There are no RESTful routes, no query parameters, no HTTP method variety. Every operation — regardless of type — hits the same endpoint with a different JSON body.

### Request Format

Every request body has this shape:

```
{
  "intent": "execute",
  "service": "<service-name>",
  "action": "<action-name>",
  "payload": { ... }
}
```

- `intent` — always `"execute"` for SDK operations
- `service` — the server-side service group (e.g., `"collections"`, `"payouts"`, `"payment-links"`)
- `action` — the specific operation within the service (e.g., `"create-collection"`, `"get-collection-status"`)
- `payload` — the operation's input data, plus `_fingerprint` injected by the SDK

The SDK maps each public operation to a service/action pair:

| SDK Operation | Service | Action |
|---------------|---------|--------|
| `collectPayment` | `sdk` | `sdk-collect-payment` |
| `collectPaymentAndResolve` | `sdk` | `sdk-collect-payment-and-resolve` |
| `getStatus` | `sdk` | `sdk-get-status` |
| `getTransaction` | `sdk` | `sdk-get-transaction` |
| `makePayout` | `sdk` | `sdk-make-payout` |
| `makePayoutAndResolve` | `sdk` | `sdk-make-payout-and-resolve` |
| `verifyPhone` | `sdk` | `sdk-verify-phone` |
| `createInvoice` | `sdk` | `sdk-create-invoice` |

### Response Format

Every server response has this shape:

```
{
  "status": true | false,
  "message": "<human-readable description>",
  "data": { ... }
}
```

- `status` — `true` for success, `false` for error
- `message` — human-readable description of the outcome
- `data` — the result payload on success, or empty object `{}` on error

The SDK normalizes this to a result type: on success, returns the `data` payload. On failure, returns an error derived from `message`.

### Request Signing

Every request is signed with HMAC-SHA256. The signing protocol:

```
canonicalPayload = JSON.stringify(payload, sortedKeys)
signatureInput = fingerprint + "." + nonce + "." + timestamp + "." + canonicalPayload
signature = HMAC-SHA256(apiSecret, signatureInput)
```

The `canonicalPayload` must use deterministic key sorting so that two identical payloads produce identical signatures regardless of field insertion order.

**Request headers:**
- `x-nylon-key` — API key (plaintext, starts with `npk_`)
- `x-nylon-nonce` — 32-character hex nonce (unique per request, from cryptographic random bytes)
- `x-nylon-timestamp` — millisecond timestamp as string
- `x-nylon-signature` — computed HMAC signature (hex-encoded)

**Request body additions:**
- `_fingerprint` — SHA-256 hash of OS and runtime metadata, injected into every authenticated request body

### Response Verification

The server signs every response to prevent tampering:

- Response body includes a `_responseSignature` field
- SDK strips the `_responseSignature` field from the response
- SDK recomputes `HMAC-SHA256(apiSecret, canonicalPayload)` over the remaining payload
- SDK compares the computed signature against the received signature using constant-time comparison
- Mismatch = tampered response = error

### Retry Policy

- Retry on HTTP status codes: 408, 429, 500, 502, 503, 504
- Retry on network errors and request timeouts
- Do NOT retry on 4xx errors except 408 and 429 (client errors like 400, 401, 403, 404, 422 are returned immediately)
- Exponential backoff: `2^attempt * 1000 + random(0-500)` ms
- Max retries: configurable (default 3)
- Per-request timeout: configurable (default 30s), enforced via AbortController equivalent
- On retry, the same idempotency key is reused — the SDK must not generate a new nonce/key per retry attempt

## Configuration

See `NylonPayConfig` type above.

**Validation at init:**
- `apiKey` must be present and start with `npk_`
- `apiSecret` must be present and start with `nps_`
- Invalid config throws immediately (programmer error)

**Test vs. live mode:** Mode is determined by the API key, not by SDK config. A
sandbox key (issued in test mode) routes transactions through test providers and
does not move real money; a live key processes real transactions. The SDK has no
`environment` option.

**Defaults:**

| Field | Default |
|-------|---------|
| `baseUrl` | `https://api.nylonpay.io/api/services` |
| `timeoutMs` | `30000` |
| `maxRetries` | `3` |
| `maxPollIntervalMs` | `2000` |
| `maxPollDurationMs` | `300000` |
| `maxPollAttempts` | `150` |

## Implementation Requirements

Every SDK implementation must satisfy these requirements:

### Typing

Languages with type systems (TypeScript, Rust, Go, C#, Kotlin) must expose all types as public exports. Untyped languages (PHP) must provide type annotations via docblocks or equivalent. Python must provide type hints and ship with `py.typed` marker.

### Documentation

Every public function, type, and constant must have a docstring/JSDoc/KDoc/doc comment. Documentation explains what the function does and why a merchant would call it — not implementation detail.

### Tests

Every SDK must ship a test suite covering:
- Configuration validation (valid config, missing fields, invalid prefixes)
- Request signing (canonical payload, HMAC computation, nonce generation)
- Response verification (valid signature, tampered response)
- Each operation's happy path (mocked transport)
- PaymentInstance lifecycle (events, polling, terminal states, timeout, reference mismatch)
- Retry behavior (retryable status codes, non-retryable status codes, backoff timing)
- Webhook signature verification (valid, invalid, tampered)

### Edge Case Testing

Every SDK must test the following edge cases:

**Input validation:**
- Zero or negative amounts
- Empty strings for required fields (name, description, phoneNumber)
- Currency codes not in the supported set
- `getTransaction` with neither `id` nor `reference`
- `getTransaction` with both `id` and `reference`
- `collectPayment` with `method: "bank"` but missing `bank` field
- `createInvoice` with zero items or items with negative quantity/unitPrice

**Network and transport:**
- Connection timeout (server unreachable)
- DNS resolution failure
- TLS handshake failure
- Server returns non-JSON response body
- Server returns empty response body
- Server returns 5xx with retryable and non-retryable variants
- Server returns 401 (invalid credentials)
- Server returns 422 (validation error)
- Request timeout mid-stream (connection established but response never arrives)

**Signing and security:**
- Canonical payload with nested objects and arrays (deterministic ordering)
- Canonical payload with unicode characters in values
- Response signature missing from server response
- Response signature from a different secret
- Response signature over partial payload (some fields stripped)
- Nonce uniqueness across rapid sequential requests

**Polling and PaymentInstance:**
- First poll returns `not found` (server hasn't propagated yet)
- Status transitions: `pending` → `processing` → `successful`
- Status transitions: `pending` → `failed` with failure reason
- Status transitions: `pending` → `cancelled`
- Reference mismatch on second poll (server returns different reference)
- Polling timeout (max attempts exhausted)
- Polling timeout (max duration exhausted)
- Network error during polling (after first successful poll)
- Calling `wait()` after terminal state already reached
- Calling `off()` for a handler that was never registered
- Calling `once()` then manually calling `off()` before event fires

**Idempotency:**
- Two calls with the same merchant-supplied `reference`
- Auto-generated references are unique across rapid sequential calls
- Retry after 500 uses the same idempotency key (not a new one)

### Spec Compliance

No SDK adds operations, parameters, events, or behavior beyond what this spec defines. If a feature is needed across SDKs, this spec updates first — then all implementations follow. Language-specific conveniences (e.g., helper functions that compose existing operations) are permitted only if they do not introduce new server interactions or alter the documented contract.

## Invariants

1. Every SDK operation routes through the signed transport layer. No operation bypasses signing.
2. The PaymentInstance stops polling on any terminal event (`success`, `failed`, `cancelled`, `error`). It never polls indefinitely.
3. The canonical payload in signing uses deterministic key ordering. Two identical payloads produce identical canonical strings regardless of field insertion order.
4. Response signature verification uses constant-time comparison. Timing attacks on signature validation are not possible.
5. The factory validates configuration eagerly. An SDK instance with invalid credentials cannot be constructed.
6. Auto-generated idempotency keys are unique per invocation. The probability of collision is negligible.
7. The `_fingerprint` field is injected into every authenticated request body. No authenticated request is sent without it.
8. Webhook signature verification operates on the raw payload bytes, not parsed JSON. Re-serialization must not alter the signed content.
9. The PaymentInstance `reference` property is immutable after creation. It cannot be reassigned or mutated.
10. Handler errors in the pubsub system are caught and do not propagate to other handlers or the polling loop.
11. All public types, functions, and constants are documented. No undocumented public surface exists.
12. `before*` hooks run after input validation and before the transport call. They cannot bypass validation. `after*` hooks run after the transport call, regardless of success or failure. Both are awaited synchronously in the call chain.
13. A `before*` hook that returns `null` or `void` leaves the payload unchanged. Only a non-null return value replaces the input. Hook errors propagate to the caller — they are not caught or swallowed by the SDK.

## Prohibitions

1. The SDK never stores API secrets on disk, in logs, or in error messages. Secrets exist only in memory for the lifetime of the SDK instance.
2. The SDK never sends API secrets in request bodies or query parameters. Secrets are used only for HMAC computation.
3. The SDK never exposes raw provider payloads, provider names, provider references, internal transaction IDs, or idempotency keys in public return types. These are stripped during sanitization.
4. The SDK never retries on 4xx errors except 408 and 429. Client errors (400, 401, 403, 404, 422) are returned immediately to the caller.
5. The SDK never makes concurrent polls for the same PaymentInstance. Only one poll is in-flight at a time. The next poll starts only after the previous completes.
6. The SDK never exposes the fingerprint generation algorithm as a public API. It is an internal transport concern.
7. The SDK never modifies merchant-supplied metadata. It passes through to the server and back unchanged.
8. The SDK never throws on runtime/operational errors (network failures, provider rejections, timeouts). These are returned as error results. Only programmer errors (invalid config, missing required fields) throw.
9. Hooks never receive or expose API secrets, raw provider payloads, or internal transport state. The `before*` and `after*` hook signatures are bounded to the merchant-facing input and a normalized result — nothing more.

## Follow-Up Work

### F1: Card payments via SDK

- **What** — Card payments are only available through `createInvoice` (hosted payment page). The SDK does not accept card details (card number, CVV, expiry) directly.
- **Why it's deferred** — Merchants should not handle raw card data due to PCI-DSS compliance requirements. The hosted payment page handles card collection securely on behalf of the merchant.
- **Impact of deferring** — Merchants collecting card payments use `createInvoice` to generate a hosted payment link. This is the supported path, not a gap.

### F2: Batch operations

- **What** — No batch API for multiple collections or payouts in a single call.
- **Why it's deferred** — No server-side batch endpoint exists. Client-side batching (loop + concurrency control) adds complexity without atomicity guarantees.
- **Impact of deferring** — Merchants making bulk payouts loop over `makePayout` individually.

### F3: Client-side SDKs

- **What** — No browser or mobile SDK exists. All current SDKs are server-side.
- **Why it's deferred** — Client-side SDKs require a different security model (no secret key in client), tokenization endpoints, and hosted checkout integration.
- **Impact of deferring** — Client-side payment collection uses the hosted payment page via `createInvoice`.

### F4: SDK version header

- **What** — The SDK does not send its version in request headers for server-side compatibility checks.
- **Why it's deferred** — No server-side version negotiation exists. All SDK versions target the same API contract.
- **Impact of deferring** — Breaking API changes require coordinated SDK updates. Mitigated by semantic versioning and changelog communication.
