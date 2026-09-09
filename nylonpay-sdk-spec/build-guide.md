# Build Guide: implement a Nylon Pay SDK

Part of the [Nylon Pay SDK Spec](./spec.md).

**Read this first.** The other documents in this spec tell you *what* everything
is; this one tells you *in what order to build it*. Work top to bottom. Each
step ends with a **Verify** that either lights up green or tells you the next
step must not start.

You are building a small library: roughly a signing module, a transport
function, a validation layer, a factory that holds them together, and a
polling loop for async payments. It is a lot of small moving parts wired to
one contract. Build the parts in isolation, prove each one, then wire them.

---

## The parts you will build, in order

| Step | Component | What it does | Contract lives in | You're done when |
|------|-----------|--------------|-------------------|------------------|
| 1 | **Signing core** | Canonical payload, HMAC-SHA256, fingerprint, nonce, constant-time compare | [Security](./security.md) | Conformance vectors V1–V7 reproduce byte-for-byte |
| 2 | **Types** | The shape of every input, output, and event | [Types and Events](./types.md) | All types compile and export |
| 3 | **Validation + normalization** | Client-side checks: amounts, phone, reference, invoice items | [Operations](./operations.md), [Types](./types.md#phone-number-normalization) | The spec's input edge cases all fail cleanly, before any network call |
| 4 | **Transport** | Envelope, headers, request signing, response verification, retries, size cap | [Transport Contract](./transport.md), [Security](./security.md#response-verification) | A mocked request round-trips and a tampered response is rejected |
| 5 | **Factory** | `createNylonPay`, config validation, per-key singleton | [Configuration](./configuration.md), [Invariants 5, 27](./invariants-and-prohibitions.md) | Invalid config throws; same credentials return the same instance |
| 6 | **Operations** | The eleven public functions, each calling its action (two of them share `sdk-list-transactions`) | [Operations](./operations.md), [transport action table](./transport.md#request-format) | Every operation's happy path passes (mocked transport) |
| 7 | **PaymentInstance** | Polling loop, events, `wait()` | [PaymentInstance Contract](./payment-instance.md) | Lifecycle tests pass: terminal stop, dedup, reference mismatch, delayed flag |
| 8 | **Webhook verification** | `verifyWebhookSignature` on raw body bytes | [Operations](./operations.md#verifywebhooksignature), [Webhook catalog](./types.md#webhook-event-catalog) | S8/S14/S18 pass; never throws |
| 9 | **Sandbox integration** | The whole SDK against a real sandbox backend | [Implementation Requirements](./implementation-requirements.md) | I1–I19 pass live |

The steps are ordered by dependency: each step needs only what came before it.
Signing goes first because it is the hardest thing to get right, it is
completely self-contained, and conformance vectors prove it in isolation. You
do not want to discover a signing bug two steps in, buried under other
failures. Steps 2 and 3 are cheap and make every later step easier to read.
The polling loop is last because it sits on top of everything else.

---

## Step 1: Signing core

Everything else trusts this module. It decides whether your requests are ever
accepted at all.

**Build:** the complete signing sequence, in pseudocode, is in
[Security, Signing flow](./security.md#signing-flow-pseudocode). Use it as the
checklist. The pieces:

- `createCanonicalPayload(payload)`, the JCS canonical form: object keys
  sorted by **UTF-16 code unit** recursively, arrays left in order, ECMAScript
  shortest-round-trip numbers, minimal string escaping, no whitespace.
  See [Canonical payload (JCS)](./security.md#canonical-payload-jcs). The three
  ways to get the sort wrong are documented there; read that section before
  writing the sort.
- `createSignature({ fingerprint, nonce, timestamp, payload, secret })`:
  HMAC-SHA256 over `fingerprint + "." + nonce + "." + timestamp + "." +
  canonicalPayload`, lowercase hex output. The key is the `apiSecret` as raw
  UTF-8 bytes, untouched.
- `createNonce()`: 32 hex chars from cryptographic random bytes.
- `createFingerprint()`: SHA-256 hex of the five OS metadata values below,
  each prefixed with its label and joined with `|`, cached per process:
  `type:` `platform:` `arch:` `release:` `hostname:`
  (see [reference composition](./security.md#fingerprint-reference-composition)).
- `constantTimeEqual(a, b)`: length-guarded, no early exit, case-sensitive.

**Verify:** reproduce the [conformance vectors V1–V7](./security.md#conformance-vectors)
exactly, both the canonical string and the hex signature, as unit tests.
Compare your signature strings character-for-character against the published
ones. V7 is the one that catches the subtle sorting bugs, so do not skip it.
Add S1–S4 from the [Security Test suite](./implementation-requirements.md#security-tests)
to the same file.

**When it's wrong:** every request fails with an `auth` error and no diagnostic.
That is why this step is gated. Get it right here and the later steps never
have to wonder.

---

## Step 2: Types

**Build:** transcribe the types from [Types and Events](./types.md) into your
language's type system with the **same field names and value constraints**.
Note the two rules that trip people up:

- Single types carry the status unions (`TransactionStatus`, `TransactionType`),
  the input shapes, the `Transaction` shape, and the webhook shapes.
  `WebhookTransactionSnapshot` intentionally omits `statusText` and uses
  `null` (never omission) for missing values. Type it exactly that way.
- A `Result<T, E>`, `SdkError`, and `SdkErrorCategory` are defined in
  [Error Categories](./errors.md). The hook signatures in Types reference
  `Result<T, E>`; use [that shape](./types.md#the-result-shape).

**Verify:** the file compiles and every type is exported. Untyped languages:
add docblocks, per [Typing](./implementation-requirements.md#typing).

---

## Step 3: Validation + normalization

These run **synchronously at the call site**, before any signing or network
round-trip (invariant 33, S20). The server re-checks everything; this layer is
for fast, friendly failure.

**Build:**

- `normalizePhone()` per [Phone Number Normalization](./types.md#phone-number-normalization):
  strip whitespace, strip leading `+`, and if the result starts with `0` and is
  10 digits, prepend `256`. This runs three layers deep (SDK, backend schema,
  provider), build the client-side one now.
- `validateCollect` / `validatePayout` / invoice checks per
  [Action Payloads](./transport.md#action-payloads) and each operation's input
  shape in [Operations](./operations.md): positive integer `amount` (reject
  floats, they produce a canonical string no other language can reproduce),
  integer `quantity`/`unitPrice` > 0, supported currency, required fields,
  at most 50 invoice items, at most 10 tags.
- Reference validation: whole-string UUID match. Do not rely on end-anchoring
  that differs across languages. Python's `$` accepts `"<uuid>\n"` where
  JavaScript does not (invariant 32, S-check). Any UUID version passes.
- `testOutcome`: only `"success"` or `"fail"`, checked synchronously; the live
  backend rejects it anyway.
- `getTransaction`: at least one of `id`/`reference` required.

**Verify:** the [input edge cases](./implementation-requirements.md#edge-case-testing)
list, plus S20, all fail cleanly with a `validation` error and never touch the
network. `normalizePhone` round-trips the accepted formats table.

---

## Step 4: Transport

**Build** a single `postAction(action, payload)` that every operation will call:

- POST to `{baseUrl}` with no added path. The body identifies the action.
  Default base URL is in [Configuration](./configuration.md#defaults).
- Envelope: `{ "intent": "execute", "service": "sdk", "action": "sdk-<name>",
  "payload": { ...input, "_fingerprint": <fingerprint> } }`.
- Headers: `content-type: application/json`, plus the four `x-nylon-*` headers
  per [Request headers](./security.md#request-headers).
- Wire serialization: **omit** absent optional fields; never send `null`
  (invariant 25). This changes the signed bytes.
- Read the response body with a **running byte count** and abort past the
  configurable cap (default 10 MB), not by trusting `Content-Length`
  (invariants 26, 31; S17).
- Verify the response: strip `_responseSignature`, recompute
  HMAC-SHA256(apiSecret, canonicalPayload of the rest, which still contains
  `_requestNonce`), and require it to equal the received signature using the
  constant-time compare. Accept only lowercase hex (reject uppercase, S16).
  Then require `_requestNonce` to equal the nonce you sent (S15). On any
  failure, return an `internal` error; **never return unverified data**
  (fail-closed, D15). This sequence, in pseudocode, is in
  [Security, Response verification](./security.md#response-verification).
- Failure responses carry the category as a message suffix; split it off.
  Do NOT classify errors by HTTP status code ([Error Categories](./errors.md)).
- Retry per [Retry Policy](./transport.md#retry-policy): statuses 408/429/500/
  502/503/504 and network/timeout errors, exponential backoff
  `2^attempt * 1000 + random(0–500)` ms, `maxRetries` default 3. **Re-sign each
  attempt with a fresh nonce and timestamp**. Reusing the nonce gets the retry
  rejected as a replay (D19).

**Verify:** mocked-transport tests from the [Security and network suites](./implementation-requirements.md#tests):
S5–S7 (accept valid / reject tampered / reject malformed without throwing),
S10/S11 (fail-closed on missing/invalid signature), S15 (nonce binding),
S16 (uppercase rejected), S17 (size cap during read), S21 (fingerprint in body
equals the signed fingerprint; you sign the inner payload only, never the
envelope). Then the network edge cases: non-JSON body, empty body, 5xx,
request timeout.

---

## Step 5: Factory

**Build** `createNylonPay(config)`:

- Validate config eagerly: `apiKey` starts with `npk_`, `apiSecret` starts with
  `nps_`; otherwise throw immediately ([Configuration](./configuration.md),
  invariant 5, S12).
- Apply the [defaults](./configuration.md#defaults).
- Return a plain object of the operations (built in Step 6).
- **Singleton:** same `apiKey + apiSecret + baseUrl` returns the same instance;
  `force: true` bypasses. Cache is secret-aware. Rotating the secret yields a
  new instance (S13). Make the cache thread-safe (invariant 27).
- Wire the optional `hooks` (`beforeCollect`/`afterCollect`/`beforePayout`/
  `afterPayout`). Run a `before*` hook after validation and re-run validation
  on its mutation (invariant 12); `null`/`void` return leaves input unchanged;
  a throwing hook routes to its `onError`, never into the payment flow
  (invariant 13).

**Verify:** S12/S13; then I8–I12 once you can talk to the sandbox.

---

## Step 6: Operations

**Build** the public functions, one per row. Each maps to an action in the
[transport action table](./transport.md#request-format) and calls `postAction`.
The synchronous ones return a `Result` (the `SdkError` shape from
[Error Categories](./errors.md)); the async initiates
(`collectPayment`, `makePayout`) return a PaymentInstance or, on server-side
initiation failure, an instance that emits `error` (invariant 17, D13, and the
[`error` event](./payment-instance.md#events)). That event carries
`category`/`retryable`. Client-side validation errors (from Step 3) still
throw. For **retries on network failure, reuse the same `reference`** so the
server replays instead of double-charging (D18).

Operations and their inputs. [Operations](./operations.md) is the full
reference; the two resolve variants take the same input as their base and are
`collectPayment` + `wait()` composed (plus the server's inline poll head-start):

`collectPayment`, `collectPaymentAndResolve`, `makePayout`,
`makePayoutAndResolve`, `getStatus`, `getTransaction`, `listTransactions`,
`getTransactionsByTag` (client-side composition of `listTransactions`), `verifyPhone`,
`createInvoice`, `verifyWebhookSignature` (Step 8).

**Verify:** each operation's happy path with mocked transport, and the
[reference replay semantics](./operations.md#reference-uniqueness-and-replay)
(own reference → `duplicate: true` replay; foreign reference → `duplicate`
error). I6 exercises this live.

---

## Step 7: PaymentInstance

**Build** the polling loop returned by the async operations
([payment-instance.md](./payment-instance.md), [Status Polling](./transport.md#status-polling)):

- Calls `getStatus` (`sdk-get-status`) with the reference. Single-flight, only
  one poll in flight, next scheduled after the current resolves (prohibition 5).
- **Jitter** every interval so concurrent instances do not synchronise;
  base interval default 2s, doubling every two minutes up to a 15s cap
  (invariant 22).
- De-duplicate: a status change to the same state emits nothing; only
  `prev !== next` emits.
- `processing` fires at most once, on the next tick after creation when the
  initiation response is non-terminal, so handlers registered after the
  operation returns still fire; a payment that goes straight to terminal must
  still fire `processing` first.
- `success`/`failed`/`cancelled` are terminal. Fetch the full transaction, emit
  the event, stop.
- `error` on network failure, reference mismatch, or merchant-configured
  timeout; stops polling.
- `delayed: true` with `onDelayed: "return"` resolves immediately with the
  still-pending transaction; `"wait"` (default) keeps polling.
- **Late-update guard:** once resolved, no further events and in-flight polls
  are ignored (invariant 21).
- `wait()` never rejects: resolves `Transaction` on success, `null` on
  failure/cancellation/timeout/polling error. Honors `maxPollDurationMs` /
  `maxPollAttempts` when set; by default no cap
  ([Configuration](./configuration.md#polling-caps)).

**Verify:** the [polling edge cases](./implementation-requirements.md#polling-and-paymentinstance)
list: pending→processing→successful, pending→failed, pending→cancelled,
first poll `not found`, reference mismatch, timeout caps, delayed flag both
modes, `wait()` after terminal, `off()` for an unregistered handler, network
error mid-poll. I19 proves it live.

---

## Step 8: Webhook verification

**Build** `verifyWebhookSignature` per
[Operations](./operations.md#verifywebhooksignature) and the
[Webhook catalog](./types.md#webhook-event-catalog). The full sequence, in
pseudocode, is in [Security, Webhook verification](./security.md#webhook-integrity):

- HMAC-SHA256 over the **raw body bytes**, before any JSON parse or
  re-serialization (invariants 8, 29), keyed with the **webhook secret**, not
  `apiSecret`. Lowercase hex, only canonical form accepted.
- After the HMAC verifies, read `timestamp` from inside the **signed body**
  (never a header) and require it within `toleranceSeconds` of now (default
  300). `0` = zero seconds (strict, not disabled, D20); opting out requires
  the `DISABLE_FRESHNESS_CHECK` sentinel.
- **Never throws** on any input: malformed signature, non-hex, invalid UTF-8,
  empty body, unparseable JSON, missing timestamp. All return `false`.

**Verify:** S8, S14, S18, and the [worked delivery](./types.md#worked-delivery)
example (a correct sender's bytes must verify; re-serialized bytes must not).

---

## Step 9: Sandbox integration

Run the full [integration suite I1–I19](./implementation-requirements.md#integration-tests)
against a real sandbox backend (`npk_test_` key). Each test uses a unique
`reference`; fresh instance via `force: true`; no assertions on server timing.
I17 (resolve returns the full transaction), I18 (metadata round-trip) and I19
(polling reaches terminal) are the three that catch assembly mistakes this
guide's mocked tests cannot.

**You're done when I1–I19 pass.** That is the definition of "the spec is
implemented": the Security suite (S1–S21, built in Steps 1–8) plus the
integration suite against a live sandbox.

---

## Pitfalls that cost implementors the most time

| Mistake | Symptom | Reference |
|---------|---------|-----------|
| Signing the full envelope instead of the inner payload | Every request = `auth` error, 100% | [What exactly is signed](./security.md#what-exactly-is-signed) |
| Key sort by locale / UTF-16LE bytes / code points | Only some payloads sign correctly | [Canonical payload (JCS)](./security.md#canonical-payload-jcs), vector V7 |
| Sending `null` for absent fields | Signature mismatch; `null` and absent are distinct signed values | [Wire Serialization Rules](./transport.md#wire-serialization-rules) |
| Float `amount` on the wire | Server cannot reproduce the canonical form | Invariant 33 |
| Reusing the nonce on retry | Retry rejected as a replay | [D19](./decision-records.md#d19-retries-are-signed-fresh-per-attempt-the-reference-not-the-nonce-carries-idempotency) |
| Reusing the `reference` for a NEW payment | Silent double-processing prevention: reuses the old transaction | [Reference replay](./operations.md#reference-uniqueness-and-replay) |
| `verifyWebhookSignature` with `apiSecret` | Every webhook fails | [verifyWebhookSignature](./operations.md#verifywebhooksignature) |
| Trusting `Content-Length` for the size cap | Memory exhaustion on chunked responses | Invariant 31 |
| Fresh `timestamp` generated once, reused across retries | Retry ages out of the server's freshness window and is rejected as stale | [What the server checks](./security.md#what-the-server-checks) |

---

## Where everything lives

| You need… | Read |
|-----------|------|
| Build order | This guide |
| The contract per operation | [Operations](./operations.md) |
| Wire bytes, envelope, retry | [Transport Contract](./transport.md) |
| Signing, verification, fingerprint, vectors | [Security](./security.md) |
| All shapes | [Types and Events](./types.md) |
| Factory config | [Configuration](./configuration.md) |
| Failure taxonomy | [Error Categories](./errors.md) |
| Hard guarantees | [Invariants and Prohibitions](./invariants-and-prohibitions.md) |
| Why the design is what it is | [Decision Records](./decision-records.md) |
| Test suites you must ship | [Implementation Requirements](./implementation-requirements.md) |