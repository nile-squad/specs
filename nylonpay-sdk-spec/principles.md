# Principles

Part of the [Nylon Pay SDK Spec](./spec.md).

1. **Functional core, idiomatic shell.** Every SDK exposes a factory function that returns an object/struct of functions. Prefer functional patterns (plain objects, closures, composition) in every language. Classes are permitted only when the language lacks idiomatic functional alternatives — the end-user DX must remain identical regardless of internal implementation.

2. **Same names, same shapes, same events.** Function names, parameter names, event names, and status values are consistent across all implementations. A merchant reading TypeScript docs can write Python code without relearning the API. Only casing conventions differ per language (`collectPayment` vs `collect_payment` vs `CollectPayment`).

3. **Named parameters everywhere.** Every operation takes a single configuration object/struct/dict. No positional arguments beyond the factory. This makes the API self-documenting and resistant to parameter ordering bugs.

4. **Signing is invisible.** HMAC-SHA256 request signing, nonce generation, fingerprinting, and response verification happen automatically inside the transport layer. The merchant never touches crypto primitives.

5. **Async operations return a PaymentInstance.** Long-running operations (collections, payouts) return an event-driven instance with pubsub (`on`/`once`/`off`) and a blocking resolver (`wait`). The instance owns its polling lifecycle and stops on terminal states.

6. **Fail loudly on misconfiguration, gracefully on runtime errors.** A missing or malformed API key/secret throws at initialization. Async initiation failures (`collectPayment`/`makePayout` rejected before a transaction exists — invalid key, bad signature, scope/limit/provider reject) also throw, carrying a structured `category`. Sync and blocking operations return error results. The boundary is: there is no transaction to observe yet → throw; otherwise → result. See [D13](./decision-records.md#d13-server-side-initiation-failures-emit-error-event).

7. **Idempotency is automatic but overridable.** Every payment operation generates a unique idempotency key by default. Merchants supply their own for retry-safe operations. The SDK never sends the same key twice unintentionally.

8. **Stateless between calls.** The SDK holds no session, connection pool, or persistent state between operations. The only stateful construct is the PaymentInstance, which is explicitly scoped to one transaction and disposable.

9. **Server-side only.** The SDK runs in server environments. It uses OS-level fingerprinting and server-grade crypto primitives.

10. **Spec is the ceiling.** No SDK implementation adds operations, parameters, events, or behavior beyond what this spec defines. If a feature is needed, the spec updates first — then implementations follow.

