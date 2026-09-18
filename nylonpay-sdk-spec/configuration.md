# Configuration

Part of the [Nylon Pay SDK Spec](./spec.md).

See `NylonPayConfig` type above.

## Validation at init
- `apiKey` must be present and start with `npk_`
- `apiSecret` must be present and start with `nps_`
- Invalid config throws immediately (programmer error)

## Test vs. live mode

Mode is determined by the API key, not by SDK config. A
sandbox key (issued in test mode) routes transactions through test providers and
does not move real money; a live key processes real transactions. The SDK has no
`environment` option.

## Defaults

| Field | Default |
|-------|---------|
| `baseUrl` | `https://api.nylonpay.nilesquad.com/api/services` |
| `timeoutMs` | `90000` |
| `maxRetries` | `3` |
| `maxPollIntervalMs` | `2000` |
| `maxPollDurationMs` | *(none, poll until terminal)* |
| `maxPollAttempts` | *(none, poll until terminal)* |
| `onDelayed` | `"wait"` |

## Polling caps

`wait()` and `*AndResolve` poll until the transaction reaches a terminal state;
there is no default cap. Merchants who want a bounded wait set
`maxPollDurationMs` and/or `maxPollAttempts`, or use `onDelayed: "return"` to
hand back a still-pending payment once it is flagged delayed (see
[PaymentInstance Contract](./payment-instance.md)).

## Delayed payments

When a non-terminal payment has been in flight for more than
three minutes, status responses include `delayed: true`. This is a flag on the
response, not a new `status` value. Merchants choose per request whether to keep
waiting (`onDelayed: "wait"`, default) or return the pending payment and rely on
webhooks (`onDelayed: "return"`).

## Offline and Nylon down

Pass `onError` when creating the SDK instance to handle structured errors from
all operations on that instance. The handler does not create another event
family. Payment operations still emit their normal PaymentInstance `"error"`
event, and Result operations still return an error.

```typescript
const nylonpay = createNylonPay({
  apiKey,
  apiSecret,
  onError: (error) => {
    if (error.code === "unreachable") {
      pauseCalls(error.message);
      return;
    }
    logSdkError(error);
  },
});
```

The `unreachable` code means the request could not complete. Its message is
exactly one of:

| `reason` | Meaning |
|----------|---------|
| `host has no internet connection` | DNS or no-route failure. This machine cannot reach the network. |
| `Nylon Pay services seem to be down` | The host has a network, but Nylon Pay did not complete the request. |

Implementations MUST export these two strings as named constants
(`UNREACHABLE_HOST_OFFLINE`, `UNREACHABLE_NYLON_DOWN`) so merchants compare
constants rather than copied prose. Implementations MUST also export
`UNREACHABLE_CODE` (`"unreachable"`).

### When the SDK checks

The SDK keeps the last successful round-trip and the last check time in memory.
It MUST NOT run a reachability check on every call.

| Situation | Required behavior |
|-----------|-------------------|
| Last success is still fresh (fewer than 5 minutes) | Skip the check. Send the signed operation. |
| Last check (success or failure) is older than 5 minutes | Check again. MUST NOT return `unreachable` from memory that old. Hours-old memory is not proof Nylon is still down. |
| No recent success in memory | The next signed request is the check. Do not add an extra request on a cold start. |
| Last call failed as unreachable | Check before the next SDK operation. MUST NOT send while that check still says down. |
| Still down, last check within 15 seconds | Return `network` / `code: "unreachable"` without hitting the network. Call `onError` for the returned error. |

While unreachable, status polls MUST NOT re-check on every tick. Re-check at
most every 15 seconds so a PaymentInstance poll loop cannot hammer the server.

### How a check is performed

A dedicated check is an unsigned POST of `{}` to `{baseUrl}` with a 3-second
timeout. Any HTTP response except 502/503/504 means Nylon answered. The SDK
MUST NOT call a non-`sdk` service for this. See [D22](./decision-records.md#d22-reachability-checks-only-when-there-is-no-recent-success).

A failed check, or a signed request that never completed, returns
`network` / `code: "unreachable"` and calls `onError`. Further signed sends are
skipped until the next check.

Handler failures MUST be contained and MUST NOT change the operation result.
