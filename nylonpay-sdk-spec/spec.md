# Nylon Pay SDK Spec

**Version:** 1.3.0

> Canonical, language-agnostic specification for the Nylon Pay SDK. Implement it
> in any language; the [TypeScript SDK](https://github.com/nile-squad/nylonpay-ts)
> is the reference implementation.

## Purpose

The Nylon Pay SDK is the merchant's programmatic interface to the payment platform. It provides a consistent API across multiple languages for collecting payments, making payouts, verifying phones, creating invoices, checking transaction status, and verifying webhooks. The SDK abstracts the transport protocol, HMAC signing, polling mechanics, and error handling so merchants interact with payment operations — not infrastructure.

All SDKs are server-side. Client-side packages (browser, mobile) are a future scope.

## Documents

The spec is split into focused documents. This page is the entry point; each
document is self-contained for its topic and links back here.

| Document | What it covers |
|----------|----------------|
| [Principles](./principles.md) | The six design principles every SDK follows |
| [Operations](./operations.md) | Every operation: inputs, outputs, reference constraints, hosted invoice behavior |
| [PaymentInstance Contract](./payment-instance.md) | The event-driven instance returned by async operations: events, polling lifecycle, `wait()` |
| [Types and Events](./types.md) | Type definitions, the Transaction shape, and the webhook event catalog |
| [Transport Contract](./transport.md) | Endpoint, request envelope, per-action payload validation, a worked request/response example, signing, and response verification |
| [Error Categories](./errors.md) | The fixed error taxonomy and how categories travel on the wire |
| [Configuration](./configuration.md) | Factory configuration: keys, base URL, timeouts, hooks |
| [Implementation Requirements](./implementation-requirements.md) | Unit, integration (I1–I19), and security (S1–S14) test suites; spec compliance rules |
| [Invariants and Prohibitions](./invariants-and-prohibitions.md) | The numbered guarantees every implementation upholds and the things no SDK ever does |
| [Decision Records](./decision-records.md) | D1–D17: why the contract is the way it is |
| [Follow-Up Work](./follow-up-work.md) | F1–F7: deferred scope and resolved findings |

## How to Read This Spec

Pick the path that matches what you came for:

| You want to… | Read |
|--------------|------|
| Build a new SDK from scratch | [Principles](./principles.md) → [Operations](./operations.md) → [Transport Contract](./transport.md) → [PaymentInstance Contract](./payment-instance.md) → [Types and Events](./types.md) → [Implementation Requirements](./implementation-requirements.md) |
| Wire up raw backend calls (no SDK yet) | [Transport Contract](./transport.md), especially [Action Payloads](./transport.md#action-payloads) |
| Understand *why* something is the way it is | [Decision Records](./decision-records.md) (D1–D17) |
| Audit or review an implementation | [Invariants and Prohibitions](./invariants-and-prohibitions.md) and the test suites in [Implementation Requirements](./implementation-requirements.md) |
| Check what is intentionally not done yet | [Follow-Up Work](./follow-up-work.md) (F1–F7) |

Normative language: **MUST**/**MUST NOT** are hard requirements verified by the
canonical test suites; everything else is contract description. Decision records
are rationale, not requirements — an implementation is judged against Operations,
Transport, Invariants, and Prohibitions.

## Implementations

| Language | Status | Package / Repository |
|----------|--------|----------------------|
| TypeScript | **Available** — reference implementation | [`@nile-squad/nylonpay-ts`](https://github.com/nile-squad/nylonpay-ts) |
| C# | In progress | — |
| Python | Planned | — |
| Go | Planned | — |
| Rust | Planned | — |
| PHP | Planned | — |
| Java | Planned | — |
| Kotlin | Planned | — |
| Elixir | Planned | — |

Want to implement one? Follow the reading path above, keep the API surface and
behavior identical to this spec (only casing conventions change per language —
`collectPayment` vs `collect_payment`), and ship the canonical test suites from
[Implementation Requirements](./implementation-requirements.md). If the spec and an
implementation disagree, the spec wins: the spec updates first, implementations
follow.
