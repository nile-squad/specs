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

- `auth`, `validation`, `limit`, `rate_limit`, `account`, `provider`, `not_found`, `internal` are server-tagged via the message suffix.
- `network`, `timeout` are produced by the SDK transport when the request never completed.
- Merchants branch on `category`, never on `message` text or HTTP status.

