# Controls and access

The other patterns keep the *data* correct. A money system must also constrain **who** may
act on it, and prove after the fact that the process was followed. Here the **No trust**
principle turns inward: your own operators and engineers are a trust boundary too, just like
external providers and internal components. An auditor examines these controls alongside the
books themselves.

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
