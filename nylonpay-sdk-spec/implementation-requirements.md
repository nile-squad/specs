# Implementation Requirements

Part of the [Nylon Pay SDK Spec](./spec.md).

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

