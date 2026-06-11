# Invariants and Prohibitions

Part of the [Nylon Pay SDK Spec](./spec.md).

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
17. `collectPayment` and `makePayout` return a PaymentInstance that emits an `"error"` event on server-side initiation failure (auth, limit, provider, network, timeout). Only client-side validation errors (zero amount, empty fields, invalid items, out-of-range `reference` length) throw synchronously. A PaymentInstance may be constructed to carry an initiation error via the `initialError` mechanism.
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

