# Nile Squad Specs

Canonical, language-agnostic specifications for Nile Squad products. This is the
source of truth a developer reads to implement an official SDK or integration in
**any** language — the published packages are reference implementations of what
lives here.

Each spec defines the API surface, types, behaviors, transport contract, and
invariants that every implementation must follow. If a spec and an
implementation disagree, the spec wins; the spec is updated first, then
implementations follow.

## Specs

| Product | Spec | What it is | Reference implementation |
|---------|------|------------|--------------------------|
| **Nylon Pay** | [`nylon-pay/sdk-spec.md`](./nylon-pay/sdk-spec.md) | Server-side SDK for collecting payments, payouts, phone verification, invoices, transaction status, and webhook verification over a signed, action-based transport. | [nylonpay-ts](https://github.com/nile-squad/nylonpay-ts) (TypeScript) |
| **AI** | _coming soon_ | Specification for Nile Squad AI integrations. | — |

## Implementing an SDK from a spec

1. Read the relevant spec end to end — principles, decision records, operations,
   types, transport contract, invariants, and prohibitions.
2. Match names, shapes, events, and status values exactly. Only casing adapts to
   each language's conventions (`collectPayment` / `collect_payment` /
   `CollectPayment`).
3. Use the reference implementation to resolve ambiguity, then mirror its public
   surface — not its internal structure.
4. Ship the test suite the spec requires (signing, response verification,
   lifecycle, retries, webhook verification, and the listed edge cases).

## Versioning

Specs are versioned independently (see the `Version` field in each document).
Breaking changes bump the major version and are called out in the spec's
follow-up/changelog section.
