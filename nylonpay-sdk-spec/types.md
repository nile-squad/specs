# Types and Events

Part of the [Nylon Pay SDK Spec](./spec.md).

## Type Definitions

TypeScript types are used below as the reference notation. Each language defines equivalent types using its own type system. The shapes, field names, and value constraints are identical across all implementations.

```typescript
type TransactionStatus =
  | "pending"
  | "processing"
  | "on_hold"
  | "successful"
  | "failed"
  | "cancelled";

/**
 * Lifecycle states a transaction can occupy. Merchants use these to drive
 * fulfillment logic: trigger order completion on "successful", notify
 * customer on "failed", release inventory on "cancelled".
 *
 * Non-terminal statuses: "pending", "processing", "on_hold" (review-stage payouts).
 * Terminal statuses: "successful", "failed", "cancelled".
 */

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
  | "transaction.successful"
  | "transaction.failed"
  | "transaction.processing"
  | "transaction.cancelled";

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

// The input handed to an after* hook: the final wire payload (reference
// resolved, phone normalized, before*-hook mutations applied), plus `raw`
// carrying the untouched original merchant input. This lets a hook log both
// what hit the wire and what the merchant typed.
type AfterHookInput<Input> = Input & { raw: Input };

type SdkHooks = {
  beforeCollect?: SdkHook<
    (
      input: CollectPaymentInput,
    ) => CollectPaymentInput | void | Promise<CollectPaymentInput | void>
  >;
  afterCollect?: SdkHook<
    (
      result: Result<{ reference: string; status: string }, string>,
      input: AfterHookInput<CollectPaymentInput>,
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
      input: AfterHookInput<MakePayoutInput>,
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
  /** Optional cap. When omitted, wait() polls until terminal. */
  maxPollDurationMs?: number;
  /** Optional cap. When omitted, wait() polls until terminal. */
  maxPollAttempts?: number;
  /** When a polled payment reports delayed: true — "wait" (default) keeps polling; "return" resolves with the still-pending payment. */
  onDelayed?: "wait" | "return";
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
  amount: number;
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
  tags?: string[];
  metadata?: Record<string, string>;
};

type MakePayoutInput = {
  amount: number;
  currency: Currency;
  customer: Customer;
  destination: Destination;
  description: string;
  reference?: string;
  tags?: string[];
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
  customerEmail: string;
  customerName?: string;
  customerPhone?: string;
  description?: string;
  dueDate?: string;
  items?: InvoiceItem[];
  merchantReference?: string;
  tags?: string[];
  metadata?: Record<string, string>;
};

type ListTransactionsInput = {
  tags?: string[];
  status?: TransactionStatus;
  type?: "collection" | "payout" | "invoice";
  limit?: number;   // 1–100, default 20
  offset?: number;  // default 0
  createdAfter?: string;  // ISO 8601
  createdBefore?: string; // ISO 8601
};

type TransactionSummary = {
  id: string;
  reference: string;
  amount: number;
  currency: Currency;
  status: TransactionStatus;
  type: TransactionType;
  method: string | null;
  mode: TransactionMode;
  tags: string[];
  createdAt: string;
  updatedAt: string;
};

type ListTransactionsResponse = {
  transactions: TransactionSummary[];
  count: number;
  limit: number;
  offset: number;
  tags: string[];
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
  // The underlying operator's (telco's/bank's) own transaction id — what the
  // paying customer sees on their receipt. For cross-validating customer pay
  // claims. Null until the operator reports it (typically at terminal status).
  operatorTid?: string | null;
  /** Normalized international format (256XXXXXXXXX) — see Phone Number Normalization below */
  phone: string;
  email: string | null;
  failureReason: string | null;
  /**
   * Humanized status description. For `on_hold` statuses, this provides
   * a plain-language explanation (e.g., "Payout is being reviewed and will
   * complete shortly"). For failed transactions, this is typically the same
   * as `failureReason`. Populated by the backend when available.
   */
  statusText?: string;
  metadata: Record<string, string>;
  mode: TransactionMode;
  createdAt: string;
  updatedAt: string;
  /** True when the payment has been in a non-terminal state longer than the delayed threshold (~3 minutes). Not a status value. */
  delayed?: boolean;
};

type StatusResponse = {
  reference: string;
  status: TransactionStatus;
  amount: number;
  currency: Currency;
  /**
   * Humanized status description. For `on_hold` statuses, this provides
   * a plain-language explanation (e.g., "Payout is being reviewed and will
   * complete shortly"). Populated by the backend when available.
   */
  statusText?: string;
  updatedAt: string;
  /** True when the payment has been in a non-terminal state longer than the delayed threshold (~3 minutes). Not a status value. */
  delayed?: boolean;
};

type PhoneVerification = {
  phoneNumber: string;
  customerName: string;
  verified: boolean;
};

type InvoiceResponse = {
  id: string;
  invoiceNumber: string;
  paymentLink: string;
  amount: string;
  currency: string;
  status: string;
};

type WebhookPayload = {
  /** Backend delivery id (`buildDeliveryBody` field name; snake_case on the wire). */
  delivery_id: string;
  event: WebhookEventType;
  payload: WebhookTransactionSnapshot;
  timestamp: string;
};
```

### WebhookTransactionSnapshot

Merchant-facing transaction record delivered inside a webhook payload. Field
names match the wire JSON exactly (camelCase in both TypeScript and Python)
because merchants type their `JSON.parse()` / `json.loads()` output against
this directly — it is NOT passed through the SDK's snake_case ↔ camelCase
wire conversion.

```typescript
type WebhookTransactionSnapshot = {
  transactionId: string;
  reference: string;
  /**
   * Decimal-string amount (matches backend wire JSON). Null only when the
   * backend could not read the transaction record while dispatching.
   */
  amount: string | null;
  currency: string | null;
  status: TransactionStatus;
  previousStatus: TransactionStatus;
  /** Null when the transaction has no stored value for the field. */
  type: TransactionType | null;
  method: PaymentMethod | null;
  mode: TransactionMode | null;
  failureReason: string | null;
  operatorTid: string | null;
};
```

Every key is always present — the backend sends an explicit null rather than
omitting one, so the shape a merchant types against never changes. There is no
`statusText` here: it belongs to the `Transaction` shape, and the statuses it
describes (`on_hold`, `under_review`) emit no webhook at all.

## Transaction Shape

The transaction record returned by `getTransaction`, `wait()`, and event handlers. See `Transaction` type above for the full shape.

## Webhook Event Catalog

Events are delivered as POST requests to the merchant's configured webhook URL. The body carries a `payload` field (not `data`) holding the merchant-facing transaction record. The signature does NOT live in the body — it travels in the `x-nylon-signature` HTTP header.

| Event Type | Trigger |
|------------|---------|
| `transaction.successful` | A transaction reaches `successful` status |
| `transaction.failed` | A transaction reaches `failed` status |
| `transaction.processing` | A transaction is mid-flight (rare; mostly for in-flight dashboards) |
| `transaction.cancelled` | A transaction is cancelled before reaching a terminal state |

**Webhook payload shape:** See `WebhookPayload` and `WebhookTransactionSnapshot` types above.

No other status emits a webhook. A payout parked for review (`on_hold`, `under_review`) stays silent until it resolves to one of the four above. Both collections and payouts use this same catalog — `payload.type` distinguishes them.

**Signature form:** lowercase hex, the one canonical form (see invariant 28). Verification rejects any other spelling.

**Delivery guarantees:**
- At-least-once delivery. Five attempts with exponential backoff over roughly fifteen minutes, then one attempt nightly for up to five further nights before the delivery is retired.
- Merchants must respond with 2xx within 10 seconds
- Duplicate delivery is possible — merchants must be idempotent (use `delivery_id` or `reference` for deduplication)

