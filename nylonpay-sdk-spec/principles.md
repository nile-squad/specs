# Principles

Part of the [Nylon Pay SDK Spec](./spec.md).

## Functional core, idiomatic shell

Every SDK exposes a factory function that returns an object/struct of functions.
Prefer functional patterns (plain objects, closures, composition) in every
language. Classes are permitted only when the language lacks idiomatic functional
alternatives. The end-user DX must remain identical regardless of internal
implementation.

## Same names, same shapes, same events

Function names, parameter names, event names, and status values are consistent
across all implementations. A merchant reading TypeScript docs can write Python
code without relearning the API. Only casing conventions differ per language
(`collectPayment` vs `collect_payment` vs `CollectPayment`).

## Named parameters everywhere

Every operation takes a single configuration object/struct/dict. No positional
arguments beyond the factory. This makes the API self-documenting and resistant
to parameter ordering bugs.

## Signing is invisible

HMAC-SHA256 request signing, nonce generation, fingerprinting, and response
verification happen automatically inside the transport layer. The merchant never
touches crypto primitives.

## Async operations return a PaymentInstance

Long-running operations (collections, payouts) return an event-driven instance
with pubsub (`on`/`once`/`off`) and a blocking resolver (`wait`). The instance
owns its polling lifecycle and stops on terminal states.

## Fail loudly on misconfiguration, gracefully on runtime errors

A missing or malformed API key/secret throws at initialization. Client-side
validation errors also throw (programmer mistakes). When an async initiation
(`collectPayment`/`makePayout`) is rejected on the **server** (invalid key, bad
signature, scope/limit/provider reject), the operation returns a PaymentInstance
that emits an `error` event carrying a structured `category`. The transaction
never started, so there is nothing to poll. Sync and blocking operations return
error results. The boundary: programmer mistakes throw; operational failures
surface as results or events. See
[D13](./decision-records.md#d13-server-side-initiation-failures-emit-error-event).

## Idempotency rides on the reference

The `reference` is the transaction identity and the idempotency key. There is no
separate `idempotencyKey` input (D18). Same reference = same transaction: the SDK
replays the existing transaction (`duplicate: true`) instead of charging again.
A fresh reference always starts a fresh payment. When omitted, the SDK generates
the reference (a UUID) automatically. On a retry, the reference MUST be reused
and the request re-signed.

## Stateless between calls

The SDK holds no session, connection pool, or persistent state between
operations. The only stateful construct is the PaymentInstance, which is
explicitly scoped to one transaction and disposable.

## Server-side only

The SDK runs in server environments. It uses OS-level fingerprinting and
server-grade crypto primitives.

## Spec is the ceiling

No SDK implementation adds operations, parameters, events, or behavior beyond
what this spec defines. If a feature is needed, the spec updates first, then
implementations follow.