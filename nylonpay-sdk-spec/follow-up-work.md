# Follow-Up Work

Part of the [Nylon Pay SDK Spec](./spec.md).

## F1: Card payments via SDK

- **What:** Card payments are only available through `createInvoice` (hosted payment page). The SDK does not accept card details (card number, CVV, expiry) directly.
- **Why it's deferred:** Handling raw card data carries PCI-DSS compliance obligations. The hosted payment page handles card collection securely on behalf of the merchant.
- **Impact of deferring:** Merchants collecting card payments use `createInvoice` to generate a hosted payment link. This is the supported path, not a gap.

## F2: Batch operations

- **What:** No batch API for multiple collections or payouts in a single call.
- **Why it's deferred:** No server-side batch endpoint exists. Client-side batching (loop + concurrency control) adds complexity without atomicity guarantees.
- **Impact of deferring:** Merchants making bulk payouts loop over `makePayout` individually.

## F3: Client-side SDKs

- **What:** No browser or mobile SDK exists. All current SDKs are server-side.
- **Why it's deferred:** Client-side SDKs require a different security model (no secret key in client), tokenization endpoints, and hosted checkout integration.
- **Impact of deferring:** Client-side payment collection uses the hosted payment page via `createInvoice`.

## F4: SDK version header

- **What:** The SDK does not send its version in request headers for server-side compatibility checks.
- **Why it's deferred:** No server-side version negotiation exists. All SDK versions target the same API contract.
- **Impact of deferring:** Breaking API changes require coordinated SDK updates. Mitigated by semantic versioning and changelog communication.

## F5: Server-push status transport

- **What:** Status updates are delivered by client polling ([D14](./decision-records.md#d14-status-updates-are-delivered-by-client-polling)). A server-push transport (e.g. server-sent events or a long-lived stream) could lower transition latency and request volume.
- **Why it's deferred:** A push transport was prototyped (SSE with a polling fallback) and removed. It needed two code paths, a streaming HTTP client able to set auth headers, and a bounded read buffer, all while polling still had to exist as the universal fallback. The added surface did not justify the latency gain for a non-latency-critical path. Revisiting requires a transport that works across restrictive proxies without a mandatory polling fallback, or evidence that latency matters enough to carry both paths.
- **Impact of deferring:** Status latency is bounded by the poll interval (default 2s) rather than near-real-time. Acceptable: the interval and caps are configurable, and jitter plus single-flight polling bound the load.

Resolved items and their history are recorded in the [Changelog](./changelog.md).
