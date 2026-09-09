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
| `timeoutMs` | `30000` |
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
