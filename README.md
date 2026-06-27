# Fintech Engineering Handbook — Claude skill

A [Claude](https://claude.com/claude-code) **skill** that packages the patterns and domain
knowledge needed to build software that handles money: ledgers, payments, wallets, FX,
settlement, reconciliation, custody, and the compliance controls around them.

It turns the excellent [*Fintech Engineering Handbook*](https://w.pitula.me/fintech-engineering-handbook/)
by **Voytek Pitula** into a progressively-disclosed skill so an AI agent reads only the
layer it needs for the task at hand.

## What it does

When a task touches money, the skill helps Claude:

- Lead every decision with the handbook's **three principles** — *no invented data, no lost
  data, no trust*.
- Pull the right reference for the layer in play — representing money, the ledger, executing
  flows, the external world, controls/access, or testing.
- Look up finance/payments/trading/crypto/compliance terms via a glossary.
- See the patterns combined in three worked flows (crypto withdrawal, card deposit, in-app
  conversion).

## Structure

```
fintech-engineering-handbook/
├── SKILL.md                 # entry point: 3 principles + a "which file do I read?" map
├── references/
│   ├── representing-money.md     # precision, rounding, currencies, FX rates
│   ├── ledger.md                 # double-entry, timestamps, audit trails, event sourcing, immutability
│   ├── executing-money-flows.md  # invariants, funds reservation, overdrafts, idempotency, resumability
│   ├── external-world.md         # consuming APIs, webhooks, outbox/CDC, reconciliation
│   ├── controls-and-access.md    # segregation of duties, access control, SDLC change trail
│   ├── testing.md                # property-based, invariant/idempotency injection, golden, prod testing
│   ├── glossary.md               # Appendix A: domain vocabulary + further reading
│   └── end-to-end-examples.md    # Appendix B: three worked flows
├── NOTICE.md                # attribution to the original handbook
└── README.md
```

## Install

**Claude Code** — clone (or copy) the skill folder into your skills directory:

```bash
git clone https://github.com/<owner>/fintech-engineering-handbook.git \
  ~/.claude/skills/fintech-engineering-handbook
```

It activates automatically when a conversation touches financial/money-handling systems.
You can also trigger it explicitly with `/fintech-engineering-handbook` (or by asking about
ledgers, idempotency for payments, reconciliation, etc.).

For other platforms (Codex, Copilot CLI, Gemini CLI), drop the folder into that tool's
skills location — the skill is plain Markdown with no dependencies.

## Attribution & license

Content is adapted from the *Fintech Engineering Handbook* by Voytek Pitula
(<https://w.pitula.me/fintech-engineering-handbook/>), reorganized into skill form. All
credit for the underlying material belongs to the original author. See [`NOTICE.md`](./NOTICE.md).
The original is described as a living document that welcomes contributions; please send
corrections upstream where appropriate.
