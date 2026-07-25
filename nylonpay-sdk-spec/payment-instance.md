# PaymentInstance Contract

Part of the [Nylon Pay SDK Spec](./spec.md).

Every async operation returns a PaymentInstance with this interface:

**Properties:**
- `reference` — the transaction reference (read-only)
- `status` — current transaction status, set from the initiation response (typically `"pending"`) and updated with each status change (read-only)

**Methods:**
- `on(event, handler)` — register a handler for an event. Returns the instance (chainable).
- `once(event, handler)` — register a handler that fires at most once. Returns the instance.
- `off(event, handler)` — remove a handler. Returns the instance.
- `wait()` — blocks until a terminal state. Resolves with the full transaction on success, or `null` on failure, cancellation, merchant-configured timeout, or polling error. Never rejects. **Default:** polls until terminal with no duration or attempt cap.

**Events:**
- `processing` — the payment is accepted and in flight (covers `pending`, `processing`, and `on_hold` statuses — to the merchant they are the same lifecycle moment of in-flight transactions). Fires at most once per instance: emitted on the next tick after creation when the initiation response is non-terminal (so handlers registered after the operation returns still fire), or on the first poll that reports an in-flight status. A payment that resolves between polls (pending → successful) MUST still fire `processing` before the terminal event. For payouts specifically, `on_hold` indicates review-stage processing — polling continues automatically; use `transaction?.statusText` for human-readable status details when available.
- `success` — transaction completed successfully (terminal, polling stops)
- `failed` — transaction failed (terminal, polling stops)
- `cancelled` — transaction cancelled (terminal, polling stops)
- `error` — server-side initiation failure (auth, limit, provider, network, timeout) or lifecycle error (network failure, reference mismatch, polling timeout). Transaction never started. (terminal, polling stops)

```typescript
type EventData = {
  event: PaymentEvent;          // which event fired
  reference: string;            // transaction reference — always present, on every event
  transaction?: Transaction;    // full transaction on terminal status events
  error?: string;               // error message on the "error" event
  category?: SdkErrorCategory;  // machine-readable error category on "error" event
  retryable?: boolean;          // whether re-invoking may succeed, on "error" event
  timestamp: string;           // ISO 8601 timestamp of when the event was emitted
};
```

`transaction` is only guaranteed on terminal events (`success`, `failed`, `cancelled`) — the
`processing` event can fire before the full record is fetched, so merchants needing an
identifier there MUST use `reference`. Lifecycle events are deduplicated by event, not by
raw status: a `pending → processing` status change does not re-fire `processing`.

**Status updates:**
- Polls the one-shot status operation at a configurable interval (default 2s), with a small random jitter added to each interval so concurrent instances do not synchronise
- Only one poll is in flight at a time; the next poll is scheduled only after the current one resolves
- Stops on any terminal event
- **Default:** polls until terminal — no max attempts or max duration unless the merchant sets `maxPollDurationMs` and/or `maxPollAttempts`
- When a status poll reports `delayed: true` and `onDelayed` is `"return"`, `wait()` resolves immediately with the still-pending transaction (status remains `pending` or `processing`)
- When `delayed: true` and `onDelayed` is `"wait"` (default), polling continues until terminal
- Poll interval: base interval (default 2s) plus jitter for the first two minutes; doubles every two minutes thereafter, capped at 15s
- Reference mismatch between initiation and a status update emits `error` and stops
- Network errors during polling emit `error` and stop (after retries exhausted)

