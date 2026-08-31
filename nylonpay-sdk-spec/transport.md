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
canonicalPayload = JCS(payload)
signatureInput   = fingerprint + "." + nonce + "." + timestamp + "." + canonicalPayload
signature        = HMAC-SHA256(key = apiSecret, message = signatureInput)   // lowercase hex
```

The HMAC key is the `apiSecret` string as raw UTF-8 bytes — it is not decoded,
hashed, or stripped of its `nps_` prefix first.

#### What exactly is signed

`payload` is the **inner `payload` object of the request envelope** — the
operation's input plus `_fingerprint` — and **NOT** the full envelope. The
`intent`, `service`, and `action` fields are outside the signature. Signing the
whole envelope is the single most common first-implementation mistake; it
produces a well-formed request that fails auth 100% of the time.

For the [Example Exchange](#example-exchange) above, the signed object is:

```json
{
  "amount": 5000,
  "currency": "UGX",
  "customer": { "name": "Jane Doe", "phoneNumber": "+256700000000" },
  "description": "Order payment",
  "method": "mobileMoney",
  "reference": "ORDER-2026-001",
  "metadata": { "orderId": "12345" },
  "_fingerprint": "<transport-generated fingerprint>"
}
```

The `fingerprint` component of `signatureInput` MUST be **byte-identical to the
`_fingerprint` value inside that payload**. The server does not receive the
fingerprint in a header: it reads `_fingerprint` out of the request body and
feeds that value into its own `signatureInput`. Signing one value and sending
another fails verification with an `auth` error and no further diagnostic.

#### Canonical payload (JCS)

The `canonicalPayload` is the **JSON Canonicalization Scheme** (RFC 8785 / JCS)
serialization of the payload — see [D17](./decision-records.md#d17-canonical-payload-uses-the-json-canonicalization-scheme-jcs):

- Object keys are sorted by **UTF-16 code unit**, recursively, at every level.
- Arrays keep their order (never sorted).
- Numbers use the shortest round-tripping form (ECMAScript number-to-string),
  strings use minimal JSON escaping, and there is no insignificant whitespace
  (no spaces after `:` or `,`).

This makes the canonical string identical across languages and locales, so two
identical payloads produce identical signatures regardless of field insertion
order or where they were serialized.

**Sorting MUST be by UTF-16 code unit, and nothing else.** Three ways to get
this wrong, all of which produce a signature the server cannot reproduce:

1. **Locale-sensitive collation** (`localeCompare`, `strcoll`, ICU collators).
   The order depends on the runtime's locale data, so the same payload
   canonicalizes differently on two machines.
2. **Sorting UTF-16 *little-endian* bytes.** Comparing UTF-16LE bytes compares
   the low byte first, which is not code-unit order: `"Ā"` (U+0100 → `00 01`)
   sorts before `"Z"` (U+005A → `5A 00`), while code-unit order puts `"Z"`
   first. If your language exposes UTF-16 only as bytes, encode **big-endian**
   (`utf-16-be`) — big-endian byte order and code-unit order coincide.
3. **Sorting by Unicode code *point* via UTF-8 bytes.** This agrees with
   code-unit order across the whole BMP but diverges above U+FFFF, where
   surrogates (U+D800–U+DFFF) sort below U+E000–U+FFFF. An emoji key sorts
   before `"�"` under JCS, and after it under code-point order.

Vector V7 below exists specifically to catch all three.

**Every numeric value on the wire MUST be an integer.** The JCS number rule is
defined in terms of ECMAScript's number-to-string, which no other language
reproduces for non-integers: the same value serializes as `5000` in JavaScript,
`5000.0` in Python, and `1.0e+21` vs `1e+21` in PHP for large magnitudes. All
amounts, quantities, and unit prices in this spec are whole units of the minor
currency, so SDKs sidestep the problem entirely by **rejecting non-integer
numbers during client-side validation** (see invariant 33) rather than trying to
match ECMAScript float formatting. An SDK that accepts `amount=5000.0` will sign
a canonical string the server cannot reproduce.

**Per-language JSON encoder settings.** Most standard encoders do not emit the
canonical form by default:

| Language | Required settings |
|----------|-------------------|
| JavaScript / TypeScript | `JSON.stringify` is already canonical — no options needed. |
| Python | `json.dumps(value, separators=(",", ":"), ensure_ascii=False)`. The default separators insert spaces; the default `ensure_ascii=True` escapes non-ASCII to `\uXXXX`. |
| PHP | `json_encode($value, JSON_UNESCAPED_UNICODE \| JSON_UNESCAPED_SLASHES)`. Without these, `/` becomes `\/` and non-ASCII becomes `\uXXXX`. Empty maps must be `stdClass`/`JSON_FORCE_OBJECT`, not `[]`. |
| Go | Set `Encoder.SetEscapeHTML(false)`. `encoding/json` escapes `<`, `>`, and `&` by default, and re-sorts map keys by code point — build an ordered structure yourself rather than relying on map iteration. |
| Java / C# | Disable HTML/non-ASCII escaping and any pretty-printing; serialize from an explicitly ordered map. |

Vector V4 below catches every one of these escaping defaults.

#### Request headers

- `x-nylon-key` — API key (plaintext, starts with `npk_`)
- `x-nylon-nonce` — 32-character hex nonce (unique per request, from cryptographic random bytes)
- `x-nylon-timestamp` — millisecond timestamp as a decimal string
- `x-nylon-signature` — computed HMAC signature, **lowercase hex** (the one canonical form; see invariant 28)

#### Request body additions

- `_fingerprint` — SHA-256 hash of OS and runtime metadata, injected into every
  authenticated request body. Its exact composition is an implementation choice
  (the server treats it as an opaque stable identifier); it MUST be a stable
  64-char lowercase hex value for the life of the process, and MUST match the
  `fingerprint` used in `signatureInput`.

#### Reference implementation

TypeScript (the reference implementation's actual signing path):

```typescript
import { createHmac } from "node:crypto";

/** Compare two keys by UTF-16 code unit (RFC 8785), never by locale. */
function compareByCodeUnit(first: string, second: string): number {
  if (first < second) return -1;
  if (first > second) return 1;
  return 0;
}

function sortValue(value: unknown): unknown {
  if (Array.isArray(value)) return value.map(sortValue);
  if (value && typeof value === "object") {
    const sorted = Object.entries(value as Record<string, unknown>)
      .sort(([a], [b]) => compareByCodeUnit(a, b));
    return Object.fromEntries(sorted.map(([k, v]) => [k, sortValue(v)]));
  }
  return value;
}

export function createCanonicalPayload(payload: unknown): string {
  return JSON.stringify(sortValue(payload));
}

export function createSignaturePayload(input: {
  fingerprint: string;
  nonce: string;
  timestamp: string;
  payload: unknown;
}): string {
  return `${input.fingerprint}.${input.nonce}.${input.timestamp}.${createCanonicalPayload(input.payload)}`;
}

export function createSignature(input: {
  fingerprint: string;
  nonce: string;
  timestamp: string;
  payload: unknown;
  secret: string;
}): string {
  return createHmac("sha256", input.secret)
    .update(createSignaturePayload(input))
    .digest("hex");
}
```

Python, as a worked port for languages without JavaScript's string comparison:

```python
import hashlib
import hmac
import json
from typing import Any


def _sort_key(key: str) -> bytes:
    """Sort key for JCS ordering: UTF-16 code units.

    UTF-16 **big-endian** is used deliberately. Comparing UTF-16BE bytes is
    equivalent to comparing UTF-16 code units numerically; comparing UTF-16LE
    bytes is not, because it compares the low byte first (see vector V7).
    """
    return key.encode("utf-16-be")


def _sort_value(value: Any) -> Any:
    if isinstance(value, list):
        return [_sort_value(item) for item in value]
    if isinstance(value, dict):
        return {
            k: _sort_value(v)
            for k, v in sorted(value.items(), key=lambda kv: _sort_key(kv[0]))
        }
    return value


def create_canonical_payload(payload: Any) -> str:
    # separators drop insignificant whitespace; ensure_ascii=False keeps
    # non-ASCII characters literal, matching JSON.stringify.
    return json.dumps(
        _sort_value(payload), separators=(",", ":"), ensure_ascii=False
    )


def create_signature_payload(
    fingerprint: str, nonce: str, timestamp: str, payload: Any
) -> str:
    canonical = create_canonical_payload(payload)
    return f"{fingerprint}.{nonce}.{timestamp}.{canonical}"


def create_signature(
    fingerprint: str, nonce: str, timestamp: str, payload: Any, secret: str
) -> str:
    message = create_signature_payload(fingerprint, nonce, timestamp, payload)
    return hmac.new(
        secret.encode("utf-8"), message.encode("utf-8"), hashlib.sha256
    ).hexdigest()
```

#### Conformance vectors

Every implementation MUST reproduce these exactly. They are generated from the
reference implementation and verified against the backend's verifier. Run them
as a unit test before sending a single live request — each one isolates a
failure mode that is otherwise diagnosed only as an opaque `auth` error.

Fixed inputs for all vectors:

```
apiSecret   = "nps_test_conformance_secret"
fingerprint = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"   (64 × "a")
nonce       = "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6"
timestamp   = "1718976000000"
```

**V1 — a representative payload.**

```json
{"amount":5000,"currency":"UGX","customer":{"name":"John Doe","phoneNumber":"+256700000000"},"description":"Test payment","reference":"ORDER-2026-001","metadata":{"orderId":"12345","items":"3"}}
```

- canonical: `{"amount":5000,"currency":"UGX","customer":{"name":"John Doe","phoneNumber":"+256700000000"},"description":"Test payment","metadata":{"items":"3","orderId":"12345"},"reference":"ORDER-2026-001"}`
- signature: `dc6e1717d7c37d7a3b334087d9882c07663edb2dfc8f2f06cd77c0d2d8a58686`

**V2 — key insertion order is irrelevant.** The same fields as V1, inserted in
reverse, MUST produce the identical canonical string and signature as V1.

```json
{"metadata":{"items":"3","orderId":"12345"},"reference":"ORDER-2026-001","description":"Test payment","customer":{"phoneNumber":"+256700000000","name":"John Doe"},"currency":"UGX","amount":5000}
```

- signature: `dc6e1717d7c37d7a3b334087d9882c07663edb2dfc8f2f06cd77c0d2d8a58686`

**V3 — arrays keep their order; objects inside arrays are still sorted.**

```json
{"items":[{"unitPrice":2000,"name":"Zeta","quantity":1},{"name":"Alpha","quantity":2,"unitPrice":500}],"tags":["b","a","c"],"amount":4500}
```

- canonical: `{"amount":4500,"items":[{"name":"Zeta","quantity":1,"unitPrice":2000},{"name":"Alpha","quantity":2,"unitPrice":500}],"tags":["b","a","c"]}`
- signature: `98478585cf5ce0193a9aa6a6e86ff7f5dfc025945d03547dd548b9356b797e4b`

Note that `tags` stays `["b","a","c"]` and the second item keeps its position,
while each item object's own keys are sorted.

**V4 — string escaping.** Catches `ensure_ascii`, escaped forward slashes, and
HTML escaping. Only `"`, `\`, and control characters are escaped; `/`, `<`,
`>`, `&`, and non-ASCII characters are emitted literally.

```json
{"note":"café / 50% <b>&\"quoted\"</b>","path":"a/b/c","backslash":"x\\y","newline":"line1\nline2\ttab"}
```

- canonical: `{"backslash":"x\\y","newline":"line1\nline2\ttab","note":"café / 50% <b>&\"quoted\"</b>","path":"a/b/c"}`
- signature: `80eb3c6e35b8b3dcc67a57e056634b6f68f2f84b9454bea3aa5e86647eb47649`

**V5 — ASCII key ordering.** Digits before uppercase before `_` before
lowercase, i.e. plain code-unit order, not dictionary or case-insensitive order.

```json
{"Z":1,"_x":2,"a":3,"A":4,"z":5,"0":6}
```

- canonical: `{"0":6,"A":4,"Z":1,"_x":2,"a":3,"z":5}`
- signature: `7b9da2fccf0140a7b721715b7b61f17ad3407bd40659b54d983c8e8379108adc`

**V6 — empty containers and zero.** An empty map serializes as `{}`, never as
`[]` (a real trap in PHP, where `[]` is both an empty list and an empty map).

```json
{"emptyObject":{},"emptyArray":[],"emptyString":"","zero":0}
```

- canonical: `{"emptyArray":[],"emptyObject":{},"emptyString":"","zero":0}`
- signature: `f1d8a628663cc9279c675b001e5142e10c6880c2713145f7ebb946c73af2e875`

**V7 — non-ASCII key ordering.** The decisive vector: it fails under
locale-sensitive collation, under UTF-16LE byte sorting, and under UTF-8
code-point sorting, and passes only under true UTF-16 code-unit order. Merchant
`metadata` keys are arbitrary merchant-supplied strings, so this is reachable in
production traffic, not a theoretical case.

```json
{"ÿ":1,"Ā":2,"a":3,"注文":4}
```

- canonical: `{"a":3,"ÿ":1,"Ā":2,"注文":4}`
- signature: `f43182515649622666b920ac1274d6be5ee395d7c295a4eab6e914a48b212a3a`

The expected order is `a` (U+0061) → `ÿ` (U+00FF) → `Ā` (U+0100) → `注` (U+6CE8).
A UTF-16LE byte sort yields `Ā, a, 注文, ÿ` and a different signature.

#### What the server checks

The signature is verified alongside three bounds an implementation must design
around. All are enforced server-side; none are negotiable from the client.

| Check | Value | Consequence |
|-------|-------|-------------|
| Timestamp freshness | `x-nylon-timestamp` within **±5 minutes** of server time | Outside the window → `auth` error. This is why retries are re-signed per attempt (D19): a frozen timestamp ages out mid-backoff. |
| Nonce replay | A given `(apiKey, nonce)` pair is accepted **once**, remembered for **10 minutes** | A repeated nonce is **rejected**, not replayed from cache. Never reuse a nonce, including on retry. |
| Rate limits | 120 requests/minute per API key. Further limits apply above that, and sustained authentication failures from one source are throttled. | Exceeding any limit → `rate_limit` error. Retry backoff must not amplify past the per-key budget, and a client MUST honour a `rate_limit` error by backing off rather than retrying immediately. |

A client whose clock drifts more than 5 minutes from real time cannot sign a
valid request at all. Sign with the system clock in UTC milliseconds; do not
derive the timestamp from a cached or monotonic-only source.

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
- On retry, the request **body is unchanged** (same payload, same `reference`), but each attempt is **signed fresh** — a new `nonce`, `timestamp`, and `signature` per try. Idempotency is carried by the constant `reference` (see [D18](./decision-records.md#d18-the-reference-is-the-only-transaction-identity-no-separate-idempotency-key-no-heuristic-duplicate-detection)), not by reusing the nonce. Re-signing keeps a post-backoff retry inside the server's timestamp-freshness window and prevents a retry from being rejected as a nonce replay (see [D19](./decision-records.md#d19-retries-are-signed-fresh-per-attempt-the-reference-not-the-nonce-carries-idempotency))

### Status Polling

A PaymentInstance tracks status transitions by repeatedly calling the one-shot status operation until the transaction reaches a terminal state (see [D14](./decision-records.md#d14-status-updates-are-delivered-by-client-polling)).

- **Single-flight.** Only one status request is in flight per instance. The next poll is scheduled only after the current one resolves, so requests never overlap.
- **Jittered interval.** Each interval is the configured poll interval plus a small random jitter, so a fleet of concurrent instances does not synchronise into a thundering herd against the status endpoint.
- **De-duplication.** A status update that matches the instance's current status emits no event. Only a transition (`prev !== next`) emits.
- **Terminal stop.** On any terminal state the instance fetches the full transaction record, emits the terminal event, and stops. Optional `maxPollAttempts` and `maxPollDurationMs` caps bound a non-terminating transaction when the merchant sets them.
- **Delayed flag.** Status responses may include `delayed: true` when a payment has been non-terminal for more than ~3 minutes. When `onDelayed` is `"return"`, the instance resolves with the still-pending transaction; when `"wait"` (default), polling continues.
- **Backoff.** For the first two minutes, each interval is the configured poll interval plus jitter. After that, the interval doubles every two minutes up to a 15s cap.
- **Late-update guard.** Once an instance has resolved (terminal, error, or timeout) it emits no further events; an in-flight poll that resolves after that point is ignored.

