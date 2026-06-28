---
name: fintech-engineering-handbook
description: >-
  Patterns and domain knowledge for building software that handles money — ledgers,
  payments, wallets, FX, settlement, reconciliation, custody. Use this whenever the user
  is designing, building, reviewing, or debugging any financial or fintech system:
  double-entry bookkeeping, money representation and precision/rounding, idempotency,
  funds reservation/holds, payment-provider webhooks, the outbox pattern, reconciliation,
  audit trails, or compliance controls (KYC/AML, four-eyes). Reach for it even when the
  user never says "fintech" — any time money amounts, balances, multiple currencies, FX
  rates, payment integrations, or a transaction ledger are involved, these patterns apply.
  Also use it as a glossary when a finance/payments/trading/crypto term is unclear.
  Scope: money-movement engineering (payments, ledgers, settlement, reconciliation, custody
  balances) is first-class; capital-markets/trading systems (market-making, order books,
  options/derivatives, fixed income, FX/rates OTC, repo/swaps) are covered only at an
  orientation + architecture-risk level — defer the deep design to specialists there.
---

# Fintech Engineering Handbook

Patterns for building software where **money is the primary subject** of the system. Use
this skill to make design decisions, review financial code, and avoid the failure modes
that are unique to money systems (silently lost precision, double-spends, drifted books,
unprovable audit trails).

> Source & attribution: this skill adapts the *Fintech Engineering Handbook* by
> **Voytek Pitula** — <https://w.pitula.me/fintech-engineering-handbook/>. See `NOTICE.md`.
> It is a living document; the original welcomes contributions.

## Scope & boundaries

**First-class:** money-movement engineering — payments, ledgers/double-entry, wallets and
custody *balances*, FX conversion, settlement, reconciliation. The money-handling patterns
this skill is built around; use it freely here.

**Orientation only — defer the deep modeling:** capital markets — trading and matching
engines, market-making, order books, options and other derivatives, fixed income, FX/rates
*OTC* structure, repo and swaps. This is **not** a capital-markets handbook. When a task
lands here: supply the vocabulary (`references/glossary.md` → Trading & markets), raise the
relevant **architecture risk flags** (positions and mark-to-market P&L instead of cash
balances; real-time margin and liquidation; latency/ordering as correctness; instrument
lifecycle/corporate actions; multi-party netted settlement; a heavier, different regulatory
surface — full list in the glossary's *Capital markets* section), and then **defer the core
design to capital-markets specialists** — fixed income and FX OTC structure in particular
take years to model correctly. For listed-market microstructure, point to *Trading and
Exchanges*. This is also a securities/derivatives **compliance-boundary** surface (below).

## The three principles (the whole point)

Every pattern in this handbook is a way to uphold three principles. When you are unsure
what to do in a money system, return to these and ask which one is at risk.

1. **No invented data.** Money can't appear from nowhere — no duplicates, no arbitrary
   balance updates. Enforced with idempotency, deduplication, and reconciliation.
2. **No lost data.** Everything that happens to money must be tracked and persisted.
   Protected with full precision, at-least-once delivery, event sourcing, audit trails,
   and immutability.
3. **No trust.** Trust neither external providers, nor internal components, nor the
   world. Upheld by verifying webhooks, cross-checking data across sources, and failing
   loudly when assumptions break.

Each reference file tags which principle(s) a pattern serves — that linkage is the point,
not decoration. If a design choice doesn't trace back to one of these, question it.

## How to use this skill

When a task touches money, **first identify which layer of the problem you're in**, then
read the matching reference file before writing code or giving advice. The layers stack:
you represent money, record it, execute flows over it, talk to the outside world, control
who can act, and test all of it. Don't dump every reference into context — pull the one(s)
the task actually needs.

| If the task is about… | Read |
| --- | --- |
| How to model/store/serialize an amount; precision, rounding, currencies, FX rate direction & timing | `references/representing-money.md` |
| Recording movements so the books balance and survive audit: double-entry, value/booking/settlement time, audit trails, event sourcing, immutability, reversals, GDPR | `references/ledger.md` |
| Keeping a single operation correct end-to-end: invariants, funds reservation/holds, overdrafts, idempotency, crash-resumable multi-step flows, the effectively-once review checklist | `references/executing-money-flows.md` |
| Talking to unreliable third parties: the truth hierarchy (which source is authoritative for what), consuming APIs, webhooks, reliable notification (outbox/CDC), reconciliation, data lineage & replayability | `references/external-world.md` |
| Who may act and proving the process was followed: the compliance boundary (what engineering must not decide alone), segregation of duties / four-eyes, access control, the SDLC change trail | `references/controls-and-access.md` |
| How to gain confidence: property-based testing, invariant/idempotency injection, crash-resume tests, round-trip, golden, backward-compat, testing in production | `references/testing.md` |
| A finance/payments/trading/crypto/compliance **term** you need defined | `references/glossary.md` |
| Seeing the patterns combined in a realistic flow (crypto withdrawal, card deposit, in-app conversion) | `references/end-to-end-examples.md` |

When you genuinely need the whole picture (e.g. an architecture review of a new money
system), the worked flows in `references/end-to-end-examples.md` are the fastest way to
see how the layers interlock; read it after the layer-specific files.

## The strongest default: representing money

State this up front whenever you design or review how amounts are stored or moved — it's
the most common *silent* failure, so don't wait to be asked. The default stance:

- **Ledger posting** always carries `amount + asset/currency + scale + a rounding-policy
  reference`. An amount without its scale and currency is ambiguous; a rounded amount with
  no pointer to why can't be audited.
- **API / cross-system boundary**: send the amount as a **string** with an **explicitly
  declared scale and currency**. Never a bare JSON number (it's a double), and never an
  integer that relies on *implied* decimals (that's how a 100× or 10¹²× error ships).
- **Computation** runs in decimal/rational at full precision; **round exactly once**, at
  the boundary, via the explicit policy.
- **Rounding residuals** are booked or attributed — never silently dropped (invented/lost
  money otherwise).

Crucially, this is **per-context**: minor-units integer is the right *storage/ledger* form
but a poor *interchange* form unless the scale travels with it. The full decision matrix
(ledger / storage / transport / compute / display) is in `references/representing-money.md`.

## Before any external integration: state the truth hierarchy

"Don't trust the webhook" is a reflex, not a law — in some domains a signed webhook is the
only authoritative signal there is. So don't mechanically declare any one channel the source
of truth. Instead, identify the domain (card PSP, ACH, blockchain, custodian,
market-data/KYC vendor, internal ledger) and **emit its source-of-truth hierarchy** across
three roles:

- **Trigger** — wakes you up (often a webhook/poll); never book on it alone.
- **Booking basis** — the most authoritative source available *at booking time* that you
  post the ledger from (usually an API read; the webhook itself only when nothing better
  exists).
- **Final reconciliation authority** — what wins disputes and you reconcile against later
  (settlement file, bank statement, chain finality, custodian statement).

Booking basis and final authority are usually *different* sources; the gap between them is
what reconciliation closes. Per-domain table in `references/external-world.md`.

## Reviewing any flow that touches the outside world: the effectively-once checklist

Exactly-once *delivery* is impossible; aim for **effectively-once processing** (exactly-once
*effect*) = at-least-once delivery + idempotent, resumable processing. When you design or
review such a flow, answer these eight out loud, every time — a design that can't answer one
has a hole:

1. What is the **idempotency key's scope** (which operation + which client)?
2. Is a **repeated payload allowed to differ** under the same key? (default: no — reject)
3. Are **error results replayed**, and which errors reprocess vs replay as-is?
4. Can **concurrent duplicates slip through**? (is the barrier atomic?)
5. What happens **after the dedup window expires**? (default: no window)
6. If an **external side effect happened but the internal transaction failed**, how does it recover?
7. Does **each workflow state have an independent driver** that can advance it?
8. Is **each step safe to re-run**?

Defaults, rationale, and the test that proves each (questions 1-5 → idempotency testing,
6-8 → crash/resume injection) are in `references/executing-money-flows.md`.

## Know the compliance boundary

This skill is engineering patterns, **not legal or regulatory advice**. In a regulated
financial institution, engineering must align with your lawyers, compliance, finance, and
internal handbooks — they own *what the rule is and who is liable*; engineering owns making
that rule correct, provable, and robust.

So whenever a task touches **fund custody, user assets, KYC/AML, sanctions screening,
financial/regulatory reporting, tax, GDPR/privacy, cross-border payments, securities or
derivatives, or crypto custody**, don't silently bake in a guess. Instead:

- **Split your recommendation into two columns** — *engineerable* (the how: data model,
  precision, idempotency, audit completeness, immutability/erasure mechanism, RBAC and
  four-eyes enforcement, reconciliation, replayability) vs *not engineering's to decide
  alone* (the what/who/whether: retention periods, screening lists and thresholds, KYC
  evidence and tiers, PII-vs-financial-record classification, tax treatment, licensing and
  travel-rule thresholds, whether an instrument is a regulated security, the custody model,
  whether a prod test moving real money is allowed).
- **Name the owner and the open decision** for each second-column item (compliance / legal /
  finance / risk) rather than hardcoding a number as if it were fact.
- **Default conservative and policy-configurable** until confirmed — a placeholder behind
  config with a visible TODO + owner; lean toward keeping data and *not* acting.

Build the engineerable half fully; flag the rest. Two-column table and mechanics in
`references/controls-and-access.md`.

## How to apply it well

- **Lead with the principle at risk.** Reviewing or designing, name which of the three is
  threatened ("this clamps a negative balance to zero → that *invents* money"). It makes
  feedback concrete and traceable.
- **Match rigor to stakes — in both directions.** Heavy patterns (event sourcing,
  hot-failover provider redundancy, strict by-construction immutability) are
  over-engineering on a low-stakes path, but above a stakes threshold they're *mandatory*,
  not optional — the references say where that line falls. Equally, the cheap-but-risky
  shortcuts (an idempotency dedup window, trusting a webhook's body, a single provider on a
  critical path) are fine *until* the stakes cross the same line. Decide by the blast radius
  of being wrong, not by habit or by what's easiest to build.
- **Representation is per-context, not one global choice.** Storage, computation, and the
  wire each get their own default (see the decision matrix). The frequent mistake is
  forcing one representation everywhere — e.g. shipping a raw minor-unit integer over an
  API and trusting the receiver to know the scale.
- **The outside world doesn't ask permission.** Many "impossible" states (overdrafts,
  out-of-order webhooks, duplicate deliveries) are forced on you by third parties. Design
  to *detect and recover*, not to assume they can't happen.
