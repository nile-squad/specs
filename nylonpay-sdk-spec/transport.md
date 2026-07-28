# Transport Contract

Part of the [Nylon Pay SDK Spec](./spec.md).

The Nylon Pay backend uses Nile.js action-based routing. All SDK requests target a single endpoint and carry structured payloads that identify the service and action being invoked.

### Endpoint

All requests are `POST` to `{baseUrl}`.

The default `baseUrl` is `https://api.nylonpay.nilesquad.com/api/services` — a single URL that includes both the origin and the path. The SDK appends no further path segments; the body identifies the service and action. Every SDK MUST default to this URL and MUST allow the merchant to override it in configuration — some merchants run against a custom base URL for special integrations, but the default is what ships.

There are no RESTful routes, no query parameters, no HTTP method variety. Every operation — regardless of type — hits the same endpoint with a different JSON body.

### Request Format

Every request body has this shape:

```
{
  "intent": "execute",
  "service": "sdk",
  "action": "<action-name>",
  "payload": { ... }
}
```

- `intent` — always `"execute"` for SDK operations
- `service` — always `"sdk"`: every SDK operation targets the `sdk` service, the merchant-facing API surface.
- `action` — the specific operation within the `sdk` service (the `sdk-*` names in the table below)
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

### Wire Serialization Rules

Optional fields whose value is absent (None/null/undefined) MUST be omitted from
the JSON body entirely — they MUST NOT be serialized as `null`. The canonical
payload for signing (JCS, see Request Signing) operates on the serialized body,
so sending `null` where an absent field is expected changes the signed content
and breaks signature verification.

### Action Payloads

What the backend's `sdk` service accepts per action, as enforced by its validation
schemas. This is the integration contract for anyone implementing an SDK (or calling
the backend directly): a payload that violates these rules is rejected with a
`validation` error before any payment work happens. SDKs MUST mirror the cheap
synchronous checks client-side (amount, reference length, required fields) so bad
input fails before a network round-trip; the server remains the source of truth.

The resolve variants (`sdk-collect-payment-and-resolve`, `sdk-make-payout-and-resolve`)
accept exactly the same payload as their base actions.

**`sdk-collect-payment`** (and `-and-resolve`):

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `amount` | number | yes | minimum 500 UGX |
| `currency` | string | no | defaults to `"UGX"` |
| `customer.name` | string | yes | |
| `customer.phoneNumber` | string | yes | validated and normalized to international format (`256XXXXXXXXX`) |
| `customer.email` | string | no | |
| `description` | string | yes | |
| `method` | string | no | `"mobileMoney"` or `"bank"`; defaults to `"mobileMoney"` |
| `bank.accountNumber` | string | with `method: "bank"` | |
| `bank.bankName` | string | with `method: "bank"` | |
| `reference` | string | no | 13–15 characters (see [Reference constraints](./operations.md#reference-constraints)) |
| `metadata` | object | no | string keys to string values; defaults to `{}` |

**`sdk-make-payout`** (and `-and-resolve`):

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `amount` | number | yes | minimum 5000 UGX |
| `currency` | string | no | defaults to `"UGX"` |
| `customer.name` | string | yes | |
| `customer.phoneNumber` | string | yes | validated and normalized to international format (`256XXXXXXXXX`) |
| `customer.email` | string | no | |
| `destination.accountHolderName` | string | yes | |
| `destination.accountNumber` | string | yes | |
| `destination.bankName` | string | no | |
| `destination.phone` | string | no | |
| `description` | string | yes | |
| `reference` | string | no | 13–15 characters |
| `metadata` | object | no | string keys to string values; defaults to `{}` |

**`sdk-get-status`**:

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `reference` | string | yes | non-empty |

**`sdk-get-transaction`**:

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `id` | string | no | at least one of `id`/`reference` required |
| `reference` | string | no | at least one of `id`/`reference` required |

**`sdk-verify-phone`**:

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `phoneNumber` | string | yes | validated and normalized to international format (`256XXXXXXXXX`) |
| `purpose` | string | no | `"collection"` or `"payout"` |

**`sdk-create-invoice`**:

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `amount` | number | yes | minimum 500 UGX |
| `currency` | string | no | defaults to `"UGX"` |
| `customerEmail` | string | yes | |
| `customerName` | string | no | |
| `customerPhone` | string | no | validated and normalized to international format (`256XXXXXXXXX`) |
| `description` | string | no | |
| `dueDate` | string | no | |
| `items[]` | array | no | max 50 of `{ name: string, quantity: number > 0, unitPrice: number > 0 }` |
| `merchantReference` | string | no | 13–15 characters |
| `tags[]` | array | no | |
| `metadata` | object | no | string keys to string values; defaults to `{}` |

Invoices are live-mode only: calling `sdk-create-invoice` with a sandbox (test-mode)
API key returns a `validation` error.

### Example Exchange

A complete `sdk-collect-payment` call. The four `x-nylon-*` headers are computed per
[Request Signing](#request-signing); `_fingerprint` is injected into the payload by the
SDK's transport layer.

Request:

```
POST https://api.nylonpay.nilesquad.com/api/services
content-type: application/json
x-nylon-key: npk_live_...
x-nylon-nonce: 3f9c1a7e5b2d48c6a0e8f4b1d7c92e50
x-nylon-timestamp: 1781136000000
x-nylon-signature: <hex HMAC-SHA256 of fingerprint.nonce.timestamp.canonicalPayload>

{
  "intent": "execute",
  "service": "sdk",
  "action": "sdk-collect-payment",
  "payload": {
    "amount": 5000,
    "currency": "UGX",
    "customer": { "name": "Jane Doe", "phoneNumber": "+256700000000" },
    "description": "Order payment",
    "method": "mobileMoney",
    "reference": "ORDER-2026-001",
    "metadata": { "orderId": "12345" },
    "_fingerprint": "<transport-generated fingerprint>"
  }
}
```

Success response (HTTP `200`):

```
{
  "status": true,
  "message": "OK",
  "data": {
    "reference": "ORDER-2026-001",
    "status": "pending",
    "transactionId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "createdAt": "2026-06-11T09:30:00.000Z",
    "_requestNonce": "3f9c1a7e5b2d48c6a0e8f4b1d7c92e50",
    "_responseSignature": "<hex HMAC-SHA256 over data minus this field>"
  }
}
```

The SDK verifies `_responseSignature`, checks that `_requestNonce` matches the
`x-nylon-nonce` it sent, strips both, and returns the rest of `data` to
the merchant. The transaction starts at `"pending"`; the PaymentInstance polls
`sdk-get-status` with the reference until a terminal status.

Error response (HTTP `400`):

```
{
  "status": false,
  "message": "Amount must be at least 500 -- error-type: validation",
  "data": {}
}
```

The SDK splits the ` -- error-type: ` suffix into the structured error's `category`
(here `validation`) and keeps the human-readable part as the message.

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

**HTTP is binary.** The backend returns HTTP `200` for success (`status: true`) and HTTP `400` for every failure (`status: false`) — regardless of cause. The SDK MUST NOT branch on HTTP status codes to classify errors. Provider/transport-level failures the backend never produced (network errors, request timeouts, gateway 5xx) are surfaced by the SDK's transport with the `network`/`timeout`/`internal` categories.

**Error categories.** Every server failure carries a machine-readable category so merchants branch on a stable value instead of parsing prose. Because the framework discards the response `data` on failures and only the `message` survives, the category travels as a suffix on the message:

```
<human-readable message> -- error-type: <category>
```

The SDK splits the suffix off, exposing `category` and the clean `message` on the structured `SdkError` (see [Error Categories](./errors.md#error-categories)). The human portion (including any server log id) is preserved as the message. A message without the suffix is treated as category `internal`.

### Request Signing

Every request is signed with HMAC-SHA256. The signing protocol:

```
canonicalPayload = JSON.stringify(payload, sortedKeys)
signatureInput = fingerprint + "." + nonce + "." + timestamp + "." + canonicalPayload
signature = HMAC-SHA256(apiSecret, signatureInput)
```

The `canonicalPayload` is the **JSON Canonicalization Scheme** (RFC 8785 / JCS)
serialization of the payload — see [D17](./decision-records.md#d17-canonical-payload-uses-the-json-canonicalization-scheme-jcs):
- Object keys are sorted by **Unicode code point** (UTF-16 code-unit order),
  recursively, at every level. The sort MUST NOT be locale-sensitive — a
  collation that depends on the runtime's locale or Unicode data (e.g. a
  `localeCompare`-style comparison) can order the same keys differently on
  different runtimes and break verification on valid traffic.
- Arrays keep their order (never sorted).
- Numbers use the shortest round-tripping form (ECMAScript number-to-string),
  strings use minimal JSON escaping, and there is no insignificant whitespace.

This makes the canonical string identical across languages and locales, so two
identical payloads produce identical signatures regardless of field insertion
order or where they were serialized.

**Request headers:**
- `x-nylon-key` — API key (plaintext, starts with `npk_`)
- `x-nylon-nonce` — 32-character hex nonce (unique per request, from cryptographic random bytes)
- `x-nylon-timestamp` — millisecond timestamp as string
- `x-nylon-signature` — computed HMAC signature, **lowercase hex** (the one canonical form; see invariant 28)

**Request body additions:**
- `_fingerprint` — SHA-256 hash of OS and runtime metadata, injected into every authenticated request body

### Response Verification

The server signs every response to prevent tampering, and binds it to the
request that solicited it (D21):

- Response body includes a `_responseSignature` field
- Response body includes a `_requestNonce` field — the nonce from the request being answered, covered by the signature
- SDK strips the `_responseSignature` field from the response
- SDK recomputes `HMAC-SHA256(apiSecret, canonicalPayload)` over the remaining payload (which still includes `_requestNonce`)
- SDK compares the computed signature against the received signature using constant-time comparison
- Mismatch = tampered response = error
- SDK then requires `_requestNonce` to equal the nonce it sent in `x-nylon-nonce`; a missing or different value is an `internal` error
- SDK strips `_requestNonce` before returning data to the caller

The signature alone proves who produced a response, not which call it answers.
Without the nonce binding, any response the server ever legitimately produced
stays validly signed forever and can be replayed onto a later request for the
same reference.

### Response Size Bounds

The SDK MUST enforce a maximum response body size. Responses whose body exceeds
the configured limit MUST be rejected as an `internal` error before signature
verification begins — the data is never read into memory beyond the limit. The
default limit is 10 MB. The limit MAY be configurable.

The cap MUST be enforced **while the body is read**, against a running byte
count, aborting as soon as it is exceeded. Checking a `Content-Length` header
alone does NOT satisfy this: it cannot bound peak memory if the HTTP client has
already buffered the body, and it is a no-op entirely when the server sends no
length (chunked transfer).

Rationale: response bodies are signed in full for verification. Without a size
bound, an oversized response could exhaust SDK memory during the read phase,
before signature verification ever runs. The limit prevents a server-side or
MITM resource-exhaustion attack against the SDK consumer.

### Retry Policy

- Retry on HTTP status codes: 408, 429, 500, 502, 503, 504
- Retry on network errors and request timeouts
- Do NOT retry on 4xx errors except 408 and 429 (client errors like 400, 401, 403, 404, 422 are returned immediately)
- Business failures are HTTP `400` and therefore never retried — they are returned (or thrown, for async initiation) immediately with their category
- Exponential backoff: `2^attempt * 1000 + random(0-500)` ms
- Max retries: configurable (default 3)
- Per-request timeout: configurable (default 30s), enforced via AbortController equivalent
- On retry, the request **body is unchanged** (same payload, same `reference`), but each attempt is **signed fresh** — a new `nonce`, `timestamp`, and `signature` per try. Idempotency is carried by the constant `reference` (see [D18](./decision-records.md#d18-the-reference-is-the-only-transaction-identity-no-separate-idempotency-key-no-heuristic-duplicate-detection)), not by reusing the nonce. Re-signing keeps a post-backoff retry inside the server's timestamp-freshness window and prevents a retry from being rejected as a nonce replay (see [D19](./decision-records.md#d19-retries-are-signed-fresh-per-attempt-reference-not-nonce-carries-idempotency))

### Status Polling

A PaymentInstance tracks status transitions by repeatedly calling the one-shot status operation until the transaction reaches a terminal state (see [D14](./decision-records.md#d14-status-updates-are-delivered-by-client-polling)).

- **Single-flight.** Only one status request is in flight per instance. The next poll is scheduled only after the current one resolves, so requests never overlap.
- **Jittered interval.** Each interval is the configured poll interval plus a small random jitter, so a fleet of concurrent instances does not synchronise into a thundering herd against the status endpoint.
- **De-duplication.** A status update that matches the instance's current status emits no event. Only a transition (`prev !== next`) emits.
- **Terminal stop.** On any terminal state the instance fetches the full transaction record, emits the terminal event, and stops. Optional `maxPollAttempts` and `maxPollDurationMs` caps bound a non-terminating transaction when the merchant sets them.
- **Delayed flag.** Status responses may include `delayed: true` when a payment has been non-terminal for more than ~3 minutes. When `onDelayed` is `"return"`, the instance resolves with the still-pending transaction; when `"wait"` (default), polling continues.
- **Backoff.** For the first two minutes, each interval is the configured poll interval plus jitter. After that, the interval doubles every two minutes up to a 15s cap.
- **Late-update guard.** Once an instance has resolved (terminal, error, or timeout) it emits no further events; an in-flight poll that resolves after that point is ignored.

