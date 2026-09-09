# Decision Records

Part of the [Nylon Pay SDK Spec](./spec.md).

## D1: PubSub is a first-class primitive

- **Decision:** Every SDK implementation includes a pubsub system for PaymentInstance events. Languages without native event emitters implement a minimal equivalent.
- **Context:** Async payment operations need status transition notifications. Polling in a loop is error-prone and wasteful.
- **Alternatives considered:** (a) Callbacks only (single handler per event). Rejected: multiple consumers need to react to the same event. (b) Channels/streams only (Go/Rust native). Rejected: inconsistent with other languages, harder for merchants switching between SDKs.
- **Rationale:** `on`/`once`/`off` is the universal event pattern. Every target language can implement it. Go uses goroutine-safe handler registration. Rust uses trait-based handlers. Elixir uses message passing. The API surface is identical.
- **Tradeoffs:** Some languages carry synchronization overhead for thread-safe pubsub. Acceptable, payment operations are I/O-bound, not CPU-bound.

## D2: Payouts use the same async primitive as collections

- **Decision:** `makePayout` returns a PaymentInstance with the same event-driven interface as `collectPayment`.
- **Context:** Collections and payouts are both async operations that resolve over time. Merchants need the same DX for both.
- **Alternatives considered:** (a) Return a sync result for payouts and require manual `getStatus` calls. Rejected: inconsistent DX, forces polling logic onto every merchant. (b) A different async primitive for payouts. Rejected: two async patterns is worse than one.
- **Rationale:** One async primitive for all long-running operations. The PaymentInstance polls internally using the returned reference. Merchants learn one pattern and apply it everywhere.
- **Tradeoffs:** Payout confirmations may be slower than collections depending on the provider. Webhooks are the authoritative source for payout completion.

## D3: Webhook event types are defined in the SDK spec

- **Decision:** The SDK spec defines the webhook event catalog and verification utility.
- **Context:** Merchants need to know what events to expect and how to verify them. The verification function is an SDK export, and event types are part of the SDK's type surface.
- **Rationale:** Webhooks are the server-push complement to the SDK's client-pull model. They belong together so merchants have one document for the full integration surface.
- **Tradeoffs:** The spec is slightly larger. Offset by keeping all merchant-facing contracts in one place.

## D4: Fingerprint uses OS-level metadata

- **Decision:** The SDK generates a fingerprint from stable OS metadata, computed once per process and cached. The value is the SHA-256 of `type|platform|arch|release|hostname` joined with `|`, in lowercase hex, and MUST remain stable for the lifetime of the process. The reference composition for a worked example with exact bytes is in [Security](./security.md#fingerprint-reference-composition).
- **Context:** The signing protocol includes a fingerprint in the signature payload. The server treats the value as opaque: it reads `_fingerprint` from the request body, feeds that value into its own HMAC, and never refers to it again. It is not stored, and it does not bind a signature to a machine, since a replayed request carries the fingerprint it was signed with; replay is prevented by the nonce and timestamp.
- **Alternatives considered:** (a) No fingerprint. Rejected: the signing payload needs the stable per-process identity component. (b) Random per-process ID. Rejected: loses the OS-metadata identity value. (c) Runtime language/version inputs. Rejected: not obtainable the same way in every language, and they churn the value on every upgrade for no benefit.
- **Rationale:** OS metadata is stable per-process, unique per-server, and available in all target languages without external dependencies. Because the server never recomputes the fingerprint, composition is an implementation choice. SDKs need not agree with one another, and changing it does not invalidate signatures from clients built before the change.
- **Tradeoffs:** Hostname may change in containerized environments. Acceptable: the fingerprint is one component of the signed payload, and replay protection rests on the nonce and timestamp, not the fingerprint.

## D5: POST-only action-based transport

- **Decision:** All SDK requests are POST to `{baseUrl}/{action}` with JSON body.
- **Context:** The backend uses action-based routing. The SDK mirrors this architecture directly.
- **Rationale:** Action-based transport eliminates a mapping layer between SDK and server. The SDK abstracts URLs entirely, merchants call functions, not endpoints.
- **Tradeoffs:** Merchants familiar with REST may find action-based URLs unfamiliar. Offset by the SDK hiding URLs behind function calls.

## D6: Classes permitted only when functional is not idiomatic

- **Decision:** SDK implementations prefer functional patterns. Classes are allowed only when the language lacks idiomatic functional alternatives, and the end-user API must remain identical to functional implementations.
- **Context:** Some target languages (C#, Kotlin, PHP) have class-centric ecosystems where pure functional patterns are awkward or non-idiomatic.
- **Alternatives considered:** (a) Ban classes entirely. Rejected: forces unnatural patterns in some languages. (b) Allow classes freely. Rejected: leads to inconsistent DX across languages.
- **Rationale:** The merchant's experience is the invariant. Whether the SDK internally uses closures, structs, or classes is an implementation detail. What matters is that `nylon.collectPayment({ ... })` looks and behaves the same everywhere.
- **Tradeoffs:** Reviewers must verify that class-based implementations expose the same API surface as functional ones.

## D7: Card payments via hosted page only

- **Decision:** Card payments are only available through `createInvoice` (hosted payment page). The SDK does not accept card details directly.
- **Context:** Collecting card data (card number, CVV, expiry) imposes PCI-DSS compliance obligations on merchants. Most merchants are not equipped to handle raw card data.
- **Alternatives considered:** Exposing card fields in `collectPayment` with tokenization. Rejected: shifts PCI scope to the merchant's server, increases integration complexity, and creates a security surface that most merchants are not equipped to manage.
- **Rationale:** The hosted payment page handles card collection, 3DS, and PCI compliance on behalf of the merchant. Mobile money and bank transfers, which don't carry PCI obligations, are supported directly in the SDK.
- **Tradeoffs:** Merchants who want a fully embedded card checkout experience must use the hosted page redirect. This is an acceptable constraint given the security implications.

## D8: Provider routing is server-side, not merchant-facing

- **Decision:** The SDK does not expose provider selection, provider names, or provider references. The backend routes transactions to the appropriate provider based on merchant configuration, routing rules, and gateway health.
- **Context:** Nylon Pay operates as a payment gateway that abstracts provider complexity from merchants. Merchants specify what they want to do (collect, pay out), not how or through whom.
- **Alternatives considered:** (a) Allow merchants to specify a preferred provider. Rejected: couples merchants to provider-specific integrations, defeats the gateway abstraction. (b) Return provider name in transaction responses. Rejected: leaks internal routing, creates support burden when providers change.
- **Rationale:** Provider routing is an internal concern. The transaction response contains only what the merchant needs: status, amount, reference, failure reason. Provider details are sanitized out.
- **Tradeoffs:** Merchants cannot debug provider-specific issues directly. This is by design, provider issues are Nylon Pay's responsibility, not the merchant's.

## D9: Resolve variants for async operations

- **Decision:** `collectPaymentAndResolve` and `makePayoutAndResolve` are provided alongside their async counterparts. They block until the transaction reaches a terminal state and return the transaction directly.
- **Context:** Some integration patterns (CLI tools, serverless functions, simple scripts) don't benefit from event-driven polling. They need a synchronous "send and wait" operation.
- **Alternatives considered:** (a) Only provide async + `wait()`. Rejected: forces every synchronous caller to chain two calls. (b) Make all operations synchronous by default. Rejected: loses the event-driven pattern that most server integrations need.
- **Rationale:** Two patterns for two use cases. Event-driven for long-lived server processes. Blocking resolve for simple integrations. Both use the same underlying transport and polling, the resolve variant is a convenience wrapper.
- **Tradeoffs:** Slightly larger API surface. Offset by the clarity of having one function per use case instead of requiring composition.

## D10: Lifecycle hooks for cross-cutting concerns

- **Decision:** The SDK exposes optional `beforeCollect`, `afterCollect`, `beforePayout`, and `afterPayout` hooks registered once at initialization via `NylonPayConfig.hooks`. Each hook is a wrapper object `{ enabled?, fn, onError }` (see `SdkHook<Fn>`). `fn` is the handler, `onError` is **required**, and `enabled` (default true) toggles the hook off without removing its config.
- **Context:** Merchants need to intercept payment calls for logging, analytics, metadata enrichment, and audit trails without wrapping every SDK call in their own middleware. A bare handler leaves hook failure undefined: a broken hook could crash the payment call or be silently `catch`-swallowed, both unacceptable in a payments SDK.
- **Alternatives considered:** (a) Middleware stack (array of functions applied in sequence). Rejected: unnecessary complexity when a single hook per event is sufficient; arrays imply ordering and composition semantics that create confusion. (b) Event emitter on the SDK instance. Rejected: event emitters are appropriate for repeated events, not single-shot lifecycle points. `afterCollect` fires exactly once per call, not on a recurring bus. (c) Bare hook functions whose errors propagate to the caller. Rejected: a broken hook crashing the payment call, or being silently `catch`-swallowed, are both unacceptable in a payments SDK.
- **Rationale:** One hook per lifecycle point. `before*` hooks allow payload enrichment (adding metadata, normalizing fields). `after*` hooks allow observability (logging, analytics) regardless of outcome. Hooks run synchronously in the call chain, `before*` is awaited before transport, `after*` is awaited before returning to the caller. The SDK runs `fn` inside a safe boundary; a throw or rejection is routed to the hook's `onError` and never bubbles into the payment flow. `onError` itself is wrapped too, so a faulty handler cannot crash the SDK.
- **Tradeoffs:** Requiring `onError` is mild boilerplate, but it forces the merchant to make an explicit, type-checked decision about hook failure. This is the deliberate middle ground between the two bad options: errors never crash the payment call, and they are never silently swallowed. A `catch {}` would hide a failed wallet-credit/fulfillment side-effect behind a "successful" payment.

## D11: Integration tests are spec'd separately from unit tests

- **Decision:** The spec defines a canonical integration test suite that every SDK implementation must run against a real sandbox backend. These tests are distinct from the unit test requirements (§Tests) which mock the transport layer.
- **Context:** Unit tests with mocked transport verify SDK internals (signing, validation, retry logic). Integration tests verify the full round-trip: real HTTP calls, real server validation, real idempotency behavior, real error responses. A bug in the transport layer, URL construction, or header serialization is invisible to mocked tests but caught immediately by integration tests.
- **Alternatives considered:** (a) Rely on unit tests only. Rejected: mocked transport cannot catch wire-format bugs, header mismatches, or server-side validation changes. (b) Let each SDK define its own integration tests. Rejected: without a canonical list, coverage drifts across languages, one SDK tests idempotency, another doesn't, and regressions go unnoticed until production.
- **Rationale:** A spec'd integration test suite ensures cross-language parity. Every SDK, regardless of language, proves the same behaviors against the same backend. The suite is small (focused on contract verification, not exhaustive coverage) and runs in CI against a sandbox environment.
- **Tradeoffs:** Integration tests are slower and require network access + valid sandbox credentials. They cannot run in offline CI environments. Mitigated by keeping the suite small (~20 tests) and providing a skip mechanism for environments without backend access.

## D12: Errors are categorized, not status-coded

- **Decision:** Every error carries a machine-readable `category` from a fixed taxonomy:
  `auth`, `validation`, `limit`, `rate_limit`, `account`, `provider`, `not_found`,
  `internal`, `network`, `timeout`. HTTP is binary: `200` for success, `400` for
  every failure. The SDK never branches on HTTP status codes and never classifies
  errors by matching message text.
- **Context:** The backend framework returns `400` for all failures and discards the response `data` field on errors, so neither the HTTP status nor a structured error body can carry meaning.
- **Alternatives considered:** (a) Distinct HTTP status codes (401/403/422). Rejected: the framework cannot emit them cleanly, and SDKs would then depend on transport status. (b) A structured error object in `data`. Rejected: the framework drops `data` on failures. (c) Substring matching on the message. Rejected: fragile and silently misclassifies errors; e.g. the auth message `"API key was not found"` collided with transaction-not-found handling, so an invalid key surfaced as a polling timeout.
- **Rationale:** The backend tags each error with ` -- error-type: <category>` appended to the message, the only channel that survives. The SDK parses the suffix into a structured `SdkError { category, message }`. Stable, explicit, language-agnostic.
- **Tradeoffs:** The category rides in the message string rather than a dedicated field. Acceptable given framework constraints; the suffix format is fixed and tested on both sides.

## D13: Server-side initiation failures emit error event

- **Decision:** `collectPayment` and `makePayout` return a `PaymentInstance` that emits an `"error"` event when the transaction fails to start on the server. That includes an invalid key, a bad signature, scope/limit rejection, provider rejection, network error, and timeout. The transaction never started, so there is nothing to poll. Only client-side validation errors (zero amount, empty required fields, invalid items, missing bank details) throw synchronously.
- **Context:** A PaymentInstance models a transaction that exists and is being polled. When initiation fails on the server, there is no transaction to poll. The error event carries `category` and `retryable` so the merchant can branch programmatically.
- **Alternatives considered:** (a) Throw on all initiation failures. Rejected: buries server-side failures in exceptions, forces merchants to wrap every call in `try/catch`, and makes errors easy to miss if the handler isn't attached yet. (b) Return a `Result` from the async ops. Rejected: breaks the PaymentInstance contract for the common success path.
- **Rationale:** Server-side initiation failure is an operational error that surfaces through the event channel with structured metadata (`category`, `retryable`). Client-side validation errors (programmer mistakes) throw immediately. This separates concerns: `try/catch` for bugs, event handlers for operational failures.
- **Tradeoffs:** The merchant must attach an `"error"` handler to catch server-side initiation failures. This is the correct trade: an unhandled event is visible in logs; an unhandled exception crashes the process.

## D14: Status updates are delivered by client polling

- **Decision:** A PaymentInstance tracks status transitions by polling the one-shot status operation at a configurable interval until the transaction reaches a terminal state. Each interval carries a small random jitter so many concurrent instances do not synchronise into a thundering herd against the status endpoint. There is one status transport, and it is polling.
- **Context:** Async payment operations need status-transition notifications without forcing merchants to write polling loops. The PaymentInstance owns that loop internally. The transport must work everywhere an SDK runs, including restricted networks and proxies that buffer or break long-lived connections.
- **Alternatives considered:** (a) Push status over a server-sent events (SSE) stream with polling as a fallback. Evaluated and removed: it required two code paths (stream plus the fallback poll), a streaming HTTP client able to set auth headers, and a bounded read buffer. All that to optimise latency on a path the fallback already had to cover. The fallback being mandatory meant polling could never be removed, so the stream was pure added surface. (b) WebSockets. Rejected: heavier than needed for one-directional status push. (c) Server-to-merchant webhooks as a substitute. Rejected: webhooks are the merchant-server push channel, not the PaymentInstance's own update source.
- **Rationale:** One transport is simpler to reason about, test, and port across languages. Polling is robust on every network and needs no special streaming client. It reuses the same signed transport and the same status shape as a one-shot status check, so a polled update and a one-shot check are handled identically. Jitter keeps a fleet of instances from aligning their requests.
- **Tradeoffs:** Higher latency and more requests than a push stream. Accepted: status transitions are not latency-critical, the interval and caps are configurable, and jitter plus single-flight polling (only one request in flight per instance) bound the load.

## D15: Response verification is fail-closed

- **Decision:** Every authenticated success response MUST carry a valid `_responseSignature`. When the signature is **missing** or **invalid**, the SDK MUST reject the response, returning an `internal` error and exposing no data. The SDK never returns response data it has not verified.
- **Context:** The backend signs every SDK success response (the server either signs the response or fails the action; there is no unsigned success path).
- **Alternatives considered:** (a) Verify only when a signature is present (fail-open). Rejected: a man-in-the-middle (or a malicious endpoint) could strip `_responseSignature`, and the SDK would then surface forged data as live. Examples: a forged `"successful"` status, a forged amount, a forged phone-verification result. (b) Make signing optional per-endpoint. Rejected: every SDK action already signs; optionality only reintroduces the hole.
- **Rationale:** Signature verification only protects integrity if a missing signature is treated as a failure. Fail-closed is the only posture consistent with the rule that the SDK never exposes unverified data. The comparison is constant-time and length-guarded, so malformed signatures are rejected without raising an error.
- **Tradeoffs:** An SDK pointed at a backend that does not sign responses will reject everything. Acceptable and intended: an unsigned response is indistinguishable from a tampered one.

## D16: Webhook verification is replay-protected

- **Decision:** `verifyWebhookSignature` rejects a correctly-signed webhook whose **signed timestamp** is outside a tolerance window (default 300s). Authenticity (HMAC over the raw body) and freshness (signed timestamp within the window) must both hold. The freshness anchor is the timestamp carried **inside the signed body**, read only after the HMAC verifies. An unsigned transport header is never trusted for freshness. `0` means a tolerance of zero seconds, maximum strictness, and does NOT disable the check; opting out requires the explicit `DISABLE_FRESHNESS_CHECK` sentinel (D20).
- **Context:** HMAC authenticates *who* sent the webhook, not *when*. Without an expiry, a captured `(body, signature)` pair stays valid forever, so an attacker who records one delivery can replay it indefinitely (re-trigger a fulfilment, re-credit an order). The backend stamps every delivery, first attempt and every retry, with a current `timestamp` inside the signed body, so the signed content needed for a freshness check is already on the wire.
- **Alternatives considered:** (a) Leave it to merchants (idempotency on their side). Rejected: replay protection is a property of the verifier; relying on every consumer to dedupe is fail-open by default. (b) Sign `${timestamp}.${body}` and add a header, with a dual-sign migration. Rejected as unnecessary: the timestamp is *already* inside the signed body, so no wire/sender change is required, the fix is verifier-only. (c) Trust the `x-nylon-timestamp` header. Rejected: the header is not covered by the HMAC, so a replay attacker could pair an old body with a fresh header; only the signed-body timestamp is trustworthy.
- **Rationale:** Reusing the already-signed timestamp makes replay protection a pure verifier change with no coordinated rollout. Because each genuine delivery (including a retry hours later) is re-stamped and re-signed, a 5-minute window never rejects legitimate traffic. Only a replayed capture is rejected, since its embedded timestamp is now stale. Fail-closed on a missing timestamp keeps an unverifiable webhook from being treated as fresh.
- **Tradeoffs:** Clock skew between the backend and the consumer must stay within the tolerance; the window is configurable for slow consumers, and `DISABLE_FRESHNESS_CHECK` opts out entirely. A webhook body without a signed timestamp is rejected when the check is on, acceptable, since every Nylon Pay delivery carries one.

## D17: Canonical payload uses the JSON Canonicalization Scheme (JCS)

- **Decision:** The canonical payload that feeds every signature (requests and responses) is the RFC 8785 (JCS) serialization. Object keys are sorted by **Unicode code point** (UTF-16 code-unit order) recursively. Arrays are left in place. Numbers use the shortest round-tripping form (ECMAScript number-to-string), with minimal JSON string escaping and no insignificant whitespace. A locale-sensitive key comparison is prohibited.
- **Context:** Key order feeds every signature, so it must be a function of the payload alone, not of any runtime environment. A locale-sensitive comparison orders keys by the runtime's locale and Unicode/ICU data. Two parties, a future non-JS SDK, or the same runtime under a different locale or ICU build can then canonicalize the same payload to different bytes. Such parties reject each other's otherwise-valid signatures. It is an availability bug (a valid request/response fails verification), not a forgery risk: an attacker still cannot produce a valid signature without the secret. It is latent while every party shares one collation. It surfaces on keys that order differently under locale vs code point: realistically, merchant-supplied `metadata` with mixed-case or non-ASCII keys.
- **Alternatives considered:** (a) Locale sorting with a fixed locale everywhere. Rejected: unenforceable across languages and deployments, and still breaks on Unicode-version differences. (b) Define an ad-hoc sort. Rejected: JCS is the published, language-neutral standard with reference implementations. (c) Dual-verify transition (accept both canonical forms during migration). Considered but not taken: only divergent-key payloads differ between the two forms, and those are rare, so a clean lockstep switch was chosen over carrying a second code path.
- **Rationale:** Code-point order is defined by the string itself, not the environment, so it is identical on every runtime. JCS additionally pins number and string serialization, so a non-JS SDK has an unambiguous target instead of having to match whatever JavaScript emits. In JavaScript, `JSON.stringify` already produces JCS-conformant numbers and string escaping, so the implementation is the code-point key sort plus that serializer.
- **Tradeoffs:** The canonical form is part of signed content, so the SDK and backend must move together (lockstep). A client and server on different canonical forms disagree **only** on divergent-key payloads. Ordinary camelCase/ASCII traffic is byte-identical under both, so the practical break is limited to the payloads that were already at risk.

## D18: The reference is the only transaction identity (no separate idempotency key, no heuristic duplicate detection)

- **Decision:** A merchant-supplied `reference` is the transaction identity AND the idempotency key. Same reference = same transaction: a create operation that reuses a reference replays the existing transaction (response carries `duplicate: true`) instead of charging again. A reference that is taken and cannot be replayed (it belongs to another account) fails with the `duplicate` error category. A fresh reference always starts a fresh payment. Amount, customer phone, email, and timing never block an operation on their own. There is no separate `idempotencyKey` input anywhere in the SDK surface.
- **Context:** The merchant already identifies the transaction with the `reference`. The backend enforces its uniqueness at the database and replays it deterministically, so identity and idempotency are the same thing and stay restart-proof.
- **Alternatives considered:** (a) Heuristic duplicate detection layered on top of the reference. That meant an in-memory cache keyed on `org:phone:amount:currency` as a 30-minute hard block, a one-pending-transaction-per-customer guard, and a 24-hour post-success cooldown per customer. Rejected: it second-guesses the merchant's own identity scheme and rejects legitimate payments, such as a customer paying the same amount twice or a retry after a stuck pending. The errors surfaced as a generic "Payment collection failed", so developers could not tell what was wrong or how to proceed. (b) Separate idempotency key + free-form reference. Rejected: two overlapping identities invite drift; the reference is already unique and provider-visible (`merchantTransactionId`). (c) Error (rather than replay) on same-account reference reuse. Rejected: replay is what makes network-failure retries safe.
- **Rationale:** Exact, explainable, restart-proof. Reference uniqueness is enforced by the database, replays are deterministic, and the failure mode ("this reference is taken, use a new one") is actionable. The `duplicate` category plus the `duplicate: true` replay flag give developers the signal the heuristics never did.
- **Tradeoffs:** A merchant bug that generates a new reference per retry can double-charge; accepted as the documented contract (retries MUST reuse the reference). Server-side abuse protection is preserved by rate limits (failure-based per customer, payout throughput/destination caps), which bound damage without blocking distinct transactions.

## D19: Retries are signed fresh per attempt; the reference (not the nonce) carries idempotency

- **Decision:** On a retry, the request **body is unchanged** (same payload, same `reference`), but each attempt is **re-signed** with a fresh `nonce`, `timestamp`, and `signature`. The SDK never freezes the signed identity across attempts. Idempotency and double-processing safety rest entirely on the constant `reference`. See [D18](#d18-the-reference-is-the-only-transaction-identity-no-separate-idempotency-key-no-heuristic-duplicate-detection).
- **Context:** Reusing one nonce/timestamp/signature for every attempt (a "byte-identical retry") fails against how the backend actually verifies requests:
  - The request `timestamp` is frozen at the first attempt. Once the cumulative backoff exceeds the server's freshness window (default 300s), the retry is rejected as stale. The retry budget (`maxRetries` × `timeoutMs`) is merchant-configurable and uncoupled from that window, so an aggressive config can age out.
  - The server's nonce store **rejects** a repeated nonce (it does not replay a cached result). Any retry that reached the auth guard a second time is refused as a replay, meaning retries only ever help for failures that occur *before* the server records the nonce.

  Nonce reuse therefore adds no idempotency (the reference already provides it) while making retries fragile.
- **Alternatives considered:** (a) Keep frozen identity, cap the retry budget below the freshness window. Rejected: it patches only the timestamp edge and leaves the self-replay-rejection. It also re-introduces the same cross-codebase coupling (the SDK must track the server's window). (b) Keep frozen identity, document the coupling. Rejected: leaves the fragility in place. (c) Server returns a cached response on nonce replay (idempotent nonce). Rejected: duplicates the reference's job and stores per-nonce responses for no benefit. (d) Fresh identity per attempt (chosen).
- **Rationale:** Re-signing per attempt is what every normal HTTP client does; a retry is just another legitimately-signed request the trusted client generates. Each attempt then always carries a current timestamp (never ages out) and a distinct nonce (never self-rejected), so the retry reaches the action and the well-tested reference-idempotency path. Safety is unchanged: same reference → the backend replays the existing transaction with `duplicate: true` rather than charging again. DB-level reference uniqueness backstops the race where two attempts are in flight at once, which is timeout-bounded and effectively nil. Nonce reuse was the anomaly: it *blocked* retries from reaching the path D18 designed to make them safe.
- **Tradeoffs:** Anti-replay of a *captured* request is unaffected. An attacker still cannot forge a signature without the secret, and the server's nonce + timestamp checks still bound any single captured request. Only the legitimate client's own retries now re-sign. Retries that reach the server consume rate-limit budget like any request (acceptable, and they are few). The backend needs no change, it never required nonce reuse.

## D20: `0` tolerance means strict, not disabled

- **Decision:** `tolerance_seconds` / `toleranceSeconds` of `0` means a tolerance of zero seconds: maximum strictness, which in practice rejects everything. Disabling the freshness check requires passing the exported `DISABLE_FRESHNESS_CHECK` sentinel deliberately.
- **Context:** A developer hardening a webhook handler reaches for `0` to mean "zero tolerance / strictest possible". If `0` disabled the check, that merchant would silently get the exact opposite: no replay protection at all, permanently, for that call site. The failure is invisible, verification keeps returning `true`, so nothing looks wrong.
- **Alternatives considered:** (a) Keep `0` as disabled and document it harder. Rejected: a security control that fails OPEN on a plausible misreading is not fixed by documentation. The whole point of the value is that someone reaching for it is thinking about security. (b) Reject `0` as invalid input. Rejected: `0` has an obvious, coherent meaning under the correct reading, and erroring on it would break callers for no benefit.
- **Rationale:** Between the two possible misunderstandings, the safe one wins. Meaning "strict" and getting "disabled" silently removes replay protection. Meaning "disabled" and getting "strict" rejects stale webhooks, visible immediately and safe. A named sentinel also makes the opt-out greppable in review, which `0` never was.
- **Tradeoffs:** Breaking for anyone passing `0` to disable the check: they start rejecting stale webhooks until they switch to the sentinel.

## D21: Signed responses are bound to the request that asked for them

- **Decision:** The backend echoes the originating request's nonce inside the **signed** response payload as `_requestNonce`. After the HMAC verifies, the SDK requires it to equal the nonce it just sent, and strips it before returning data to the merchant. A response that omits it or carries a different one is rejected as `internal`.
- **Context:** The request side always signs a fresh nonce specifically so a leaked signature cannot be replayed. The response side had no equivalent: `verifyResponseSignature(data, signature, secret)` authenticated only `HMAC(secret, canonical(data))`, with nothing tying it to the call it answered. So any response the backend ever legitimately produced stayed validly signed forever. Anyone able to observe one (a compromised proxy, request/response logging, a buggy cache, log access) could replay a captured "successful" status onto a later poll for the same reference. The SDK could not tell the reply from a live answer, even after the real transaction had failed or been reversed.
- **Alternatives considered:** (a) A client-side freshness guard tracking the last-seen `updatedAt` per reference. Rejected as the primary fix: it is per-process state, lost on restart, and scoped to one action. By construction it cannot stop a replay of the *most recent* response; it narrows the window rather than closing it. (b) Sign a server timestamp into the response. Rejected: bounds staleness but still does not bind the response to a *specific* request, so a capture inside the window is still replayable. (c) Response nonces tracked server-side. Rejected: needs shared state across instances for no gain over echoing what the client already sent.
- **Rationale:** The nonce is already generated per request, already unique, and already on the wire. Echoing it inside the signed payload reuses that with no new state anywhere: the client compares against a value it already holds. Because the echo lives **inside** the signed data, an SDK that does not know about it still verifies the signature normally, so the backend change is additive and can ship first.
- **Tradeoffs:** Deployment is ordered: the backend must echo the nonce **before** any SDK that requires it is published, or every call from an updated SDK fails verification. The reverse order is safe (an SDK that does not require the echo ignores the field and verifies normally). The check is deliberately strict rather than "verify it only when present", since a lenient check would accept exactly the pre-change responses an attacker is most likely to hold.