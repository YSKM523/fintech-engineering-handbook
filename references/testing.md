# Testing

Tests matter everywhere, but in a money system they matter more. The difficulty: you
usually can't enumerate the expected outputs — the space of operation sequences is too
large, and the interesting failures live in the combinations. Treat the techniques below as
a **menu**; pick the ones with the most impact on your system.

1. **Property-based testing.** Instead of asserting specific outputs, assert that a
   property holds for any generated input. A natural fit for invariants and money math —
   the framework generates the awkward cases you wouldn't think to write by hand.
2. **Invariant checks between steps.** When you generate a sequence of operations, don't
   only assert invariants at the end — assert them after **every single step**. Impossible
   to do by hand at scale, so use a harness that automatically injects the assertions.
3. **Generative idempotency testing.** Since every operation that touches the outside world
   must be idempotent, make that a property: automatically repeat all declared operations
   and assert the second call has no impact on the system.
4. **Crash and resume injection.** Long flows must survive dying between any two steps, so
   test exactly that — **inject a failure at every step** and assert recovery.
5. **Round-trip testing.** Encode then decode, serialize then deserialize, convert then
   convert back — and assert you land where you started (or within a known tolerance). A
   quick way to catch precision loss at boundaries and serialization bugs in money/currency
   types. Plays well with automatic data generation.
6. **Golden testing.** Pin the output of a calculation or projection (a fee breakdown, a
   statement, a report) to a stored expected result, so any unintended change shows up as a
   diff. Useful for gnarly, hard-to-reason-about computations where you trust a
   reviewed-once result more than a freshly written assertion.
7. **Backward-compatibility testing.** Events and stored records live for years; today's
   code must still read what old code wrote. Keep a corpus of real, old-format payloads and
   assert current code still deserializes and projects them correctly — this is what stops
   a schema change from silently breaking history.
8. **Testing in production.** Some confidence is only obtainable against the real thing —
   provider sandboxes diverge significantly from production. The final proof an integration
   works often has to happen live: a canary release, a controlled small-blast-radius
   rollout, or synthetic transactions pushing small real amounts through continuously as a
   health check. The money-specific caveat: these are **real movements**. A test in
   production moves real money, so it must go through the same ledger, reconciliation, and
   audit trail as everything else, be clearly tagged, and be cleaned up through the normal
   correction/reversal machinery — never a backdoor that bypasses the books.

**Principles touched** — *No trust:* tests verify the patterns actually hold rather than
assuming they do; the invariant is the oracle, not a value you happened to expect.
*No invented data:* replaying operations and injecting failures proves retries and recovery
don't double-count or mint money. *No lost data:* round-trip and backward-compatibility
tests prove precision and history survive boundaries and the passage of time.
