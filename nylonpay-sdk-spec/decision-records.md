# Decision Records

Part of the [Nylon Pay SDK Spec](./spec.md).

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

