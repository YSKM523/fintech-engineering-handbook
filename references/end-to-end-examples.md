# End-to-end examples

The handbook takes each pattern on its own, which makes it hard to build holistic
intuition. These three flows are simplified but representative examples of what you'll see
in a real system (production would have more steps, failure branches, and bookkeeping).
They cover the three directions money moves: **out** of the system (a withdrawal), **into**
it (a card deposit), and **within** it (a conversion). Read this after the layer-specific
references to see how the layers interlock.

## Flow 1: A crypto withdrawal (money out)

A user asks to withdraw 0.5 ETH to an external address. The richest of the three, because
money is leaving through an **irreversible external effect** — once the chain confirms the
send, there's no taking it back.

1. **The request arrives with an idempotency key.** The client may retry the submit (the
   network ate the response; the user double-clicked), so the first concern is that two
   deliveries create *one* withdrawal, not two. The key is scoped to this user and this
   operation. → *idempotency*
2. **Funds are reserved before anything irreversible happens.** Reserve 0.5 ETH plus an
   estimated network fee against the user's *available* balance. The balance check and the
   reservation are a single **linearizable** step — otherwise two concurrent withdrawals
   could both pass and back their spends with the same coins. → *funds reservation,
   invariants*
3. **The compliance gate runs — and the flow may sleep here for days.** Before
   broadcasting, screen the transaction (sanctions, AML, the destination address). Several
   patterns tie together:
   - It's an **external call**, so it will be slow, fail, or lie — build defensively. →
     *consuming APIs*
   - It may escalate to **manual review** taking hours or days, so the flow must survive
     dying between this step and the next: its state is persisted, an independent driver
     resumes it, and the reservation simply stays in place. → *full resumability*
   - A **daily withdrawal limit** is enforced here — nothing more than an invariant. Being
     a time-windowed, stateful invariant evaluated under concurrency, it has the same
     atomicity problem as the reservation: two withdrawals racing past the limit must not
     both pass. → *invariants*
   - Every decision (passed, blocked, who overrode it) lands in the **audit trail**.
4. **The transaction is broadcast on-chain.** Once cleared, sign and broadcast through a
   node — another external call with all the same caveats. This step must be **idempotent**:
   a resume after a crash must *re-check the chain*, not blindly broadcast a second time.
   The real network fee isn't known upfront — which is exactly why you reserved an
   *estimate*; you'll settle the actual amount and release the remainder.
5. **Wait for finality, then post to the ledger.** A single confirmation isn't the end — a
   **reorg** can undo a send declared "done" too early, so wait for enough confirmations.
   Only then post the double-entry movements: debit the user's account, credit the external
   on-chain account (the outside world gets an account too), book the network fee against an
   expense account and your service fee against a revenue account. → *double-entry*
6. **A nightly job reconciles against the chain.** Independently, a job compares the ledger
   with on-chain reality and the node's view of your transactions. It's the safety net that
   catches a broadcast that never confirmed, or a fee that came out different from what you
   booked. → *reconciliation*

**Where it gets interesting:** if the actual network fee exceeds the reserved estimate, the
settlement can push the account negative — book the **overdraft** and recover it explicitly.
And if the process crashes between broadcast (step 4) and confirmation (step 5),
resumability plus idempotency let it pick up by querying the chain rather than sending the
funds twice.

## Flow 2: A card deposit (money in)

A user tops up via a card payment through a PSP. Money comes *in*, which shifts the hard
part from "don't send twice" to **"don't trust what the outside world tells you, and don't
credit money that hasn't really arrived."**

1. **The user initiates the deposit.** They enter an amount and submit card details; you
   open a deposit transaction with the PSP — again with an **idempotency key**, since the
   submit can be retried.
2. **Authorization places a hold.** The PSP authorizes the card, placing a hold without yet
   capturing the money — the card world's version of **funds reservation** (authorization
   vs capture). You do not credit the balance yet; the money is not yours.
3. **A webhook says "captured" — and you believe none of it.** The PSP calls your webhook
   endpoint. Verify the signature over the **raw bytes**, persist the raw payload,
   acknowledge fast with a 2xx, and only then process asynchronously. Treat the webhook as a
   *hint that something happened*, not as truth: **query the PSP's API for authoritative
   state**, because webhooks arrive out of order, carry stale data, get duplicated, and get
   lost. Processing is **idempotent**, so a re-delivered "captured" credits the user once. →
   *handling webhooks, idempotency*
4. **The credit goes through a clearing account.** The money is in flight (**float**) —
   captured by the PSP but not yet settled to your bank — so post it through a
   suspense/clearing account rather than pretending it's arrived: credit the user's balance
   (a liability — their balance is an IOU from you to them) and debit a PSP receivable. The
   interchange/processing fee is booked as an expense. Booking time is now; settlement time
   is later, T+X. → *value/booking/settlement time, double-entry*
5. **Settlement arrives as a batch, and reconciliation is one-to-many.** Days later the PSP
   settles a single transfer covering many deposits at once. Reconcile that batch against
   the clearing account, matching one settlement against many transactions. The T+X delay is
   baked in so you don't alert on transactions simply not settled *yet* — and the same job
   catches a webhook that never arrived. → *reconciliation*
6. **Weeks later, a chargeback.** The cardholder disputes and their bank forces a reversal.
   Don't edit the original posting — post a **linked compensating entry**, which usually
   lands in a later reporting period and may push the user's balance negative if they've
   already spent the funds. → *reversals and corrections, overdrafts*

**Where it gets interesting:** the whole flow is built on *not* believing the happy-path
signal. The webhook is a trigger, not a fact; the clearing account refuses to recognize
money until it has actually moved; reconciliation verifies the PSP against your books rather
than the other way around. Every step is an application of **no trust**.

## Flow 3: An in-app conversion with cashback (money within)

A user converts 1,000 EUR into USDC and earns a small promotional cashback. Money moves
entirely *within* the system, so there's no unreliable external rail — instead this flow
stresses the **representation layer** (precision, rounding, currencies, rates) and the
sharpest form of **no invented data**.

1. **A directional quote, and a reservation.** Quote a rate for EUR→USDC. That rate is its
   own price, not the inverse of USDC→EUR — buying and selling sit on opposite sides of the
   bid/ask spread. Reserve the 1,000 EUR; the request carries an idempotency key like any
   other. → *FX rates, funds reservation*
2. **The two sides never get added together.** EUR and USDC are distinct currencies — USDC
   is identified by `(network, contract address)`, not a bare code, and is not
   interchangeable with the fiat it's pegged to. The system forbids cross-currency
   arithmetic; the only bridge between the two amounts is an explicit conversion at the
   controlled rate. → *currency handling*
3. **Full precision through the math, rounding only at the edge.** Compute the conversion
   keeping full precision and round **exactly once**, at the boundary, with a deliberately
   chosen strategy. The spread you earn is **revenue** — book it explicitly to a revenue
   account via double-entry, not allowed to disappear into a rounding residual. Don't store
   the transactional rate as a separate field; it falls out of the two amounts. A separate
   reference rate is what you'd use later to value the holding. → *precision, rounding,
   double-entry*
4. **The cashback is the hardest test of "no invented data."** Tempting to treat a bonus as
   a free balance bump — but that mints money from nothing. The cashback is real money: it
   must be **funded** — moved out of a company promotional/expense account into the user's
   balance via a proper double-entry posting — and the percentage that defines it needs the
   same explicit rounding decision as everything else.
5. **Settle, then notify reliably.** Settle the reservation, post all movements with their
   timestamps and audit trail, and publish the outcome so the rest of the system
   (statements, notifications, analytics) learns about it. That publish must be reliable
   even though it spans a separate channel — which is what the **outbox** (or CDC, or the
   event log) is for; downstream consumers dedupe on a stable event id because delivery is
   at-least-once. → *notifying reliably*

**Where it gets interesting:** the cashback and the spread pull in opposite directions on
the same posting. The spread is money the user *loses* to you (revenue); the cashback is
money you *give* the user (an expense). Both are real, both go through the books, and both
round — so the one transaction must satisfy **no invented data** (the books still balance,
nothing minted) and **no lost data** (every residual tracked) at the same time.
