# Nylon Pay SDK Spec

**Version:** 2.1.0. See the [Changelog](./changelog.md) for version history.

> Canonical, language-agnostic specification for the Nylon Pay SDK. Implement it
> in any language; the [TypeScript SDK](https://github.com/nile-squad/nylonpay-ts)
> is the reference implementation.

## Purpose

The Nylon Pay SDK is the merchant's programmatic interface to the payment platform. It provides a consistent API across multiple languages for collecting payments, making payouts, verifying phones, creating invoices, checking transaction status, and verifying webhooks. The SDK abstracts the transport protocol, HMAC signing, polling mechanics, and error handling so merchants interact with payment operations, not infrastructure.

All SDKs are server-side. Client-side packages (browser, mobile) are a future scope.

## Start Here: The Build Guide

**Implementing a new SDK? Read the [Build Guide](./build-guide.md) first.** It is
the recipe: it lists the components you build overall, then walks you through
them in dependency order: signing core, types, validation, transport, factory,
operations, polling, webhooks, with a **Verify** gate closing each step, so
you prove each piece before wiring the next. The rest of this spec is the
reference that the guide points at; the guide is the order to build it in.

## Documents

The spec is split into focused documents. This page is the entry point; each
document is self-contained for its topic and links back here.

| Document | What it covers |
|----------|----------------|
| [Build Guide](./build-guide.md) | **Start here when implementing:** the components and the build order, step by step |
| [Principles](./principles.md) | The ten design principles every SDK follows |
| [Operations](./operations.md) | Every operation: inputs, outputs, reference constraints, hosted invoice behavior |
| [PaymentInstance Contract](./payment-instance.md) | The event-driven instance returned by async operations: events, polling lifecycle, `wait()` |
| [Types and Events](./types.md) | Type definitions, the Transaction shape, and the webhook event catalog |
| [Transport Contract](./transport.md) | Endpoint, request envelope, per-action payload validation, a worked request/response example, retry policy, and status polling |
| [Security](./security.md) | Signing protocol (canonical payload, request headers, `_fingerprint`, conformance vectors, server-side checks), response verification and size bounds, webhook integrity, secret handling |
| [Error Categories](./errors.md) | The fixed error taxonomy and how categories travel on the wire |
| [Configuration](./configuration.md) | Factory configuration: keys, base URL, timeouts, hooks |
| [Implementation Requirements](./implementation-requirements.md) | Unit, integration (I1–I19), and security (S1–S21) test suites; spec compliance rules |
| [Invariants and Prohibitions](./invariants-and-prohibitions.md) | The numbered guarantees every implementation upholds and the things no SDK ever does |
| [Decision Records](./decision-records.md) | D1–D21: why the contract is the way it is |
| [Follow-Up Work](./follow-up-work.md) | F1–F5: deferred scope |
| [Changelog](./changelog.md) | Version history and the current spec version |

## How to Read This Spec

Pick the path that matches what you came for:

| You want to… | Read |
|--------------|------|
| Build a new SDK from scratch | [Build Guide](./build-guide.md) (the recipe, in order), then the reference docs it points to |
| Wire up raw backend calls (no SDK yet) | [Transport Contract](./transport.md) and [Security](./security.md), especially [Action Payloads](./transport.md#action-payloads) and [Request Signing](./security.md#request-signing) |
| Understand *why* something is the way it is | [Decision Records](./decision-records.md) (D1–D21) |
| Audit or review an implementation | [Invariants and Prohibitions](./invariants-and-prohibitions.md) and the test suites in [Implementation Requirements](./implementation-requirements.md) |
| Check what is intentionally not done yet | [Follow-Up Work](./follow-up-work.md) (F1–F5) |
| Follow the spec's history or current version | [Changelog](./changelog.md) |

Normative language: **MUST**/**MUST NOT** are hard requirements verified by the
canonical test suites; everything else is contract description. Decision records
are rationale, not requirements, an implementation is judged against Operations,
Transport, Invariants, and Prohibitions.

## Implementations

| Language | Status | Package / Repository |
|----------|--------|----------------------|
| TypeScript | **Available:** reference implementation | [`@nile-squad/nylonpay-ts`](https://github.com/nile-squad/nylonpay-ts) |
| C# | In progress | n/a |
| Python | **Available** | [`nylonpay-py`](https://github.com/nile-squad/nylonpay-py) |
| Go | Planned | n/a |
| Rust | Planned | n/a |
| PHP | **Available** (alpha) | [`nile-squad/nylonpay-php`](https://github.com/nile-squad/nylonpay-php) |
| Java | Planned | n/a |
| Kotlin | Planned | n/a |
| Elixir | Planned | n/a |

Want to implement one? Follow the reading path above, keep the API surface and
behavior identical to this spec (only casing conventions change per language, 
`collectPayment` vs `collect_payment`), and ship the canonical test suites from
[Implementation Requirements](./implementation-requirements.md). If the spec and an
implementation disagree, the spec wins: the spec updates first, implementations
follow.
