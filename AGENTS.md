# Fintech Engineering Handbook — agent skill

This repository is a portable **agent skill**: patterns for building software that handles
money — ledgers, payments, FX, settlement, reconciliation, custody, and the controls around
them. It is plain Markdown with no platform-specific code, usable by any AI coding agent or
by a human.

**Entry point: [`SKILL.md`](./SKILL.md). Read it first.** It carries the scope, the three
principles, the forced-output defaults, and a map of which reference file to read for which
task. Then pull only the relevant file(s) from [`references/`](./references/) for the task at
hand — don't load everything at once.

## The three principles (the whole point)

Every pattern is a way to uphold three principles; when unsure, ask which one is at risk.

1. **No invented data** — money can't appear from nowhere (idempotency, dedup, reconciliation).
2. **No lost data** — everything that happens to money is tracked and persisted (full
   precision, at-least-once delivery, event sourcing, audit trails, immutability).
3. **No trust** — verify external providers, internal components, and the world; fail loudly
   when assumptions break.

## When to use

Whenever you are designing, building, reviewing, or debugging any system that moves or records
money: double-entry ledgers, money representation/precision, idempotency, funds reservation,
payment-provider webhooks, the outbox pattern, reconciliation, audit trails, or compliance
controls. Reach for it even when "fintech" is never said — any time money amounts, balances,
multiple currencies, FX rates, payment integrations, or a transaction ledger are involved.

## Reference files (read on demand)

- `references/representing-money.md` — precision (default stance + decision matrix), rounding, currencies, FX rates
- `references/ledger.md` — double-entry, balances-as-projections, timestamps, audit trails, event sourcing, immutability/GDPR
- `references/executing-money-flows.md` — invariants, funds reservation, overdrafts, idempotency, resumability, the effectively-once checklist
- `references/external-world.md` — consuming APIs, the truth hierarchy, webhooks, data lineage & replay, outbox/CDC, reconciliation
- `references/controls-and-access.md` — the compliance boundary, segregation of duties, access control, the SDLC change trail
- `references/testing.md` — property-based, invariant/idempotency injection, crash-resume, golden, prod testing
- `references/glossary.md` — domain vocabulary (+ capital-markets orientation) and further reading
- `references/end-to-end-examples.md` — three worked flows (crypto withdrawal, card deposit, in-app conversion)

**Scope:** money-movement engineering is first-class; capital markets (trading, market-making,
derivatives, fixed income, FX/rates OTC, repo/swaps) is orientation + architecture-risk
flagging only — defer the deep design to specialists. See `SKILL.md` → *Scope & boundaries*.

> Source: adapted from the *Fintech Engineering Handbook* by Voytek Pitula
> (<https://w.pitula.me/fintech-engineering-handbook/>). See [`NOTICE.md`](./NOTICE.md).
