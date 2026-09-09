# Security

Part of the [Nylon Pay SDK Spec](./spec.md).

The SDK's security surface covers request signing, canonical payload rules, the
client fingerprint, response verification, response size bounds, and webhook
integrity. The wire mechanics (endpoint, envelope, action payloads) live in the
[Transport Contract](./transport.md); errors and their categories live in
[Error Categories](./errors.md); the security test
suite an implementation must ship is S1–S21 in
[Implementation Requirements](./implementation-requirements.md).

Every signature this platform produces, request, response, and webhook alike, is
HMAC-SHA256 and travels in exactly one canonical form: **lowercase hex**
(invariant 28). Nothing else is accepted.

## Request Signing

Every request is signed with HMAC-SHA256. The signing protocol:

```
canonicalPayload = JCS(payload)
signatureInput   = fingerprint + "." + nonce + "." + timestamp + "." + canonicalPayload
signature        = HMAC-SHA256(key = apiSecret, message = signatureInput)   // lowercase hex
```

The HMAC key is the `apiSecret` string as raw UTF-8 bytes, it is not decoded,
hashed, or stripped of its `nps_` prefix first.

## What exactly is signed

`payload` is the **inner `payload` object of the request envelope:** the
operation's input plus `_fingerprint`, and **NOT** the full envelope. The
`intent`, `service`, and `action` fields are outside the signature. Signing the
whole envelope is the single most common first-implementation mistake; it
produces a well-formed request that fails auth 100% of the time.

For the example exchange in the [Transport Contract](./transport.md#example-exchange),
the signed object is:

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

## Canonical payload (JCS)

The `canonicalPayload` is the **JSON Canonicalization Scheme** (RFC 8785 / JCS)
serialization of the payload. See [D17](./decision-records.md#d17-canonical-payload-uses-the-json-canonicalization-scheme-jcs):

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
   (`utf-16-be`), big-endian byte order and code-unit order coincide.
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
| JavaScript / TypeScript | `JSON.stringify` is already canonical, no options needed. |
| Python | `json.dumps(value, separators=(",", ":"), ensure_ascii=False)`. The default separators insert spaces; the default `ensure_ascii=True` escapes non-ASCII to `\uXXXX`. |
| PHP | `json_encode($value, JSON_UNESCAPED_UNICODE \| JSON_UNESCAPED_SLASHES)`. Without these, `/` becomes `\/` and non-ASCII becomes `\uXXXX`. Empty maps must be `stdClass`/`JSON_FORCE_OBJECT`, not `[]`. |
| Go | Set `Encoder.SetEscapeHTML(false)`. `encoding/json` escapes `<`, `>`, and `&` by default, and re-sorts map keys by code point, build an ordered structure yourself rather than relying on map iteration. |
| Java / C# | Disable HTML/non-ASCII escaping and any pretty-printing; serialize from an explicitly ordered map. |

Vector V4 below catches every one of these escaping defaults.

## Request headers

- `x-nylon-key`: API key (plaintext, starts with `npk_`)
- `x-nylon-nonce`: 32-character hex nonce (unique per request, from cryptographic random bytes)
- `x-nylon-timestamp`: millisecond timestamp as a decimal string
- `x-nylon-signature`: computed HMAC signature, **lowercase hex** (the one canonical form; see invariant 28)

## Request body additions

- `_fingerprint`: SHA-256 hash of OS metadata, injected into every authenticated
  request body. Its exact composition is an implementation choice (the server
  treats it as an opaque stable identifier and never recomputes it, so
  implementations need not agree with one another); it MUST be a stable 64-char
  lowercase hex value for the life of the process, and MUST match the
  `fingerprint` used in `signatureInput`. Implementations SHOULD NOT include a
  language or runtime version, which is not obtainable the same way in every
  language and changes the value on every upgrade for no benefit (see D4).

## Fingerprint reference composition

The reference implementation hashes this exact string, and the Python and PHP
SDKs follow the same shape:

```
<type>:<value>|<platform>:<value>|<arch>:<value>|<release>:<value>|<hostname>:<value>
```

That is, the five OS inputs each prefixed `type:`/`platform:`/`arch:`/`release:`/`hostname:`
and joined with `|`, then SHA-256, hex. For example:

```typescript
// TypeScript reference
import { createHash } from "node:crypto";
import { arch, hostname, platform, release, type } from "node:os";

const components = [
  `type:${type()}`,
  `platform:${platform()}`,
  `arch:${arch()}`,
  `release:${release()}`,
  `hostname:${hostname()}`,
].join("|");

const fingerprint = createHash("sha256").update(components).digest("hex"); // 64 lowercase hex chars
```

The value is cached for the life of the process (the environment cannot change
within a running process). SDKs MUST NOT need to agree with each other on the
exact inputs or their formatting. The server treats `_fingerprint` as an opaque
stable identifier, so each implementation's exact composition is its own. What
every implementation must share is the *shape*: stable for the process lifetime,
64-char lowercase hex, and byte-identical between the `signatureInput` first
component and the `_fingerprint` value in the body (invariant 34).

The fingerprint is not a security boundary for replay: a captured request is
replayed with the fingerprint it was signed with, so replay is prevented by the
nonce and timestamp, not by this value.

## Signing flow (pseudocode)

The whole request-signing sequence, in one block. Every step below is a MUST
for any implementation; the concrete code follows in the next section.

```
# ONE authenticated request, end to end:
#
# 1. Fingerprint. WHAT GOES IN IT: SHA-256 hex of the five OS metadata values
#    below, each prefixed with its `type:`/`platform:`/`arch:`/`release:`/`hostname:`
#    label and joined with "|" (see "Fingerprint reference composition" above).
#    Computed once, cached for the life of the process.
fingerprint = sha256_hex(
    "type:"     + os.type()     + "|" +
    "platform:" + os.platform() + "|" +
    "arch:"     + os.arch()     + "|" +
    "release:"  + os.release()  + "|" +
    "hostname:" + os.hostname()
)                                  # -> 64 lowercase hex chars (invariant 34)

# 2. Nonce: 32 hex chars from cryptographically random bytes (S3).
nonce = random_hex(16 bytes)       # e.g. "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6"

# 3. Timestamp: milliseconds since epoch, system clock, decimal string (UTC).
timestamp = now_utc_ms()           # e.g. "1718976000000"

# 4. The signed object is the INNER payload only (what exactly is signed):
payload = operation_input + { "_fingerprint": fingerprint }

# 5. JCS canonical form of that payload (Canonical payload (JCS)):
canonical = jcs(payload)           # sorted keys, arrays in order, integers,
                                   # no whitespace, minimal escaping

# 6. Signature: HMAC-SHA256 over the four parts joined with ".".
#    Key = apiSecret as raw UTF-8 bytes, untouched (not decoded/hashed/stripped).
raw_secret = utf8_bytes(apiSecret)
message    = fingerprint + "." + nonce + "." + timestamp + "." + canonical
signature  = hmac_sha256_hex(raw_secret, message)   # lowercase hex (invariant 28)

# 7. Wire it: headers + body.
POST {baseUrl}
  headers:
    content-type:      "application/json"
    x-nylon-key:       apiKey          # plaintext, npk_ prefix
    x-nylon-nonce:     nonce
    x-nylon-timestamp: timestamp
    x-nylon-signature: signature       # lowercase hex
  body (envelope):
    { "intent": "execute",
      "service": "sdk",
      "action": "sdk-<operation>",
      "payload": payload }             # payload already carries _fingerprint

# 8. Retry (only statuses 408/429/500/502/503/504, network errors, timeouts):
#    keep the SAME body and reference, but redo steps 2, 3, 6 for each attempt
#    (fresh nonce + timestamp + signature) so the retry is not rejected as a
#    replay and does not age out of the freshness window (D19).
```

## Reference implementation

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

## Conformance vectors

Every implementation MUST reproduce these exactly. They are generated from the
reference implementation and verified against the backend's verifier. Run them
as a unit test before sending a single live request, each one isolates a
failure mode that is otherwise diagnosed only as an opaque `auth` error.

Fixed inputs for all vectors:

```
apiSecret   = "nps_test_conformance_secret"
fingerprint = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"   (64 × "a")
nonce       = "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6"
timestamp   = "1718976000000"
```

**V1: a representative payload.**

```json
{"amount":5000,"currency":"UGX","customer":{"name":"John Doe","phoneNumber":"+256700000000"},"description":"Test payment","reference":"ORDER-2026-001","metadata":{"orderId":"12345","items":"3"}}
```

- canonical: `{"amount":5000,"currency":"UGX","customer":{"name":"John Doe","phoneNumber":"+256700000000"},"description":"Test payment","metadata":{"items":"3","orderId":"12345"},"reference":"ORDER-2026-001"}`
- signature: `dc6e1717d7c37d7a3b334087d9882c07663edb2dfc8f2f06cd77c0d2d8a58686`

**V2: key insertion order is irrelevant.** The same fields as V1, inserted in
reverse, MUST produce the identical canonical string and signature as V1.

```json
{"metadata":{"items":"3","orderId":"12345"},"reference":"ORDER-2026-001","description":"Test payment","customer":{"phoneNumber":"+256700000000","name":"John Doe"},"currency":"UGX","amount":5000}
```

- signature: `dc6e1717d7c37d7a3b334087d9882c07663edb2dfc8f2f06cd77c0d2d8a58686`

**V3: arrays keep their order; objects inside arrays are still sorted.**

```json
{"items":[{"unitPrice":2000,"name":"Zeta","quantity":1},{"name":"Alpha","quantity":2,"unitPrice":500}],"tags":["b","a","c"],"amount":4500}
```

- canonical: `{"amount":4500,"items":[{"name":"Zeta","quantity":1,"unitPrice":2000},{"name":"Alpha","quantity":2,"unitPrice":500}],"tags":["b","a","c"]}`
- signature: `98478585cf5ce0193a9aa6a6e86ff7f5dfc025945d03547dd548b9356b797e4b`

`tags` stays `["b","a","c"]` and the second item keeps its position,
while each item object's own keys are sorted.

**V4: string escaping.** Catches `ensure_ascii`, escaped forward slashes, and
HTML escaping. Only `"`, `\`, and control characters are escaped; `/`, `<`,
`>`, `&`, and non-ASCII characters are emitted literally.

```json
{"note":"café / 50% <b>&\"quoted\"</b>","path":"a/b/c","backslash":"x\\y","newline":"line1\nline2\ttab"}
```

- canonical: `{"backslash":"x\\y","newline":"line1\nline2\ttab","note":"café / 50% <b>&\"quoted\"</b>","path":"a/b/c"}`
- signature: `80eb3c6e35b8b3dcc67a57e056634b6f68f2f84b9454bea3aa5e86647eb47649`

**V5: ASCII key ordering.** Digits before uppercase before `_` before
lowercase, i.e. plain code-unit order, not dictionary or case-insensitive order.

```json
{"Z":1,"_x":2,"a":3,"A":4,"z":5,"0":6}
```

- canonical: `{"0":6,"A":4,"Z":1,"_x":2,"a":3,"z":5}`
- signature: `7b9da2fccf0140a7b721715b7b61f17ad3407bd40659b54d983c8e8379108adc`

**V6: empty containers and zero.** An empty map serializes as `{}`, never as
`[]` (a real trap in PHP, where `[]` is both an empty list and an empty map).

```json
{"emptyObject":{},"emptyArray":[],"emptyString":"","zero":0}
```

- canonical: `{"emptyArray":[],"emptyObject":{},"emptyString":"","zero":0}`
- signature: `f1d8a628663cc9279c675b001e5142e10c6880c2713145f7ebb946c73af2e875`

**V7: non-ASCII key ordering.** The decisive vector: it fails under
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

## What the server checks

The signature is verified alongside three bounds an implementation must design
around. All are enforced server-side; none are negotiable from the client. The
specific thresholds are operational settings and may change; design for the
behavior, not the numbers.

| Check | Behavior | Consequence |
|-------|----------|-------------|
| Timestamp freshness | `x-nylon-timestamp` must be within a bounded window of server time | Outside the window → `auth` error. This is why retries are re-signed per attempt (D19): a frozen timestamp ages out mid-backoff. |
| Nonce replay | A given `(apiKey, nonce)` pair is accepted **once** and remembered for a bounded window | A repeated nonce is **rejected**, not replayed from cache. Never reuse a nonce, including on retry. |
| Rate limits | Requests per API key are rate-limited; sustained authentication failures from one source are additionally throttled | Exceeding any limit → `rate_limit` error. Retry backoff must not amplify past the per-key budget, and a client MUST honour a `rate_limit` error by backing off rather than retrying immediately. |

A client whose clock drifts beyond the freshness window cannot sign a valid
request at all. Sign with the system clock in UTC milliseconds; do not derive
the timestamp from a cached or monotonic-only source.

## Response Verification

The server signs every response to prevent tampering, and binds it to the
request that solicited it (D21):

- Response body includes a `_responseSignature` field
- Response body includes a `_requestNonce` field, the nonce from the request being answered, covered by the signature
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

Response verification, in one block. **Fail-closed: on any failure, return an
`internal` error and expose no data** (D15):

```
# verifyResponse(body, nonceWeSent):
# 1. Size cap FIRST, during the read, never via Content-Length alone (S17).
body = read_response_body(max_bytes = 10MB)   # abort the read if exceeded

# 2. Signature must be present and canonical (lowercase 64-char hex).
signature = body["_responseSignature"]
if signature is missing or not lowercase_hex_64:   return internal_error

# 3. Recompute over the REST of the body (strip _responseSignature only;
#    _requestNonce stays inside and is covered by the signature).
rest                  = body minus "_responseSignature"
canonical             = jcs(rest)
expected              = hmac_sha256_hex(utf8_bytes(apiSecret), canonical)
if not constant_time_equal(expected, signature):   return internal_error   # S6/S11

# 4. Bind to the request that asked: the echoed nonce must match ours.
if rest["_requestNonce"] != nonceWeSent:          return internal_error   # S15

# 5. Only now return the data, with _responseSignature and _requestNonce
#    stripped. Never return unverified data; never classify by HTTP status.
return rest minus "_requestNonce"
```

## Response Size Bounds

The SDK MUST enforce a maximum response body size. Responses whose body exceeds
the configured limit MUST be rejected as an `internal` error before signature
verification begins, the data is never read into memory beyond the limit. The
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

## Webhook Integrity

Webhooks are the server-push complement to the signed request/response surface.
The merchant verifies each delivery with `verifyWebhookSignature`
([Operations](./operations.md#verifywebhooksignature)); the catalog of events
and the exact delivery shape live in [Types and Events](./types.md#webhook-event-catalog).

- **Signature over the raw body.** The `x-nylon-signature` header carries
  HMAC-SHA256 over the **raw request body bytes**, keyed with the merchant's
  **webhook secret** (a separate credential from `apiSecret`), lowercase hex.
  The signature does NOT live in the body. Verify the exact bytes received,
  before any JSON parse or re-serialization (invariants 8 and 29).
- **Canonical form.** The signature must arrive in lowercase hex; verification
  rejects any other spelling (invariant 28).
- **Freshness and replay protection.** The authenticity check proves *who* sent
  the webhook, not *when*. Every delivery (including retries) stamps a current
  ISO 8601 UTC `timestamp` **inside the signed body** (never trusted from a
  header). After the HMAC verifies, the verifier rejects the delivery if that
  signed timestamp is outside the tolerance window (default 300s). `0` means
  zero seconds (strict, not disabled); opting out requires the
  `DISABLE_FRESHNESS_CHECK` sentinel. A captured `(body, signature)` pair goes
  stale, so a replay fails closed (invariant 23, D16).
- **Delivery headers.** `x-nylon-event`, `x-nylon-delivery-id`, and
  `x-nylon-timestamp` are conveniences for routing and logging; they are NOT
  covered by the signature and can be set to anything by a replay attacker. The
  freshness timestamp is always read from the signed body, never from these
  headers.
- **At-least-once delivery.** Duplicate deliveries are possible; merchants must
  be idempotent on `delivery_id` or `reference`. A merchant must respond `2xx`
  within 10 seconds.

Webhook verification, in one block. **Never throws on any input; always
returns `false` on failure** (S8, S14):

```
# verifyWebhookSignature(bodyBytes, xNylonSignatureHeader, toleranceSeconds):
# 1. Must arrive in lowercase hex (the one canonical form, invariant 28).
if xNylonSignatureHeader is not lowercase_hex:        return false

# 2. HMAC over the RAW body bytes, exactly as received. No JSON parse, no
#    re-serialization (invariants 8 and 29). Keyed with the WEBHOOK secret,
#    never apiSecret.
expected = hmac_sha256_hex(utf8_bytes(webhookSecret), bodyBytes)
if not constant_time_equal(expected, xNylonSignatureHeader):  return false

# 3. Freshness: read `timestamp` from INSIDE the now-verified body, never from
#    an unsigned header (they are not covered by the signature).
parsed       = json_parse(bodyBytes)          # safe: authenticity already proven
signedStamp  = parsed["timestamp"]
if signedStamp is missing:                    return false          # fail-closed
if toleranceSeconds is not DISABLE_FRESHNESS_CHECK:
    age = now() - parse(signedStamp)
    if age > toleranceSeconds:                return false          # stale (D16/D20)
    # tolerance = 0 means ZERO seconds, not disabled; only the sentinel opts out.

return true
```

## Secret handling

- API secrets exist only in memory for the lifetime of the SDK instance. They
  are never stored on disk, in logs, or in error messages, and never sent in
  request bodies or query parameters (prohibitions 1–2).
- The webhook secret is a separate credential from `apiSecret`. Requests and
  responses are signed with `apiSecret`; webhooks are signed with the webhook
  secret configured on the API key. Using `apiSecret` for webhook verification
  makes every webhook fail.
- All signature comparisons, request, response, and webhook alike, use a
  constant-time, length-guarded primitive (invariant 4, S9).