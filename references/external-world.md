# The external world

Interacting with the external world — third-party providers (payments, KYC, AML, banks,
custodians) or internal services — is unavoidable. The job is to stay correct regardless of
how unreliable those dependencies become.

## Consuming APIs

You don't control someone else's API — its code, quality, or uptime — so the safe default
is to assume it will misbehave and build defensively.

1. **Don't trust the schema.** Responses won't always match the contract: fields go
   missing, types change, nulls appear where they shouldn't. **Validate everything you act
   on or trust** — the amount, the currency, a status you branch on, anything
   security-relevant — and fail loudly on anything unexpected, so malformed data can't leak
   in. Conversely, **don't validate fields you merely pass through or ignore**: validating
   cosmetic data you never use just hands a third party the power to cause you a needless
   outage when it breaks its own contract (and it will). The test is "do I act on this?",
   not "is it in the schema?".
2. **Expect imperfect engineering.** Given time, every questionable practice appears:
   tokens in URLs, lost precision, HTTP codes that lie (a `200` carrying an error body),
   inconsistent pagination, custom date formats. Treat it as the job, not the exception.
3. **All calls will fail.** Design to handle a missing response — retries and timeouts are
   necessary protection. But a retry is only safe on an **idempotent** operation, and only
   with bounded backoff + jitter and a give-up threshold; retrying a non-idempotent call, or
   retrying tightly with no ceiling, amplifies an outage and can double-move money (see
   Idempotency).
4. **Timeouts and bounded concurrency are mandatory; the circuit breaker is the
   escalation.** The non-negotiable defaults on every external call are a timeout and a
   concurrency cap (a bulkhead) — without them a single slow dependency drains your threads
   and connections. A breaker on top is *required*, not optional, when the call sits on a
   synchronous user/critical path, or shares a finite resource pool a slow dependency can
   exhaust. It's genuinely optional only for isolated async/background callers with their
   own resources — there, shedding load is reasonably the server's job. Don't skip the
   timeout + bulkhead either way.
5. **Mind the quotas.** Rate limits and usage quotas are easy to forget and cause nasty
   weekend outages. Do napkin math up front (expected volume vs the provider's limits).
6. **Store every request and response.** Sounds excessive; it's a lifesaver during an
   investigation when an API returns something it shouldn't. Persist what you sent and what
   came back, in structured, queryable form (e.g. a Redshift table). It doubles as audit
   trail, dispute evidence, and material for reprocessing after a bug.
7. **Default to one provider; add redundancy only past a clear threshold.** A second
   provider is extremely expensive (development, fees, complexity), so don't reach for it by
   reflex. Escalate to redundancy when a single source's failure *or silent wrongness* is
   both **unrecoverable** and **material** — an irreversible money path with no other safety
   net, a hard regulatory availability bar, or data you can't otherwise verify. Separate the
   two kinds: **cheap redundancy** (cross-checking a read across two sources, e.g. two
   blockchain nodes) is worth reaching for early; **expensive redundancy** (a hot-failover
   backup bank/custodian/KYC vendor) only clears the bar for the highest-stakes paths.
8. **Don't trust the sandbox.** A provider sandbox is a good sign and fine for basic
   scenarios, but it diverges significantly from production. Be prepared to **test in
   production** (canary releases, controlled small-impact usage).

**Principles touched** — *No trust:* the provider's code, schema, and uptime are outside
your control, so verify facts against independent sources and validate at the boundary.
*No lost data:* persisting every request/response keeps a record to reconcile against and
reprocess from.

## Handling webhooks

Webhooks (HTTP endpoints you expose, called by an external system with a payload it
defines) are the most common way to receive external signals. Processing them safely is not
trivial. Many of these points apply to other transports too.

1. **Don't assume ordering.** Messages can arrive out of order or carry stale data — the
   last webhook isn't necessarily the latest truth. Don't blindly overwrite state; reconcile
   against what you already know (e.g. query the API for current state).
2. **Don't assume validity — default to "trigger, then confirm."** A webhook may come from
   a secondary part of the issuer's system and carry stale or improperly transformed data,
   so the default is to **treat the payload as a trigger and query the API for the
   authoritative state** rather than acting on the webhook's contents. Beware the API can be
   eventually consistent and lag the webhook — a query right after the trigger may still
   return the old state, so be ready to retry. You may trust the payload's content only when
   *all* of these hold: the signature is verified over the raw bytes (point 7); the field is
   monotonic or otherwise safe to apply twice; and there is no authoritative read endpoint
   to confirm against (some providers send data their API never exposes). Either way,
   persist the raw event and keep processing idempotent.
3. **Don't assume delivery.** Webhooks get lost sooner or later, regardless of the issuer's
   re-delivery promises. Handle a missing webhook with an independent process that fixes
   the completeness of your data — see Reconciliation.
4. **Don't assume single delivery.** The same webhook will arrive more than once.
   Processing must be **idempotent**.
5. **Acknowledge fast, process asynchronously.** Return a 2xx as soon as you've durably
   stored the raw event; do the real work async. Process inline and slow, and the issuer
   times out and retries — multiplying your load.
6. **Persist the raw payload.** Store what you received *verbatim* before acting. More
   reliable processing, an audit trail of what the provider actually said, and the ability
   to reprocess after a bug without asking the provider to resend.
7. **Verify the caller.** The issuer typically attaches a signature — usually an **HMAC**
   with a shared secret, less commonly an asymmetric signature with a published public
   half. Caveat: **verify the signature over the raw bytes you received**, not a
   re-serialized payload (re-serialization changes bytes and breaks the signature). Even
   then, prefer not to trust the content (see point 2).

Recurring theme: **don't trust the webhook.** Treat it as a hint that *something* happened,
not a trustworthy account of *what* happened.

**Principles touched** — *No trust:* a webhook is an unauthenticated, unordered,
possibly-lost, possibly-duplicated hint; verify the source and confirm actual state against
the API. *No lost data:* persist the raw event and back delivery with reconciliation so a
dropped webhook isn't a dropped fact.

## Notifying reliably (outbox and CDC)

Often you must reliably tell the external world about changes — publish a Kafka event,
dispatch a webhook, etc. The hard word is *reliably*: you need at-least-once delivery, and
these channels don't fit the usual transactionality model. Without transactionality you
risk either:

- **Publish then rollback.** The publish succeeded but you didn't get the response (network
  issue), so you roll back your state.
- **State change without publish.** The publish genuinely failed but you didn't roll back.

The textbook answer (2-phase commit / distributed transaction) is rarely used — too
complex, hard to standardize. Practical options:

1. **Outbox pattern.** Write a "publishing" event **transactionally** (with the state
   change) into a dedicated store, then reliably process it from there (take a row, retry
   until success). You reliably save *publishing intent*, then process it later.
2. **Change Data Capture (CDC).** An automated mechanism that detects committed DB changes
   (typically by tailing the write-ahead/replication log) and turns them into events.
   Because it reads straight from the log, every committed change is captured and nothing is
   missed, with no explicit publishing code. Tools: Debezium, AWS DMS. Tradeoff: coupling
   and operational weight — raw CDC emits events shaped like your table rows and needs
   postprocessing to avoid leaking the internal schema to consumers.
3. **Listen-to-yourself.** Reverse the order — publish the event first (e.g. to Kafka),
   then rebuild your own state from it.
4. **Event sourcing.** The event log already lives in the database, so publishing is just
   reading from it.

Picking a default: reach for the **outbox** when you control the application code (the
common case); **CDC** when you can't change the app or want zero publishing code in it;
**event sourcing** when you already keep an event log; **listen-to-yourself** only in the
niche where rebuilding state from the published event is genuinely simpler.

Whichever you pick, delivery is **at-least-once** — the relay/connector can publish and
crash before recording that it did, re-sending on restart. Consumers must therefore be
**idempotent** and deduplicate on a stable event id.

**Principles touched** — *No lost data:* a committed change must reliably reach consumers;
the outbox (or log) guarantees the notification can't be dropped because a separate publish
step failed. *No invented data:* never publish a notification for a change that didn't
commit, and duplicate deliveries collapse into a single effect.

## Reconciliation

Any system relying on external data is prone to **data drift** — one system not matching
another (a missed webhook; a transaction posted to the ledger but not reflected in the
provider's system). Reconciliation is the process that aligns the two. (In practice it may
be more than two — ledger, processor, bank — but the approach is the same.)

1. **Cadence.** Hourly, daily, monthly, or yearly — but *pick* it, don't default to "daily
   because everyone does." Set cadence no slower than the window in which an undetected drift
   becomes unrecoverable or materially harmful (tighter for high-value or irreversible
   flows), and no faster than the settlement horizon resolves — reconciling T+3 settlements
   every hour just alarms on transfers that aren't late, they're simply not settled yet.
2. **Nature of drift.** Data can be **missing** (the easy case) or **different** (same
   transaction, different amounts — much harder). Timing matters: with T+3 settlement,
   records stay unreconciled for 3 days, so build that into the process and don't alert on
   it.
3. **Matching algorithm.** Knowing *what* to compare is the hard part. Usually persist the
   external provider id in your system so matching is straightforward; otherwise heuristics
   enter (matching by amount and time).
4. **One-to-many.** Sometimes you reconcile many records on one side against one on the
   other — a single settlement transfer covering several transactions.
5. **Aligning is not trivial.** You can't simply overwrite data to make reconciliation
   happy. Each discrepancy must be understood and fixed through first-class support — a
   correction record, reprocessing webhook data, etc.

**Principles touched** — *No trust:* reconciliation is how you verify across independent
sources instead of believing any single one. *No lost data:* it's the safety net that
catches the dropped fact — the missing webhook, the unsettled transfer — before it
disappears for good.
