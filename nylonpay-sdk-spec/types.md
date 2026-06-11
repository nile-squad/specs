# Types and Events

Part of the [Nylon Pay SDK Spec](./spec.md).

## Type Definitions

TypeScript types are used below as the reference notation. Each language defines equivalent types using its own type system. The shapes, field names, and value constraints are identical across all implementations.

```typescript
type TransactionStatus =
  | "pending"
  | "processing"
  | "successful"
  | "failed"
  | "cancelled";

type TransactionType =
  | "collection"
  | "payout"
  | "transfer"
  | "escrow"
  | "refund"
  | "reversal"
  | "charge"
  | "chargeback";

type PaymentMethod = "mobileMoney" | "bank";

type TransactionMode = "test" | "live";

type PaymentEvent =
  | "processing"
  | "success"
  | "failed"
  | "cancelled"
  | "error";

type WebhookEventType =
  | "collection.completed"
  | "collection.failed"
  | "payout.completed"
  | "payout.failed"
  | "payout.reversed"
  | "refund.completed"
  | "chargeback.received";

type Currency = "USD" | "EUR" | "GBP" | "KES" | "UGX" | "TZS" | "RWF";

// Every hook is wrapped: `fn` is the handler, `onError` (required) receives any
// throw/rejection from `fn`, and `enabled` (default true) toggles it off.
// The SDK runs `fn` inside a safe boundary so merchant code can never crash the
// payment flow — failures route to `onError` instead of bubbling.
type SdkHook<Fn> = {
  enabled?: boolean; // default true
  fn: Fn;
  onError: (error: unknown) => void | Promise<void>;
};

type SdkHooks = {
  beforeCollect?: SdkHook<
    (
      input: CollectPaymentInput,
    ) => CollectPaymentInput | void | Promise<CollectPaymentInput | void>
  >;
  afterCollect?: SdkHook<
    (
      result: Result<{ reference: string; status: string }, string>,
      input: CollectPaymentInput,
    ) => void | Promise<void>
  >;
  beforePayout?: SdkHook<
    (
      input: MakePayoutInput,
    ) => MakePayoutInput | void | Promise<MakePayoutInput | void>
  >;
  afterPayout?: SdkHook<
    (
      result: Result<{ reference: string; status: string }, string>,
      input: MakePayoutInput,
    ) => void | Promise<void>
  >;
};

type NylonPayConfig = {
  apiKey: string;
  apiSecret: string;
  baseUrl?: string;
  timeoutMs?: number;
  maxRetries?: number;
  maxPollIntervalMs?: number;
  maxPollDurationMs?: number;
  maxPollAttempts?: number;
  /** Custom fetch implementation. Defaults to `globalThis.fetch`. Essential for edge runtimes and testing. */
  fetch?: typeof globalThis.fetch;
  /** Force a new instance even if one already exists for this key+secret+url. Defaults to `false`. See D11. */
  force?: boolean;
  hooks?: SdkHooks;
};

type Customer = {
  name: string;
  /** Phone number in any common format — normalized automatically to international format */
  phoneNumber: string;
  email?: string;
};

#### Phone Number Normalization

Every `phoneNumber` field accepted by the SDK is normalized to international format
(`256XXXXXXXXX`) before it reaches the backend. The normalization runs at three
layers for defense-in-depth:

1. **SDK (client-side)** — `normalizePhone()` runs synchronously before the request
   is signed and sent. The wire payload always carries the normalized number.
2. **Backend Zod schema** — `phoneNumberSchema` validates then transforms the
   number to normalized form. Catches callers that bypass the SDK.
3. **Provider formatters** — `formatPhoneForPivot()` normalizes before handing the
   number to the provider. Defense-in-depth at the provider boundary.

Normalization rules:

| Rule | Example |
|------|---------|
| Strip all whitespace | `+256 768 499 027` → `+256768499027` |
| Strip leading `+` | `+256768499027` → `256768499027` |
| If starts with `0` and length is 10, prepend `256` | `0768499027` → `256768499027` |
| Already normalized passes through | `256768499027` → `256768499027` |

The normalized result is what gets stored in the `Transaction.phone` field and sent
to payment providers.

**Accepted input formats (any of these work):**

| Format | Pattern | Example |
|--------|---------|---------|
| Local (10-digit) | `0XXXXXXXXX` | `0768499027` |
| International with `+` | `+256XXXXXXXXX` | `+256768499027` |
| International without `+` | `256XXXXXXXXX` | `256768499027` |
| With spaces (any format) | — | `+256 768 499 027`, `256 768 499 027` |

Merchants can pass phone numbers in any of these formats. The system handles
normalization — the merchant does not need to format numbers before calling
the SDK.

type Destination = {
  accountHolderName: string;
  accountNumber: string;
  bankName?: string;
  phone?: string;
};

type InvoiceItem = {
  name: string;
  quantity: number;
  unitPrice: number;
};

type BankDetails = {
  accountNumber: string;
  bankName: string;
};

type CollectPaymentInput = {
  amount: number;
  currency: Currency;
  customer: Customer;
  description: string;
  reference?: string;
  method?: PaymentMethod;
  bank?: BankDetails;
  metadata?: Record<string, string>;
};

type MakePayoutInput = {
  amount: number;
  currency: Currency;
  customer: Customer;
  destination: Destination;
  description: string;
  reference?: string;
  metadata?: Record<string, string>;
};

type GetStatusInput = {
  reference: string;
};

type GetTransactionInput = {
  id?: string;
  reference?: string;
};

type VerifyPhoneInput = {
  phoneNumber: string;
  purpose?: "collection" | "payout";
};

type CreateInvoiceInput = {
  amount: number;
  currency: Currency;
  description: string;
  items?: InvoiceItem[];
  redirectUrl?: string;
  reference?: string;
  metadata?: Record<string, string>;
};

type VerifyWebhookInput = {
  payload: string | Uint8Array;
  signature: string;
  secret: string;
};

type Transaction = {
  id: string;
  reference: string;
  amount: number;
  currency: Currency;
  status: TransactionStatus;
  type: TransactionType;
  method: PaymentMethod;
  description: string;
  // Present (true) only when this response replayed an existing transaction
  // for a reused reference — no new payment was initiated. See
  // "Reference uniqueness and replay" in operations.md.
  duplicate?: boolean;
  /** Normalized international format (256XXXXXXXXX) — see Phone Number Normalization below */
  phone: string;
  email: string | null;
  failureReason: string | null;
  metadata: Record<string, string>;
  mode: TransactionMode;
  createdAt: string;
  updatedAt: string;
};

type StatusResponse = {
  reference: string;
  status: TransactionStatus;
  amount: number;
  currency: Currency;
  updatedAt: string;
};

type PhoneVerification = {
  phoneNumber: string;
  customerName: string;
  verified: boolean;
};

type InvoiceResponse = {
  id: string;
  url: string;
  token: string;
  expiresAt: string;
  status: "pending";
};

type WebhookPayload = {
  event: WebhookEventType;
  data: Transaction;
  timestamp: string;
  signature: string;
};
```

## Transaction Shape

The transaction record returned by `getTransaction`, `wait()`, and event handlers. See `Transaction` type above for the full shape.

## Webhook Event Catalog

Events are delivered as POST requests to the merchant's configured webhook URL. Each event has a `type` field identifying the event and a `data` field containing the relevant transaction record.

| Event Type | Trigger |
|------------|---------|
| `collection.completed` | A collection transaction reaches `successful` status |
| `collection.failed` | A collection transaction reaches `failed` status |
| `payout.completed` | A payout transaction reaches `successful` status |
| `payout.failed` | A payout transaction reaches `failed` status |
| `payout.reversed` | A failed payout is reversed by the system |
| `refund.completed` | A refund transaction is processed successfully |
| `chargeback.received` | A chargeback is initiated by a bank or provider |

**Webhook payload shape:** See `WebhookPayload` type above.

**Delivery guarantees:**
- At-least-once delivery with exponential backoff retries
- Merchants must respond with 2xx within the timeout window
- Duplicate delivery is possible — merchants must be idempotent (use `reference` for deduplication)

