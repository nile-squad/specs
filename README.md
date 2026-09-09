# Nile Squad Specs

Canonical, language-agnostic specifications for Nile Squad products. This is the
source of truth a developer reads to implement an official SDK or integration in
**any** language, the published packages are reference implementations of what
lives here.

Each spec defines the API surface, types, behaviors, transport contract, and
invariants that every implementation must follow. If a spec and an
implementation disagree, the spec wins; the spec is updated first, then
implementations follow.

## Specs

| Product | Spec | What it is | Reference implementation |
|---------|------|------------|--------------------------|
| **Nylon Pay** | [`nylonpay-sdk-spec/`](./nylonpay-sdk-spec/spec.md) | Server-side SDK for collecting payments, payouts, phone verification, invoices, transaction status, and webhook verification over a signed, action-based transport. | [nylonpay-ts](https://github.com/nile-squad/nylonpay-ts) (TypeScript) |

## Implementing an SDK from a spec

1. **Start with the [Build Guide](./nylonpay-sdk-spec/build-guide.md)** — the
   recipe that lists the components and the order to build them in, with a
   verify step at the end of each stage.
2. Read the spec end to end, principles, decision records, operations, types,
   transport contract, security, invariants, and prohibitions.
3. Match names, shapes, events, and status values exactly. Only casing adapts to
   each language's conventions (`collectPayment` / `collect_payment` /
   `CollectPayment`).
4. Use the reference implementation to resolve ambiguity, then mirror its public
   surface, not its internal structure.
5. Ship the test suite the spec requires (signing, response verification,
   lifecycle, retries, webhook verification, and the listed edge cases).

## Versioning

Specs are versioned independently (see the `Version` field in each spec, which
points at the spec's [changelog](./nylonpay-sdk-spec/changelog.md)). Breaking
changes bump the major version and are recorded there.
