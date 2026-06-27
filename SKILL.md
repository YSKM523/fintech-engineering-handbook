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
---

# Fintech Engineering Handbook

Patterns for building software where **money is the primary subject** of the system. Use
this skill to make design decisions, review financial code, and avoid the failure modes
that are unique to money systems (silently lost precision, double-spends, drifted books,
unprovable audit trails).

> Source & attribution: this skill adapts the *Fintech Engineering Handbook* by
> **Voytek Pitula** — <https://w.pitula.me/fintech-engineering-handbook/>. See `NOTICE.md`.
> It is a living document; the original welcomes contributions.

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
| Keeping a single operation correct end-to-end: invariants, funds reservation/holds, overdrafts, idempotency, crash-resumable multi-step flows | `references/executing-money-flows.md` |
| Talking to unreliable third parties: consuming APIs, webhooks, reliable notification (outbox/CDC), reconciliation | `references/external-world.md` |
| Who may act and proving the process was followed: segregation of duties / four-eyes, access control, the SDLC change trail | `references/controls-and-access.md` |
| How to gain confidence: property-based testing, invariant/idempotency injection, crash-resume tests, round-trip, golden, backward-compat, testing in production | `references/testing.md` |
| A finance/payments/trading/crypto/compliance **term** you need defined | `references/glossary.md` |
| Seeing the patterns combined in a realistic flow (crypto withdrawal, card deposit, in-app conversion) | `references/end-to-end-examples.md` |

When you genuinely need the whole picture (e.g. an architecture review of a new money
system), the worked flows in `references/end-to-end-examples.md` are the fastest way to
see how the layers interlock; read it after the layer-specific files.

## How to apply it well

- **Lead with the principle at risk.** Reviewing or designing, name which of the three is
  threatened ("this clamps a negative balance to zero → that *invents* money"). It makes
  feedback concrete and traceable.
- **Don't over-engineer.** Several patterns (event sourcing, provider redundancy, strict
  immutability, idempotency time-windows) carry real cost. The references note when the
  cheaper option is correct. Match the rigor to the stakes.
- **Representation and computation are separate decisions.** A system commonly stores
  amounts one way (integer minor units) and computes another (`BigDecimal`); say so
  rather than forcing one choice everywhere.
- **The outside world doesn't ask permission.** Many "impossible" states (overdrafts,
  out-of-order webhooks, duplicate deliveries) are forced on you by third parties. Design
  to *detect and recover*, not to assume they can't happen.
