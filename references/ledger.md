# Recording money: the ledger

Once represented, money movements must be recorded so they balance, survive audit, and can
be reconstructed years later. This is where the books, their timestamps, and their history
live.

## Double-entry bookkeeping

Store financial transactions as a list of entries of the form
`(credit account, debit account, amount)` (a compact form; the classic representation uses
a separate debit and credit row per movement). Because every entry moves the same amount
out of one account and into another, the books always balance — money is only moved, never
created or destroyed.

- **Money always has a source and a destination.** External providers get dedicated
  accounts too, so money entering/leaving the system is still tracked.
- **Balance is derived, not authoritative.** The entries are the source of truth; any
  balance is a projection *computed* from them. You may store that projection (cache,
  snapshot) for performance — what you must never do is treat a stored balance as the truth
  and mutate it directly (see *Stored balances are projections* below).
- **Accounts have a type** — asset / liability / equity — so the **accounting equation**
  (`assets = liabilities + equity`) holds and each account has a defined side on which it
  increases. In practice you also need income (revenue) and expense accounts, e.g. to book
  a fee as revenue or a write-off as a loss
  (`assets = liabilities + equity + revenue − expenses`).
- **One transaction, many movements.** A single transaction usually creates several
  movements, e.g. one for the net amount and another for the fees.
- **Posted entries are immutable.** By convention, corrections are made by adding new
  compensating entries that offset the original.

**Principles touched** — *No invented data:* money only ever moves between accounts; the
total is conserved.

### Stored balances are projections, not the source of truth

"Balance is derived from movements" is a statement about *authority*, not a ban on balance
tables. In any real ledger system balances get stored constantly — the question is *how*,
not *whether*. Writing "never store a balance" into a design is over-simplified to the point
of being bad advice.

- **A balance is a projection over entries.** It can be cached, snapshotted, or
  materialized for performance; the entries remain the single source of truth it derives
  from (this is the event-sourcing relationship — see Event sourcing).
- **There isn't one balance — there are several.** Production systems carry multiple
  distinct projections over the same entries: **posted/total**, **available**
  (`total − reserved`), **reserved/hold**, **pending vs cleared**, **settled**, and
  **as-of-date / reporting** balances. Each answers a different question, and conflating
  them is its own class of bug (see Funds reservation for total vs available).
- **Any stored balance must be recomputable, reconcilable, explainable, and
  drift-detectable.** You must be able to (1) recompute it from the entries, (2) reconcile
  the stored value against that recomputation on a schedule, (3) explain it by pointing to
  the entries that produced it, and (4) detect and alert on drift when the two disagree —
  the post-factum check applied to balances (see Invariants and Reconciliation).
- **What's actually forbidden** is a balance that is its *own* source of truth: an
  `UPDATE accounts SET balance = balance + :x` with no entry behind it. That mutation can
  invent or lose money, can't be audited, and drifts silently. Every change to a balance
  must be the *consequence* of a posting, never a write in its own right.

In short: don't ban the cache, ban the *uncomputable* balance.

**Principles touched** — *No invented data:* a directly-mutated balance can mint or lose
money with no entry to account for it. *No trust:* a stored balance is verified against the
entries (reconciliation, drift detection), never assumed correct.

## Value time vs booking time vs settlement time

Transactions usually carry two, sometimes three timestamps:

1. **Value time** — when the transaction occurred.
2. **Booking time** — when it was recorded in the system.
3. **Settlement time** — when money was actually transferred/materialized. Not every
   transaction has one. Usually expressed as **T+X** (X days after value; T+2 = 2 days
   after value).

Value and booking time almost always diverge:

- **Backdated** (booking > value). Technically almost everything is backdated, but the term
  matters most when booking and value fall in different reporting periods (days, months,
  years).
- **Forward-dated** (booking < value). Less frequent — scheduled/future-dated payments, a
  standing order recorded today but effective next week.

Example: a card payment happens at T1 (value), you record it at T2 (booking), the provider
transfers money to your account at T3 (settlement).

Business and business-consumed reports usually care about value or settlement time;
booking time is useful for traceability.

**Principles touched** — *No lost data:* record every relevant timestamp; collapsing them
into a single `created_at` loses information you can't reconstruct later.

## Audits and audit trails

Financial systems face regulatory audits. Things an audit might verify:

- Are company funds **not commingled** with user funds or used for company expenses?
- Are all revenues registered, reported, and explainable — can you pinpoint the
  transactions behind a revenue stream in a period?
- Does information given to the outside world (users, the tax office) match reality — do
  you hold as much in assets as you owe users?
- Are funds protected against external threats — who can access them and how?

To answer these, keep not just current state but the **full history** of how that state
came to be. That history is the **audit trail**: a record of everything that happened,
detailed enough that any balance, report, or decision can be explained and reproduced.

For every change, capture:

- **What** happened.
- **When** it happened (see value vs booking time).
- **Who or what** triggered it — a user, an operator, an automated job.
- **Why** it happened — a reference to the order, instruction, or incident that caused it.

Money movements are the obvious subject, but **manual interventions, configuration changes**
(fee schedules, rate sources, limits) and **permission changes** need trails too.

The **why** is often itself the output of a decision (a compliance check, a risk score).
Recording just the outcome ("blocked") rarely satisfies an audit — you'll be asked *how*
it was reached. If that logic lives in a decision table or rules engine (DMN, Drools,
Decisions4s) rather than buried in imperative code, the decision becomes a structured,
replayable artifact: which rules fired, on which inputs, with what result.

**Principles touched** — *No lost data:* current state alone can't answer an audit's
questions; only the full history can.

### Event sourcing

The most principled, systemic way to build an audit trail. Instead of storing current
state with a log next to it, store **only the events** and derive state from them. The
double-entry ledger is this pattern applied to money — the balance isn't the stored
*truth*, it's computed from the entries (and may still be cached or snapshotted, see
*Stored balances are projections*). The trail is then a *primary artifact* and cannot drift
from reality.

Practical notes:

1. **You don't need it everywhere.** The ledger already covers money; for surrounding
   domains a conventional model with a reliable change log may suffice.
2. **Derived state can be cached.** Balances/projections can be cached or snapshotted.
3. **Projections are work-intensive.** You may need many, and you can't effectively query
   the raw event set, so you build dedicated/generic projections to look into your data.
4. **Plan for schema evolution.** Events live for years; today's code must still read
   events written long ago.

Event sourcing is excellent *when an audit trail is required*, but it carries a very high
system-complexity price. Don't reach for it reflexively.

**Principles touched** — *No lost data:* when state is derived from events, the trail
can't drift out of sync with reality because it *is* the source of truth.

### Immutability

An audit trail that can be edited proves nothing — records can never be updated or
deleted. The log must be **append-only**; every correction is a new record.

Immutability is an invariant; the usual toolbox applies:

1. **By construction.** Append-only tables; revoke `UPDATE`/`DELETE` at the
   database-permission level.
2. **Runtime checks.** The application exposes no mutating operations on posted records.
3. **Post-factum.** Tamper evidence — checksums or hash chains over records, periodically
   verified, so any after-the-fact modification is detectable.

Real systems have bugs that may require fixing the log. Sometimes it's easier to update
the trail in place than to keep it strictly immutable. The balance comes from your
**reporting schedule and obligations**: usually data must be set in stone only once it's
been reported (e.g. the month-end financial statement). Until then you may still fix
problems in place — if you catch them before the data leaves your system.

**Principles touched** — *No trust:* an editable history proves nothing; immutability and
tamper evidence make the trail trustworthy to an outsider — including yourself
investigating an incident.

### Reversals and corrections

Mistakes still happen — a wrong amount posted, a transaction on the wrong account.
Immutability means **fixing forward**: post a new compensating entry and link it to the
record it corrects, in both directions.

1. **Reversal.** Negates the original in full, as if it never happened economically — but
   it stays visible in history alongside the original.
2. **Correction (adjustment).** Books the difference between what was recorded and what
   should have been, or reverses and re-posts with the right values.
3. **Mind the reporting period.** Corrections often land in a different period than the
   original; the linkage lets reports attribute them correctly and distinguish real
   activity from cleanup.

When posting corrections/reversals, decide whether to **backdate** (a value time in the
past) or not. This depends on the reporting schedule again — usually you can't backdate
anything into an already-closed period, because it was already reported.

**Principles touched** — *No invented data:* mistakes are fixed by posting linked
compensating entries that offset the original record.

### Immutability vs GDPR

GDPR's right to erasure appears to contradict an immutable ledger. In practice it's easy to
make a non-problem:

1. **Financial records are largely exempt.** Legal retention obligations (accounting law,
   AML — typically 5–10 years) take precedence over erasure for transactional data. You
   don't delete postings in that window.
2. **Separate PII from financial data.** The exemption covers what you're *obliged* to
   keep. The immutable ledger references users only by opaque internal identifiers, while
   PII (names, addresses, documents) lives in a separate, mutable store that can be
   redacted/erased independently.
3. **Crypto-shredding for embedded PII.** Where personal data must be embedded in
   immutable records (e.g. event payloads), encrypt each user's personal fields with a
   per-user key and erase by deleting the key. Erasure becomes a key deletion, not a
   rewrite of history.

**Principles touched** — *No lost data:* separating PII from financial data lets you honor
erasure without losing the financial history you're obliged to keep.
