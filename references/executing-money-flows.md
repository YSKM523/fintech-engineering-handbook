# Executing money flows

A money operation is rarely a single write. It spans steps, concurrency, and failure, and
must stay correct — never inventing or losing money — through all of it. These patterns
keep a single flow correct, from the invariants it must preserve to surviving a crash
mid-way.

## Invariants

Special properties that must always hold. The accounting equation is one; your business
stakeholders will define many more. Three complementary ways to enforce them:

1. **By construction.** Allow only valid objects, so invalid states are unrepresentable —
   factory methods (smart constructors), type-level programming (refined types), database
   constraints.
2. **Runtime checks.** Verify invariants while executing logic — assertions in production
   or tests. **Property-based testing** shines here ("for any sequence of postings, the
   books balance").
3. **Post-factum.** Analyse persisted data for violations — reconciliation jobs, nightly
   checks that ledger balances still satisfy the accounting equation.

These are **complementary**; use all three side by side. By construction is strongest but
can't express everything (especially cross-aggregate or cross-system invariants); runtime
checks catch violations at the point of occurrence; post-factum is the only one that
catches bugs that already shipped — but catches them late.

**Principles touched** — *No trust:* invariants are verified, not assumed; even your own
code's output gets checked.

## Funds reservation (hold-and-release)

Transactions usually interact with the external world (a compliance check before a
withdrawal, registering it in an external system). You must avoid race conditions —
spending the same money twice, or discovering "insufficient balance" only *after* the
external interaction already happened.

**Funds reservation** (hold-and-release): reserve funds for a transaction *before* the
external interaction starts. When it completes, settle the reservation and proceed; if
anything goes wrong, release it and the funds return to the available balance.

This introduces two balances:

- **Total balance** — everything the user owns, including reserved funds.
- **Available balance** — `available = total − reserved`.

Balance checks and new reservations are made against **available**, which is what prevents
the same funds from backing two transactions.

Practical notes:

1. **The final amount may differ.** It's not always known upfront (fees/rates differ from
   the estimate). Reserve the estimate, settle the actual amount, release the remainder.
2. **A reservation must always resolve.** One never settled nor released locks user funds,
   so every flow that creates one must guarantee it eventually resolves it. An explicit
   expiry/timeout is a useful safety net but optional — you can rely on internal
   discipline. The failure mode is conservative: an orphaned reservation *locks* money, it
   never loses or creates it.
3. **It needs strong consistency.** Checking the balance and recording the reservation
   must be **linearizable**. On a stale read, two transactions can both pass the check and
   back their spends with the same funds. No eventual consistency here.

**Principles touched** — *No invented data:* the same funds can never back two
transactions; a reservation makes this explicit instead of relying on a racy balance check.

## Handling overdrafts

An overdraft is a negative account balance. Two kinds:

1. **Intentional.** A credit product the business explicitly offers, with limits and
   interest. A feature, not an anomaly — mostly out of scope. Usually modeled as a
   separate overdraft account (liability for the user, receivable for the operator) with a
   positive balance.
2. **Unintentional.** The balance goes negative even though policy forbids it.

Unintentional overdrafts happen even in correct systems, because the external world
doesn't ask permission: a settlement comes in higher than the reserved estimate, or a
reversal lands after the funds already left. Funds reservation **reduces** the window but
can't eliminate it.

**Forbidden is not the same as unrepresentable.** It's tempting to encode "balance is never
negative" at the type/storage level (unsigned int, `CHECK (balance >= 0)`). But when you're
*forced* to accept a negative balance, a system that can't represent it will crash mid-flow,
silently clamp to zero (**inventing money**), or do something similarly wrong.

So treat "balance ≥ 0" as an invariant: enforce it at runtime when authorizing
transactions, detect violations post-factum with monitoring and reconciliation — but don't
force it by construction. A detected overdraft is a signal to investigate, not necessarily
a bug.

When an overdraft does happen, **book it and recover explicitly** — net it against future
deposits, request repayment, or write it off as an explicit compensating entry to an
expense/loss account.

**Principles touched** — *No invented data:* clamping a negative balance to zero mints
money. *No trust:* the external world can force an overdraft no matter what your checks say.

## Idempotency

In a distributed system you can't guarantee exactly-once delivery — any call can be
interrupted and you won't know whether it arrived. To ensure delivery you must **retry**;
in doing so you risk delivering more than once, so processing must be **idempotent** — the
same message delivered twice triggers processing once.

1. **Prefer explicit keys.** Idempotency keys beat business-derived idempotency
   (deduping on the payload), which is fragile — hard to tell whether two transactions with
   the same amount are a duplicate or two genuine operations. Scope keys to a specific
   operation and client.
2. **Decide how errors replay.** When a call failed the first time, does a retry re-raise
   the stored error or re-trigger processing? Usually simplest to treat the error as the
   idempotent result and replay it (the client can retry with a new key). It depends on
   the error: permanent ones (validation) replay as-is; temporary ones (network) might be
   reprocessed.
3. **Validating the repeated payload.** Good practice to ensure a repeat carries the same
   payload — but it gets costly and buys little, at the cost of complexity and flexibility
   (the caller might legitimately change the request).
4. **At scale it's hard.** You may dedupe billions of requests *and* get the behavior right
   under concurrent access (two duplicates in the same millisecond). The idempotency
   barrier must be **atomic**.
5. **Beware time windows.** Deduping only within, say, 24h simplifies implementation
   (otherwise data grows forever) but costs correctness. Make this tradeoff only if you
   absolutely must — it will haunt you.
6. **Test for retries.** Bake a generic middleware into integration/system tests that
   automatically repeats every call.
7. **Handle out-of-order retries.** Stay idempotent even after moving to a new state — e.g.
   keep "put funds on hold" idempotent even if they were already released.

Idempotency matters on **both sides** — when you make calls and when you receive them.
Keep it in mind every time you consume or expose an operation.

**Principles touched** — *No invented data:* retries are unavoidable, so processing must
collapse duplicate deliveries into a single effect instead of moving money twice.

## Full resumability

A money flow rarely happens in one step. A withdrawal might reserve funds, run compliance
checks, register the operation externally, and finally settle. The sequence is stretched
across time and can die between any two steps — so **assume failure every two steps**. A
flow can never assume it runs to completion in one go, and a half-finished one must always
land in a **recoverable** state, never an inconsistent one.

1. **Persist progress, don't keep it in memory.** Model the flow as an explicit state
   machine whose state is durably stored, and commit each step's completion before
   starting the next. A restart must tell exactly where the flow was.
2. **Something must resume stalled flows.** An independent driver (scheduler, worker,
   poller) picks up incomplete flows and pushes them forward. A crash of the orchestrator
   must not strand a flow forever.
3. **Every step must be safe to re-run.** On resume you may re-execute a partially-happened
   step, so each must be idempotent.
4. **Roll forward or compensate.** External effects can't be rolled back — once you've
   called the outside world, a database rollback won't undo it. So either retry forward
   until completion, or, when a later step fails for good, post compensating actions to
   unwind earlier ones (the **saga** pattern).

Use a durable-execution engine (Temporal, Camunda, Workflows4s, AWS Step Functions) or
hand-roll a persistent state machine.

**Principles touched** — *No lost data:* a crash mid-flow must never lose track of in-flight
money; persisted progress lets the flow be picked up and finished. *No invented data:*
resuming re-runs steps, so they must re-apply without double-counting — the flow completes
exactly once.
