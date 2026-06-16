# Error Categories

Part of the [Nylon Pay SDK Spec](./spec.md).

Every SDK error is a structured `SdkError`:

```typescript
type SdkErrorCategory =
  | "auth"        // invalid/missing/revoked/expired key, bad signature, replay, scope
  | "validation"  // input the server rejected
  | "limit"       // account/KYC transaction limits exceeded
  | "rate_limit"  // too many requests
  | "account"     // merchant account missing or not active
  | "provider"    // payment provider/engine rejected the operation
  | "duplicate"   // reference already used for another transaction — retry with a NEW reference
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

- `auth`, `validation`, `limit`, `rate_limit`, `account`, `provider`, `duplicate`, `not_found`, `internal` are server-tagged via the message suffix.
- `network`, `timeout` are produced by the SDK transport when the request never completed.
- Merchants branch on `category`, never on `message` text or HTTP status.

## Message style — humanized, no internal mechanics

The `category` (and `retryable`) carry the machine-readable signal; the `message`
is for a human and MUST stay human. Every `message` the SDK surfaces:

- States the outcome and, where useful, a next step, in plain language.
- Exposes NO internal mechanics — no `polling`, `nonce`, `HMAC`/signature
  verification, library names, internal field names, or raw error/stack dumps.
- Expresses any time or duration in human units ("about 2 minutes", "a few
  seconds"), never raw milliseconds or ISO timestamps.

Examples: a status-resolution timeout reads "Timed out waiting for the transaction
status to update" (category `timeout`), not "Polling timeout: exceeded maximum
duration"; an unverifiable response reads "Could not verify the server response"
(category `internal`), not "Response signature verification failed". The same rule
applies to any duration the SDK or backend renders for a person (for example
processing time in a transaction view): humanized, never raw units.

## The `duplicate` category

The reference is the transaction identity: **same reference = same transaction**.

- Reusing a reference **you own** does not error — the server replays the
  existing transaction, and the response carries `duplicate: true` (see the
  `Transaction` shape). No new payment is initiated.
- A `duplicate` **error** means the reference is taken and cannot be replayed to
  you (it belongs to another account). Retrying the same call with a **new,
  unique reference** will pass.
- Amount, customer phone, and timing never trigger `duplicate` on their own —
  only the reference does.

