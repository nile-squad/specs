# Nylon Pay SDK Spec

**Version:** 1.2.0

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

6. **Fail loudly on misconfiguration, gracefully on runtime errors.** A missing or malformed API key/secret throws at initialization. Async initiation failures (`collectPayment`/`makePayout` rejected before a transaction exists — invalid key, bad signature, scope/limit/provider reject) also throw, carrying a structured `category`. Sync and blocking operations return error results. The boundary is: there is no transaction to observe yet → throw; otherwise → result. See [D13](#d13-async-initiation-failures-throw).

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

- **Decision** — The SDK exposes optional `beforeCollect`, `afterCollect`, `beforePayout`, and `afterPayout` hooks registered once at initialization via `NylonPayConfig.hooks`. Each hook is a wrapper object `{ enabled?, fn, onError }` (see `SdkHook<Fn>`): `fn` is the handler, `onError` is **required**, and `enabled` (default true) toggles the hook off without removing its config.
- **Context** — Merchants need to intercept payment calls for logging, analytics, metadata enrichment, and audit trails without wrapping every SDK call in their own middleware. Before we had hooks, merchants had to wrap every `collectPayment` call manually — leading to duplicated logic and missed edge cases (e.g., the error path not being instrumented).
- **Alternatives considered** — (a) Middleware stack (array of functions applied in sequence). Rejected: unnecessary complexity when a single hook per event is sufficient; arrays imply ordering and composition semantics that create confusion. (b) Event emitter on the SDK instance. Rejected: event emitters are appropriate for repeated events, not single-shot lifecycle points; `afterCollect` fires exactly once per call, not on a recurring bus. (c) Bare hook functions whose errors propagate to the caller (the original v1.0.x design). Rejected: see Update below — a broken hook crashing the payment call, or being silently `catch`-swallowed, are both unacceptable in a payments SDK.
- **Rationale** — One hook per lifecycle point. `before*` hooks allow payload enrichment (adding metadata, normalizing fields). `after*` hooks allow observability (logging, analytics) regardless of outcome. Hooks run synchronously in the call chain — `before*` is awaited before transport, `after*` is awaited before returning to the caller. The SDK runs `fn` inside a safe boundary; a throw or rejection is routed to the hook's `onError` and never bubbles into the payment flow. `onError` itself is wrapped too, so a faulty handler cannot crash the SDK.
- **Tradeoffs** — Requiring `onError` is mild boilerplate, but it forces the merchant to make an explicit, type-checked decision about hook failure. This is the deliberate middle ground between the two bad options: errors no longer crash the payment call (the old "propagate to caller" behaviour), and they are never silently swallowed (a `catch {}` would hide a failed wallet-credit/fulfillment side-effect behind a "successful" payment).
- **Update (v1.0.8)** — Reshaped bare hook functions into `SdkHook<Fn>` wrappers with a required `onError` and internal error containment (the implementation runs `fn` inside its language's safe-call/try equivalent). Breaking change to the `hooks` config shape; bumped per the changelog. Supersedes the original "hook errors propagate to the caller" tradeoff.

### D11: Integration tests are spec'd separately from unit tests

- **Decision** — The spec defines a canonical integration test suite that every SDK implementation must run against a real sandbox backend. These tests are distinct from the unit test requirements (§Tests) which mock the transport layer.
- **Context** — Unit tests with mocked transport verify SDK internals (signing, validation, retry logic). Integration tests verify the full round-trip: real HTTP calls, real server validation, real idempotency behavior, real error responses. A bug in the transport layer, URL construction, or header serialization is invisible to mocked tests but caught immediately by integration tests.
- **Alternatives considered** — (a) Rely on unit tests only. Rejected: mocked transport cannot catch wire-format bugs, header mismatches, or server-side validation changes. (b) Let each SDK define its own integration tests. Rejected: without a canonical list, coverage drifts across languages — one SDK tests idempotency, another doesn't, and regressions go unnoticed until production.
- **Rationale** — A spec'd integration test suite ensures cross-language parity. Every SDK, regardless of language, proves the same behaviors against the same backend. The suite is small (focused on contract verification, not exhaustive coverage) and runs in CI against a sandbox environment.
- **Tradeoffs** — Integration tests are slower and require network access + valid sandbox credentials. They cannot run in offline CI environments. Mitigated by keeping the suite small (~20 tests) and providing a skip mechanism for environments without backend access.

### D12: Errors are categorized, not status-coded

- **Decision** — Every error carries a machine-readable `category` from a fixed taxonomy (`auth`, `validation`, `limit`, `rate_limit`, `account`, `provider`, `not_found`, `internal`, `network`, `timeout`). HTTP is binary — `200` for success, `400` for every failure. The SDK never branches on HTTP status codes, and never classifies errors by matching message text.
- **Context** — The backend framework returns `400` for all failures and discards the response `data` field on errors, so neither the HTTP status nor a structured error body can carry meaning. Earlier the SDK inferred error kind from message substrings, which silently misclassified errors — e.g. the auth message `"API key was not found"` collided with transaction-not-found handling, so an invalid key surfaced as a polling timeout.
- **Alternatives considered** — (a) Distinct HTTP status codes (401/403/422). Rejected: the framework cannot emit them cleanly, and SDKs would then depend on transport status. (b) A structured error object in `data`. Rejected: the framework drops `data` on failures. (c) Substring matching on the message. Rejected: fragile and the source of the original bug.
- **Rationale** — The backend tags each error with ` -- error-type: <category>` appended to the message (the only channel that survives), and the SDK parses the suffix into a structured `SdkError { category, message }`. Stable, explicit, language-agnostic.
- **Tradeoffs** — The category rides in the message string rather than a dedicated field. Acceptable given framework constraints; the suffix format is fixed and tested on both sides.

### D13: Server-side initiation failures emit error event

- **Decision** — `collectPayment` and `makePayout` return a `PaymentInstance` that emits an `"error"` event when the transaction fails to start on the server (invalid key, bad signature, scope/limit rejection, provider rejection, network error, timeout). The transaction never started, so there is nothing to poll. Only client-side validation errors (zero amount, empty required fields, invalid items, missing bank details) throw synchronously.
- **Context** — A PaymentInstance models a transaction that exists and is being polled. When initiation fails on the server, there is no transaction to poll. Previously these failures were thrown, which forced merchants to wrap every call in `try/catch` and made it easy to miss errors if the handler wasn't attached yet. The error event carries `category` and `retryable` so the merchant can branch programmatically.
- **Alternatives considered** — (a) Throw on all initiation failures. Rejected: buries server-side failures in exceptions; the merchant may never see them if not caught. (b) Return a `Result` from the async ops. Rejected: breaks the PaymentInstance contract for the common success path.
- **Rationale** — Server-side initiation failure is an operational error that should surface through the event channel with structured metadata (`category`, `retryable`). Client-side validation errors (programmer mistakes) throw immediately. This separates concerns: `try/catch` for bugs, event handlers for operational failures.
- **Tradeoffs** — The merchant must attach an `"error"` handler to catch server-side initiation failures. This is the correct trade: an unhandled event is visible in logs; an unhandled exception crashes the process.

### D14: Status updates are delivered by client polling

- **Decision** — A PaymentInstance tracks status transitions by polling the one-shot status operation at a configurable interval until the transaction reaches a terminal state. Each interval carries a small random jitter so many concurrent instances do not synchronise into a thundering herd against the status endpoint. There is one status transport, and it is polling.
- **Context** — Async payment operations need status-transition notifications without forcing merchants to write polling loops. The PaymentInstance owns that loop internally. The transport must work everywhere an SDK runs — including restricted networks and proxies that buffer or break long-lived connections.
- **Alternatives considered** — (a) Push status over a server-sent events (SSE) stream with polling as a fallback. Evaluated and removed: it required two code paths (stream plus the fallback poll), a streaming HTTP client able to set auth headers, and a bounded read buffer — all to optimise latency on a path the fallback already had to cover. The fallback being mandatory meant polling could never be removed, so the stream was pure added surface. (b) WebSockets. Rejected: heavier than needed for one-directional status push. (c) Server-to-merchant webhooks as a substitute. Rejected: webhooks are the merchant-server push channel, not the PaymentInstance's own update source.
- **Rationale** — One transport is simpler to reason about, test, and port across languages. Polling is robust on every network, needs no special streaming client, and reuses the same signed transport and the same status shape as a one-shot status check — so a polled update and a one-shot check are handled identically. Jitter keeps a fleet of instances from aligning their requests.
- **Tradeoffs** — Higher latency and more requests than a push stream. Accepted: status transitions are not latency-critical, the interval and caps are configurable, and jitter plus single-flight polling (only one request in flight per instance) bound the load.

### D15: Response verification is fail-closed

- **Decision** — Every authenticated success response MUST carry a valid `_responseSignature`, and the SDK MUST reject the response (return an `internal` error, expose no data) when the signature is **missing** or **invalid**. The SDK never returns response data it has not verified.
- **Context** — The backend signs every SDK success response (the server either signs the response or fails the action — there is no unsigned success path). An earlier SDK only verified *when a signature was present* and accepted the data when the field was absent. That is fail-open: a man-in-the-middle (or a malicious endpoint) could strip `_responseSignature` and the SDK would surface forged data — a forged `"successful"` status, a forged amount, a forged phone-verification result.
- **Alternatives considered** — (a) Verify-if-present (the old behaviour). Rejected: trivially bypassed by stripping the field. (b) Make signing optional per-endpoint. Rejected: every SDK action already signs; optionality only reintroduces the hole.
- **Rationale** — Signature verification only protects integrity if a missing signature is treated as a failure. Fail-closed is the only posture consistent with "the SDK never exposes unverified data." The comparison is constant-time and length-guarded, so malformed signatures are rejected without raising an error.
- **Tradeoffs** — An SDK pointed at a backend that does not sign responses will reject everything. Acceptable and intended: an unsigned response is indistinguishable from a tampered one. Introduced in v1.0.8.

### D16: Webhook verification is replay-protected

- **Decision** — `verifyWebhookSignature` rejects a correctly-signed webhook whose **signed timestamp** is outside a tolerance window (default 300s). Authenticity (HMAC over the raw body) and freshness (signed timestamp within the window) must both hold. The freshness anchor is the timestamp carried **inside the signed body**, read only after the HMAC verifies; an unsigned transport header is never trusted for freshness. A tolerance of `0` disables the check.
- **Context** — HMAC authenticates *who* sent the webhook, not *when*. Without an expiry, a captured `(body, signature)` pair stays valid forever, so an attacker who records one delivery can replay it indefinitely (re-trigger a fulfilment, re-credit an order). The backend already stamps every delivery — first attempt and every retry — with a current `timestamp` inside the signed body, so the signed content needed for a freshness check is already on the wire.
- **Alternatives considered** — (a) Leave it to merchants (idempotency on their side). Rejected: replay protection is a property of the verifier; relying on every consumer to dedupe is fail-open by default. (b) Sign `${timestamp}.${body}` and add a header, with a dual-sign migration. Rejected as unnecessary: the timestamp is *already* inside the signed body, so no wire/sender change is required — the fix is verifier-only. (c) Trust the `x-nylon-timestamp` header. Rejected: the header is not covered by the HMAC, so a replay attacker could pair an old body with a fresh header; only the signed-body timestamp is trustworthy.
- **Rationale** — Reusing the already-signed timestamp makes replay protection a pure verifier change with no coordinated rollout. Because each genuine delivery (including a retry hours later) is re-stamped and re-signed, a 5-minute window never rejects legitimate traffic; only a replayed capture, whose embedded timestamp is now stale, is rejected. Fail-closed on a missing timestamp keeps an unverifiable webhook from being treated as fresh.
- **Tradeoffs** — Clock skew between the backend and the consumer must stay within the tolerance; the window is configurable for slow consumers, and `0` disables it. A webhook body without a signed timestamp is rejected when the check is on — acceptable, since every Nylon Pay delivery carries one. Introduced in v1.0.8.

### D17: Canonical payload uses the JSON Canonicalization Scheme (JCS)

- **Decision** — The canonical payload that feeds every signature (requests and responses) is the RFC 8785 (JCS) serialization: object keys sorted by **Unicode code point** (UTF-16 code-unit order) recursively, arrays left in place, numbers in shortest round-tripping form (ECMAScript number-to-string), minimal JSON string escaping, no insignificant whitespace. A locale-sensitive key comparison is prohibited.
- **Context** — An earlier version sorted keys with a locale-sensitive comparison. That order depends on the runtime's locale and Unicode/ICU data, so two parties — a future non-JS SDK, or the same runtime under a different locale or ICU build — can canonicalize the same payload to different bytes and reject each other's otherwise-valid signatures. It is an availability bug (a valid request/response fails verification), not a forgery risk: an attacker still cannot produce a valid signature without the secret. It is latent while every party shares one collation, and surfaces on keys that order differently under locale vs code point — realistically merchant-supplied `metadata` with mixed-case or non-ASCII keys.
- **Alternatives considered** — (a) Keep locale sorting, require a fixed locale everywhere. Rejected: unenforceable across languages and deployments, and still breaks on Unicode-version differences. (b) Define an ad-hoc sort. Rejected: JCS is the published, language-neutral standard with reference implementations. (c) Dual-verify transition (accept both old and new canonical forms during migration). Considered but not taken: only divergent-key payloads differ between the two forms, and those are rare, so a clean lockstep switch was chosen over carrying a second code path.
- **Rationale** — Code-point order is defined by the string itself, not the environment, so it is identical on every runtime. JCS additionally pins number and string serialization, giving a non-JS SDK an unambiguous target instead of "match whatever JavaScript emits." In JavaScript, `JSON.stringify` already produces JCS-conformant numbers and string escaping, so the implementation is the code-point key sort plus that serializer.
- **Tradeoffs** — Changing the canonical form changes signed content, so the SDK and backend must move together (lockstep). During version skew, an old SDK still signing with the locale order and a new backend disagree **only** on divergent-key payloads; ordinary camelCase/ASCII traffic is byte-identical under both, so the practical break is limited to the payloads that were already at risk. Introduced in v1.0.8.

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

See [D16](#d16-webhook-verification-is-replay-protected).

## PaymentInstance Contract

Every async operation returns a PaymentInstance with this interface:

**Properties:**
- `reference` — the transaction reference (read-only)
- `status` — current transaction status, set from the initiation response (typically `"pending"`) and updated with each status change (read-only)

**Methods:**
- `on(event, handler)` — register a handler for an event. Returns the instance (chainable).
- `once(event, handler)` — register a handler that fires at most once. Returns the instance.
- `off(event, handler)` — remove a handler. Returns the instance.
- `wait()` — blocks until a terminal state. Resolves with the full transaction on success, or `null` on failure, cancellation, or timeout. Never rejects.

**Events:**
- `processing` — provider acknowledged, transaction in progress
- `success` — transaction completed successfully (terminal, polling stops)
- `failed` — transaction failed (terminal, polling stops)
- `cancelled` — transaction cancelled (terminal, polling stops)
- `error` — server-side initiation failure (auth, limit, provider, network, timeout) or lifecycle error (network failure, reference mismatch, polling timeout). Transaction never started. (terminal, polling stops)

```typescript
type EventData = {
  event: PaymentEvent;          // which event fired
  transaction?: Transaction;    // full transaction on status-change events
  error?: string;               // error message on the "error" event
  category?: SdkErrorCategory;  // machine-readable error category on "error" event
  retryable?: boolean;          // whether re-invoking may succeed, on "error" event
  timestamp: string;           // ISO 8601 timestamp of when the event was emitted
};
```

**Status updates:**
- Polls the one-shot status operation at a configurable interval (default 2s), with a small random jitter added to each interval so concurrent instances do not synchronise
- Only one poll is in flight at a time; the next poll is scheduled only after the current one resolves
- Stops on any terminal event
- Stops after max attempts (default 150) or max duration (default 5 minutes)
- Reference mismatch between initiation and a status update emits `error` and stops
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

// Every hook is wrapped: `fn` is the handler, `onError` (required) receives any
// throw/rejection from `fn`, and `enabled` (default true) toggles it off.
// The SDK runs `fn` inside a safe boundary so merchant code can never crash the
// payment flow — failures route to `onError` instead of bubbling.
type SdkHook<Fn> = {
  enabled?: boolean; // default true
  fn: Fn;
  onError: (error: unknown) => void | Promise<void>;
};

type SdkHooks = {
  beforeCollect?: SdkHook<
    (
      input: CollectPaymentInput,
    ) => CollectPaymentInput | void | Promise<CollectPaymentInput | void>
  >;
  afterCollect?: SdkHook<
    (
      result: Result<{ reference: string; status: string }, string>,
      input: CollectPaymentInput,
    ) => void | Promise<void>
  >;
  beforePayout?: SdkHook<
    (
      input: MakePayoutInput,
    ) => MakePayoutInput | void | Promise<MakePayoutInput | void>
  >;
  afterPayout?: SdkHook<
    (
      result: Result<{ reference: string; status: string }, string>,
      input: MakePayoutInput,
    ) => void | Promise<void>
  >;
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
  /** Custom fetch implementation. Defaults to `globalThis.fetch`. Essential for edge runtimes and testing. */
  fetch?: typeof globalThis.fetch;
  /** Force a new instance even if one already exists for this key+secret+url. Defaults to `false`. See D11. */
  force?: boolean;
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

All requests are `POST` to `{baseUrl}`.

The default `baseUrl` is `https://api.nylonpay.nilesquad.com/api/services` — a single URL that includes both the origin and the path. The SDK appends no further path segments; the body identifies the service and action.

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

**HTTP is binary.** The backend returns HTTP `200` for success (`status: true`) and HTTP `400` for every failure (`status: false`) — regardless of cause. The SDK MUST NOT branch on HTTP status codes to classify errors. Provider/transport-level failures the backend never produced (network errors, request timeouts, gateway 5xx) are surfaced by the SDK's transport with the `network`/`timeout`/`internal` categories.

**Error categories.** Every server failure carries a machine-readable category so merchants branch on a stable value instead of parsing prose. Because the framework discards the response `data` on failures and only the `message` survives, the category travels as a suffix on the message:

```
<human-readable message> -- error-type: <category>
```

The SDK splits the suffix off, exposing `category` and the clean `message` on the structured `SdkError` (see [Error Categories](#error-categories)). The human portion (including any server log id) is preserved as the message. A message without the suffix is treated as category `internal`.

### Request Signing

Every request is signed with HMAC-SHA256. The signing protocol:

```
canonicalPayload = JSON.stringify(payload, sortedKeys)
signatureInput = fingerprint + "." + nonce + "." + timestamp + "." + canonicalPayload
signature = HMAC-SHA256(apiSecret, signatureInput)
```

The `canonicalPayload` is the **JSON Canonicalization Scheme** (RFC 8785 / JCS)
serialization of the payload — see [D17](#d17-canonical-payload-uses-the-json-canonicalization-scheme-jcs):
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
- Business failures are HTTP `400` and therefore never retried — they are returned (or thrown, for async initiation) immediately with their category
- Exponential backoff: `2^attempt * 1000 + random(0-500)` ms
- Max retries: configurable (default 3)
- Per-request timeout: configurable (default 30s), enforced via AbortController equivalent
- On retry, the same idempotency key is reused — the SDK must not generate a new nonce/key per retry attempt

### Status Polling

A PaymentInstance tracks status transitions by repeatedly calling the one-shot status operation until the transaction reaches a terminal state (see [D14](#d14-status-updates-are-delivered-by-client-polling)).

- **Single-flight.** Only one status request is in flight per instance. The next poll is scheduled only after the current one resolves, so requests never overlap.
- **Jittered interval.** Each interval is the configured poll interval plus a small random jitter, so a fleet of concurrent instances does not synchronise into a thundering herd against the status endpoint.
- **De-duplication.** A status update that matches the instance's current status emits no event. Only a transition (`prev !== next`) emits.
- **Terminal stop.** On any terminal state the instance fetches the full transaction record, emits the terminal event, and stops. The max-attempt and max-duration caps bound a non-terminating transaction.
- **Late-update guard.** Once an instance has resolved (terminal, error, or timeout) it emits no further events; an in-flight poll that resolves after that point is ignored.

## Error Categories

Every SDK error is a structured `SdkError`:

```typescript
type SdkErrorCategory =
  | "auth"        // invalid/missing/revoked/expired key, bad signature, replay, scope
  | "validation"  // input the server rejected
  | "limit"       // account/KYC transaction limits exceeded
  | "rate_limit"  // too many requests
  | "account"     // merchant account missing or not active
  | "provider"    // payment provider/engine rejected the operation
  | "not_found"   // referenced transaction does not exist
  | "internal"    // unexpected server-side failure
  | "network"     // request never reached the server (DNS, TLS, connection)
  | "timeout";    // request exceeded the configured timeout

type SdkError = {
  category: SdkErrorCategory;
  message: string;
  retryable?: boolean;
};
```

- `auth`, `validation`, `limit`, `rate_limit`, `account`, `provider`, `not_found`, `internal` are server-tagged via the message suffix.
- `network`, `timeout` are produced by the SDK transport when the request never completed.
- Merchants branch on `category`, never on `message` text or HTTP status.

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
| `baseUrl` | `https://api.nylonpay.nilesquad.com/api/services` |
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
- Webhook signature verification (valid, invalid, tampered, stale/replayed)
- The canonical Security Test suite (S1–S14, see §Security Tests) — required, not optional

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
- Canonical payload with keys that order differently under locale vs code point (mixed-case, leading underscore, diacritic, CJK, non-BMP) — must sort by code point (JCS)
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

### Security Tests

Every SDK implementation **MUST** ship a dedicated security test suite covering the canonical cases below. These are the cross-language contract for the SDK's cryptographic surface; IDs (S1–S14) are traceable from this document to each SDK's test code. They run with mocked transport — no network required.

| ID  | Requirement |
|-----|-------------|
| S1  | The request signature is deterministic for identical inputs and changes if **any** of payload, secret, nonce, or timestamp changes. It is a 64-char hex digest. A signature cannot be reproduced without the secret. |
| S2  | The canonical payload is **object-key-order independent** (top-level and nested) but **array-order significant**. Two payloads differing only in key insertion order produce the same signature. |
| S3  | Nonces are cryptographically random, 32 hex chars (16 bytes) by default, and unique across many rapid generations. |
| S4  | The fingerprint is a stable 64-char hex (SHA-256) value within a process. |
| S5  | Response verification **accepts** a payload with a valid signature. |
| S6  | Response verification **rejects** a tampered payload, and rejects a signature produced with a different secret. |
| S7  | Response verification rejects malformed, empty, short, or non-hex signatures **without throwing** (length-guarded constant-time compare), and rejects a one-byte-flipped signature. |
| S8  | Webhook verification accepts a valid signature over the **raw body**, and rejects a tampered body, a wrong secret, and a malformed signature (without throwing). |
| S9  | All signature comparisons use a constant-time, length-guarded comparison primitive — never an ordinary string/`==` comparison on the digest. |
| S10 | The transport **rejects a success response whose signature is missing** (fail-closed, per D15). It must not return the data. |
| S11 | The transport **rejects a success response whose signature is invalid**, and accepts one whose signature is valid. |
| S12 | Config construction rejects an `apiKey` without the `npk_` prefix and an `apiSecret` without the `nps_` prefix. |
| S13 | The API secret never appears on the SDK's public/serialized surface, and the instance cache is secret-aware (rotating the secret yields a different instance — it is never reused under a stale secret). |
| S14 | Webhook verification is replay-protected: a correctly-signed but **stale** webhook (signed timestamp older than the tolerance window) is **rejected**, a fresh one is accepted, a valid signature carrying **no timestamp fails closed**, and swapping in a fresh timestamp while keeping the captured signature is rejected (the timestamp is signed, so it cannot be refreshed without the secret). |

### Integration Tests

Every SDK must ship an integration test suite that runs against a real sandbox backend (not mocked transport). The suite verifies end-to-end contract compliance: real HTTP calls, real server-side validation, real idempotency behavior, and real error responses.

**Environment:**

- Tests require valid sandbox credentials (`apiKey` with `npk_test_` prefix, matching `apiSecret`).
- Tests run in sandbox/test mode — no real money moves, sandbox provider returns `"pending"` immediately then transitions through `"processing"` to terminal during polling (2–8 s total).
- A `NYLONPAY_TEST_MODE` environment variable (or language equivalent) gates tests that require live-only behavior (e.g., revoked key detection). These tests are skipped when the variable is not set to `live`.
- Each test creates a fresh SDK instance with singleton bypass (`force: true` or equivalent) to prevent state leakage between test suites.

**Required coverage — every SDK must implement these tests:**

| # | Test | Operation | What it proves |
|---|------|-----------|---------------|
| I1 | Collect payment happy path | `collectPayment` | Returns a valid reference and pending status from the real server |
| I2 | Get transaction after collect | `getTransaction` | Server-side record matches the reference returned by collect |
| I3 | Idempotency on collect | `collectPayment` × 2 | Same `reference` returns the same transaction, not a duplicate |
| I4 | Payout happy path | `makePayout` | Returns a valid reference and pending status from the real server |
| I5 | Get transaction after payout | `getTransaction` | Server-side record matches the reference returned by payout |
| I6 | Idempotency on payout | `makePayout` × 2 | Same `reference` returns the same transaction, not a duplicate |
| I7 | Verify phone | `verifyPhone` | Returns a verified result with customer name from the real provider |
| I8 | Key validation — missing apiKey | `createNylonPay` | Throws/returns error before any network call |
| I9 | Key validation — bad apiKey prefix | `createNylonPay` | Throws/returns error for keys without `npk_` prefix |
| I10 | Key validation — missing apiSecret | `createNylonPay` | Throws/returns error before any network call |
| I11 | Key validation — bad apiSecret prefix | `createNylonPay` | Throws/returns error for secrets without `nps_` prefix |
| I12 | Singleton behavior | `createNylonPay` × 2 | Second call without `force` returns the same instance |
| I13 | Unknown reference | `getTransaction` | Returns error for a reference that doesn't exist |
| I14 | Sub-minimum amount | `collectPayment` | Server rejects amounts below 500 with a validation error |
| I15 | Revoked key (live-only) | `collectPayment` | Server rejects a revoked API key — `collectPayment` throws an error with category `auth` (HTTP 400, not 401). Skipped unless `NYLONPAY_TEST_MODE=live` |
| I16 | Unknown key → auth category | `getStatus` / `collectPayment` | A well-formed but unknown key yields category `auth` — `getStatus` returns an error result, `collectPayment` throws. Sandbox-testable (unlike I15) |
| I17 | Resolve returns full Transaction | `collectPaymentAndResolve` | Returns `id`, numeric `amount`, `metadata`, and (on failure) `failureReason` — never a partial stub |
| I18 | Metadata round-trip | `collectPayment` + `getTransaction` | Merchant-supplied `metadata` is returned unchanged |
| I19 | Polling reaches terminal | `collectPayment` + `wait()` | A polling instance resolves to a terminal state and never hangs |

**Test isolation rules:**

- Each test uses a unique `reference` (either auto-generated or explicitly unique per run) to prevent cross-test idempotency collisions.
- Tests do not depend on execution order. Each test is independently runnable.
- Tests do not assert on server-side timing (e.g., "response arrived within 2 seconds") — sandbox latency is non-deterministic.
- Tests do not assert on internal server state beyond what the SDK's public API returns.

**Cross-language parity:**

- Every SDK language runs the same canonical tests (I1–I19). Language-specific additions are permitted but must not replace or weaken the canonical set.
- Test names must reference the spec ID (e.g., `I3: idempotency on collect`) so coverage audits can trace from spec to implementation.

### Spec Compliance

No SDK adds operations, parameters, events, or behavior beyond what this spec defines. If a feature is needed across SDKs, this spec updates first — then all implementations follow. Language-specific conveniences (e.g., helper functions that compose existing operations) are permitted only if they do not introduce new server interactions or alter the documented contract.

## Invariants

1. Every SDK operation routes through the signed transport layer. No operation bypasses signing.
2. The PaymentInstance stops polling on any terminal event (`success`, `failed`, `cancelled`, `error`). It never polls indefinitely.
3. The canonical payload in signing is the JCS (RFC 8785) form: object keys sorted by Unicode code point (never a locale-sensitive comparison), arrays left in order, JCS number/string serialization. Two identical payloads produce identical canonical strings on every runtime and locale, regardless of field insertion order (D17).
4. Response signature verification uses constant-time comparison. Timing attacks on signature validation are not possible.
5. The factory validates configuration eagerly. An SDK instance with invalid credentials cannot be constructed.
6. Auto-generated idempotency keys are unique per invocation. The probability of collision is negligible.
7. The `_fingerprint` field is injected into every authenticated request body. No authenticated request is sent without it.
8. Webhook signature verification computes the HMAC over the raw payload bytes, not parsed JSON. Re-serialization must not alter the signed content. The freshness check (invariant 23) reads the timestamp from the body only after the HMAC has verified, so it operates on trusted content.
9. The PaymentInstance `reference` property is immutable after creation. It cannot be reassigned or mutated.
10. Handler errors in the pubsub system are caught and do not propagate to other handlers or the polling loop.
11. All public types, functions, and constants are documented. No undocumented public surface exists.
12. `before*` hooks run after input validation and before the transport call. They cannot bypass validation. `after*` hooks run after the transport call, regardless of success or failure. Both are awaited synchronously in the call chain. A hook with `enabled: false` is skipped entirely.
13. A `before*` hook whose `fn` returns `null` or `void` leaves the payload unchanged; only a non-null return value replaces the input. A hook `fn` that throws or rejects never bubbles into the payment flow — the error is routed to the hook's required `onError`, and the call proceeds (for `before*`, with the original unmutated payload). `onError` is itself contained, so a faulty handler cannot crash the SDK.
14. Integration tests run against a real sandbox backend, not mocked transport. Every SDK implementation covers the same canonical test set (I1–I19) with spec IDs traceable from this document to the test code.
15. Integration tests are isolated: each test uses a unique reference, does not depend on execution order, and does not assert on non-deterministic server timing.
16. Every error exposes a `category` from the fixed taxonomy. The SDK never classifies errors by HTTP status code or by matching message text.
17. `collectPayment` and `makePayout` return a PaymentInstance that emits an `"error"` event on server-side initiation failure (auth, limit, provider, network, timeout). Only client-side validation errors (zero amount, empty fields, invalid items) throw synchronously. A PaymentInstance may be constructed to carry an initiation error via the `initialError` mechanism.
18. The blocking resolve variants return the full `Transaction` shape — the same fields as `getTransaction` — including `failureReason` and `metadata`. They never return a partial stub.
19. Response verification is fail-closed (D15): an authenticated success response without a valid `_responseSignature` is rejected as an `internal` error and its data is never returned. The SDK never exposes unverified response data.
20. Every SDK ships the canonical Security Test suite (S1–S14) with spec IDs traceable from this document to the test code. All signature comparisons use a constant-time, length-guarded primitive.
21. A PaymentInstance emits at most one terminal event and fires no events after it resolves (terminal state, error, or timeout). An in-flight poll that resolves after the instance has resolved is ignored.
22. The PaymentInstance makes at most one status request in flight at a time, and adds a random jitter to each poll interval so concurrent instances do not synchronise their requests.
23. Webhook verification is replay-protected (D16): after the HMAC verifies, the timestamp inside the signed body must be within the tolerance window (default 300s) or verification fails. The freshness anchor is the signed timestamp, never an unsigned header, so it cannot be refreshed without the secret. A `0` tolerance disables the check.

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

### F5: Server-push status transport

- **What** — Status updates are delivered by client polling ([D14](#d14-status-updates-are-delivered-by-client-polling)). A server-push transport (e.g. server-sent events or a long-lived stream) could lower transition latency and request volume.
- **Why it's deferred** — A push transport was prototyped (SSE with a polling fallback) and removed: it required two code paths, a streaming HTTP client able to set auth headers, and a bounded read buffer, all while polling still had to exist as the universal fallback. The added surface did not justify the latency gain for a non-latency-critical path. Revisiting requires a transport that works across restrictive proxies without a mandatory polling fallback, or evidence that latency matters enough to carry both paths.
- **Impact of deferring** — Status latency is bounded by the poll interval (default 2s) rather than near-real-time. Acceptable: the interval and caps are configurable, and jitter plus single-flight polling bound the load.

### F6: Webhook signature replay protection — RESOLVED

Resolved in v1.0.8. See [D16](#d16-webhook-verification-is-replay-protected),
the [verifyWebhookSignature](#verifywebhooksignature) contract, and invariant 23.
The original concern assumed a coordinated dual-sign rollout, but the backend
already stamps a current `timestamp` inside the signed body on every delivery and
retry — so the fix was a verifier-only freshness window (default 300s) with no
wire or sender change. Consumers should still apply their own idempotency as
defence in depth, but a captured webhook no longer verifies forever.

### F7: Locale-independent canonical payload sort — RESOLVED

Resolved in v1.0.8. See [D17](#d17-canonical-payload-uses-the-json-canonicalization-scheme-jcs),
the [Request Signing](#request-signing) canonical-payload rules, and invariant 3.
Both the SDK and the backend now sort object keys by Unicode code point (RFC 8785
/ JCS) instead of a locale-sensitive comparison, changed in lockstep. A
cross-language parity test covers mixed-case, underscore, digit, diacritic, CJK,
and non-BMP keys plus number and unicode string values. The change was a clean
break (no dual-verify): only divergent-key payloads differ between the old and
new canonical forms, and ordinary ASCII/camelCase traffic is byte-identical under
both.
