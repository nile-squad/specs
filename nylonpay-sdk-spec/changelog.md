# Changelog

Part of the [Nylon Pay SDK Spec](./spec.md).

The changelog records the spec's version history and its current version. The
`Version` field in [spec.md](./spec.md) points here, and the normative
documents describe the contract as it exists today. They do not narrate
superseded behavior. When a version changes, the contract change is recorded
here, not in the spec body.


## Current version

**2.1.0**


## 2.1.0

- Signing specified so it can be proven across languages: conformance vectors
  V1–V7 (S19–S21) pin the canonical payload and signatures byte-for-byte, and
  every SDK ships them as a unit test. New implementations must pass them before
  the first live request.
- Optional signing-space documentation added: signing-flow, response
  verification, and webhook verification are each described in pseudocode in
  [Security](./security.md).
- Fingerprint composition pinned: SHA-256 of
  `type|platform|arch|release|hostname` (labels prefixed, joined with `|`),
  lowercase hex, computed once per process.
- `Result<T, E>` shape defined explicitly in [Types](./types.md); transport
  action table completed so each operation maps to exactly one action.
- Build [guide](./build-guide.md) added as the spec's recommended entry point:
  component list, dependency order, and a verify gate per step.
- Spec rebased to describe the current contract only. Superseded behavior and
  version archaeology (old hook shapes, verify-if-present, locale key sort,
  heuristic duplicate detection, `0`-means-off) were removed from the body;
  their reasoning lives in the [Decision Records](./decision-records.md), and
  their history lives here.
- Resolved follow-ups folded in: webhook replay protection, locale-independent
  canonical sort, and cross-language signing proof are now part of the contract
  (see [Follow-Up Work](./follow-up-work.md) for what remains open).


## 1.5

- `wait()` and `*AndResolve` poll until the transaction reaches a terminal
  state by default. The previous default polling cap was removed;
  `maxPollDurationMs` / `maxPollAttempts` now opt in to a bounded wait, and
  `onDelayed: "return"` hands back a still-pending payment once it is flagged
  delayed.


## 1.3.0

- The merchant-supplied `reference` became the only transaction identity and
  idempotency key (D18). Heuristic duplicate detection (same-amount-in-time
  blocks, one-pending-per-customer, post-success cooldown) was removed: a reused
  reference replays the existing transaction with `duplicate: true`, and a taken
  reference that cannot be replayed fails with the `duplicate` category.


## 1.0.8

- Response verification made fail-closed (D15): every authenticated success
  response must carry a valid `_responseSignature`; missing or invalid means the
  SDK returns an `internal` error and exposes no data.
- Webhook verification made replay-protected (D16): the signed-body timestamp
  must fall within the freshness window (default 300s) after the HMAC verifies.
- Canonical payload switched to JCS (RFC 8785) sorting (D17): object keys
  ordered by Unicode code point, language-neutral, so no implementation depends
  on a locale or ICU collation.
- Lifecycle hooks reshaped into `SdkHook<Fn>` wrappers (D10): each hook has a
  required `onError` and is contained, so a throwing hook never bubbles into the
  payment flow and is never silently swallowed.