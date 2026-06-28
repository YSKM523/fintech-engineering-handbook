# Controls and access

The other patterns keep the *data* correct. A money system must also constrain **who** may
act on it, and prove after the fact that the process was followed. Here the **No trust**
principle turns inward: your own operators and engineers are a trust boundary too, just like
external providers and internal components. An auditor examines these controls alongside the
books themselves.

## The compliance boundary (what engineering must not decide alone)

The patterns in this handbook are **engineering** patterns. In a regulated financial
institution they sit inside a second system of rules — laws, regulator guidance, your
compliance policies, internal handbooks — and **engineering does not own those rules**. The
healthy division of labour: compliance / legal / finance / risk decide *what the rule is and
who is liable*; engineering decides *how to implement it correctly, provably, and robustly*.
Most of this handbook is the second half of a requirement whose first half a regulator wrote.

This is not a disclaimer to wave and move on — that would be the useless version. It's an
operational habit: when a task touches a regulated surface, **partition the work and flag the
decisions that aren't yours**, instead of silently hardcoding a guess that *looks*
authoritative.

**When to flag.** Treat these as trigger surfaces — if the task touches one, the compliance
boundary is in play: fund custody, user assets, KYC, AML/CFT, sanctions screening, financial
or regulatory reporting, tax, GDPR/privacy and data retention, cross-border payments,
securities or derivatives, and crypto custody.

**Split the recommendation into two columns:**

| Engineerable (the *how* — decide and build) | Not engineering's to decide alone (the *what / who / whether*) |
|---|---|
| Data model, precision, the rounding *mechanism* | The rounding/tax *rule* and who gets the fraction (tax/legal) |
| Idempotency, reconciliation, audit-trail completeness | *Which* transactions/parties must be screened or reported, and the thresholds (compliance) |
| Immutability + the crypto-shredding *mechanism* for erasure | Whether a given erasure request may be refused; what counts as PII vs a financial record (legal) |
| Retention *enforcement* (TTL jobs, legal hold) | The retention *period* per record class (legal/regulator) |
| RBAC, four-eyes, break-glass *enforcement* | Who is *allowed* to approve what; segregation requirements (compliance) |
| Sanctions-screening *integration* and match plumbing | The list sources, the match threshold, and the action on a hit — block / freeze / file a report (compliance) |
| Replayable decision records (rules engine, audit) | The KYC evidence required, risk tiers, re-verification cadence (compliance) |
| The custody data model and segregation *mechanism* | Whether the custody / omnibus / FBO model is *legally permitted* here (legal) |
| Synthetic prod transactions wired through the ledger | Whether testing in production with real money is *permitted*, and how it must be disclosed (compliance) |
| Cross-border payment *plumbing* and travel-rule field capture | Licensing, permitted corridors, the travel-rule *threshold* (legal/compliance) |

**How to handle the second column:**

1. **Name the owner and the open question** — "retention period: compliance to confirm per
   record class," not a silent `RETENTION_DAYS = 2555`.
2. **Default conservative and make it policy-configurable** — a placeholder threshold/period
   behind config with a visible TODO + owner, never a confident guess presented as fact. The
   safe default leans toward *keeping* data and *not* acting (don't auto-delete, don't
   auto-release funds) until confirmed.
3. **Make the rule a replayable artifact, not buried code.** When compliance does set a
   threshold or rule, encode it where it can be reviewed, versioned, and replayed (a decision
   table / rules engine — see Audits and audit trails), so the control is auditable and the
   business can change it without an engineer.
4. **Still build the engineerable half fully.** Flagging the boundary is not an excuse to
   stop — the value you add is a correct, provable, robust implementation that is *ready* for
   the rule the moment compliance sets it.

**Principles touched** — *No trust:* the compliance boundary is **No trust** pointed at your
own authority — engineering verifying it isn't overstepping a decision that belongs to legal,
compliance, or finance. *No lost data:* defaulting conservative (keep, don't act) until a rule
is confirmed protects records you may be legally obliged to retain.

## Segregation of duties and four-eyes

Some actions are too sensitive to leave to a single person, however trusted. Splitting them
is the oldest control in finance, in two related forms:

- **Segregation of duties** — no one person owns a whole process.
- **Four-eyes / maker-checker (dual control)** — a specific action needs a second person to
  approve it before it takes effect.

1. **It applies to money operations.** Large or manual withdrawals, manual ledger
   corrections, treasury and cold-wallet moves, changing a fee schedule or a limit —
   anything that can move or misstate funds is a candidate for a second approver.
2. **It applies to engineering too.** Merging code, deploying to production, and changing
   infrastructure are sensitive in a money system. Require review and approval.
3. **The approval is part of the trail.** Record who requested, who approved, and that the
   two were different people — otherwise the control is unprovable (see audit trails).
4. **Break-glass needs a path.** Emergencies happen, and a rigid control invites people to
   route around it. Provide an explicit, heavily-audited override rather than forcing a
   backdoor.

**Principles touched** — *No trust:* a single internal actor — even a trusted one — is not
sufficient authority for a sensitive or irreversible action.

## Access control

Who can do what is itself part of the system's state, and it changes over time as people
join, move teams, and leave. It's not enough to know who can touch funds *today*; auditors
also ask **how** they came to have that access.

1. **Least privilege.** Grant each actor — human or service — the minimum needed, and
   prefer roles (**RBAC**) over per-person grants so access stays reviewable.
2. **Authorization changes need a trail.** Granting or revoking a capability is a sensitive
   event, exactly like a money movement: record what changed, who changed it, and why. The
   audit-trail discipline from the ledger applies here too.
3. **Review access periodically.** Permissions go stale. Scheduled access reviews
   (recertification) are the **post-factum** check applied to access, catching the drift.

**Principles touched** — *No trust:* standing access quietly accumulates; least privilege
and periodic review keep it in check.

## The change trail (SDLC)

In a regulated environment you usually must audit how code reaches production: who reviewed
a change, who approved it, when it shipped. Your version control and CI/CD are a great help
if done right.

1. **Source control is the record.** Commit history attributes every change to an author
   and ties it — through review and linked tickets — to the reason it was made (the usual
   *what / who / why* an audit trail demands). Protect it: signed commits, protected
   branches, no force-pushing shared history.
2. **Reviews and pipelines must be enforced.** Required (non-optional) reviews, status
   checks, and "no direct pushes to main" are crucial — discipline doesn't fly in audits.
3. **Deployments are traceable.** Which version is running, who released it, and when should
   be reconstructable — this is what lets an incident be tied back to the change that
   caused it.

**Principles touched** — *No lost data:* the history of how the system itself came to be is
as much a part of the trail as the history of the money it holds. *No trust:* the system
enforces the delivery controls — it doesn't rely on people remembering to follow them.
