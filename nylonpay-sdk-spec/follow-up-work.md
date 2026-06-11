# Follow-Up Work

Part of the [Nylon Pay SDK Spec](./spec.md).

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

- **What** — Status updates are delivered by client polling ([D14](./decision-records.md#d14-status-updates-are-delivered-by-client-polling)). A server-push transport (e.g. server-sent events or a long-lived stream) could lower transition latency and request volume.
- **Why it's deferred** — A push transport was prototyped (SSE with a polling fallback) and removed: it required two code paths, a streaming HTTP client able to set auth headers, and a bounded read buffer, all while polling still had to exist as the universal fallback. The added surface did not justify the latency gain for a non-latency-critical path. Revisiting requires a transport that works across restrictive proxies without a mandatory polling fallback, or evidence that latency matters enough to carry both paths.
- **Impact of deferring** — Status latency is bounded by the poll interval (default 2s) rather than near-real-time. Acceptable: the interval and caps are configurable, and jitter plus single-flight polling bound the load.

### F6: Webhook signature replay protection — RESOLVED

Resolved in v1.0.8. See [D16](./decision-records.md#d16-webhook-verification-is-replay-protected),
the [verifyWebhookSignature](./operations.md#verifywebhooksignature) contract, and invariant 23.
The original concern assumed a coordinated dual-sign rollout, but the backend
already stamps a current `timestamp` inside the signed body on every delivery and
retry — so the fix was a verifier-only freshness window (default 300s) with no
wire or sender change. Consumers should still apply their own idempotency as
defence in depth, but a captured webhook no longer verifies forever.

### F7: Locale-independent canonical payload sort — RESOLVED

Resolved in v1.0.8. See [D17](./decision-records.md#d17-canonical-payload-uses-the-json-canonicalization-scheme-jcs),
the [Request Signing](./transport.md#request-signing) canonical-payload rules, and invariant 3.
Both the SDK and the backend now sort object keys by Unicode code point (RFC 8785
/ JCS) instead of a locale-sensitive comparison, changed in lockstep. A
cross-language parity test covers mixed-case, underscore, digit, diacritic, CJK,
and non-BMP keys plus number and unicode string values. The change was a clean
break (no dual-verify): only divergent-key payloads differ between the old and
new canonical forms, and ordinary ASCII/camelCase traffic is byte-identical under
both.
