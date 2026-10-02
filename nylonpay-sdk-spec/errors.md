# Error Reasons

Part of the [Nylon Pay SDK Spec](./spec.md).

Every SDK error is a structured `SdkError`:

```typescript
type SdkErrorReason =
  | "AUTH"          // invalid/missing/revoked/expired key, bad signature, replay, scope
  | "VALIDATION"    // input the server rejected
  | "LIMIT"         // account/KYC transaction limits exceeded
  | "RATE_LIMIT"    // too many requests
  | "ACCOUNT"       // merchant account missing or not active
  | "PROVIDER"      // payment provider/engine rejected the operation
  | "DUPLICATE"     // reference already used for another transaction, retry with a NEW reference
  | "NOT_FOUND"     // referenced transaction does not exist
  | "INTERNAL"      // unexpected server-side failure
  | "NETWORK"       // this machine is offline
  | "SERVICES_DOWN" // Nylon Pay did not complete the request
  | "TIMEOUT";      // request exceeded the configured timeout

type SdkError = {
  reason: SdkErrorReason;
  message: string;
  retryable?: boolean;
};
```

- `AUTH`, `VALIDATION`, `LIMIT`, `RATE_LIMIT`, `ACCOUNT`, `PROVIDER`,
  `DUPLICATE`, `NOT_FOUND`, `INTERNAL` are derived from the server's
  `-- error-type: <category>` suffix, uppercased at the parse boundary.
- `NETWORK` means this machine is offline (DNS or no-route). `SERVICES_DOWN`
  means Nylon Pay did not complete the request. `TIMEOUT` means the
  configured time limit expired. The transport produces these three.
- A skipped call (still down inside the re-check pause) uses `NETWORK` or
  `SERVICES_DOWN` from the last check. See [Offline and Nylon down](./configuration.md#offline-and-nylon-down).
- Merchants branch on `reason`, never on `message` text or HTTP status.
- The wire stays lowercase (`-- error-type: auth`). New SDKs normalize to
  ALL CAPS `reason`. An optional `-- error-code: <code>` may follow the
  category when the client listed `error-code` in `x-nylon-features`. A
  recognized outcome-named code becomes the reason. An unrecognized code
  falls back to the category-derived reason and is never surfaced as
  `reason`. Do not emit the code suffix to clients that cannot parse it.

`category` and string `code` remain deprecated aliases on `SdkError` for
this release line. Docs and snippets show `reason` only.

## Global error handler

`NylonPayConfig.onError` receives the structured `SdkError` returned by an
operation on the SDK instance. It also receives errors skipped by the
reachability guard. Retries are internal, so one final operation error produces
one handler call. Handler failures MUST be contained.

PaymentInstance `"error"` events and `Result` error values still fire as usual.
`onError` is the instance-wide place for logging and for `NETWORK` /
`SERVICES_DOWN` handling.

## Message style, humanized, no internal mechanics

The `reason` (and `retryable`) carry the machine-readable signal; the `message`
is for a human and MUST stay human. Every `message` the SDK surfaces:

- States the outcome and, where useful, a next step, in plain language.
- Exposes NO internal mechanics, no `polling`, `nonce`, `HMAC`/signature
  verification, library names, internal field names, or raw error/stack dumps.
- Expresses any time or duration in human units ("about 2 minutes", "a few
  seconds"), never raw milliseconds or ISO timestamps.

Examples: a status-resolution timeout reads "Timed out waiting for the transaction
status to update" (`TIMEOUT`), not "Polling timeout: exceeded maximum
duration"; an unverifiable response reads "Could not verify the server response"
(`INTERNAL`), not "Response signature verification failed". The same rule
applies to any duration the SDK or backend renders for a person (for example
processing time in a transaction view): humanized, never raw units.

## The `DUPLICATE` reason

The reference is the transaction identity: **same reference = same transaction**.

- Reusing a reference **you own** does not error, the server replays the
  existing transaction, and the response carries `duplicate: true` (see the
  `Transaction` shape). No new payment is initiated.
- A `DUPLICATE` **error** means the reference is taken and cannot be replayed to
  you (it belongs to another account). Retrying the same call with a **new,
  unique reference** will pass.
- Amount, customer phone, and timing never trigger `DUPLICATE` on their own,
  only the reference does.
