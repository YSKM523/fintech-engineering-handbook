# Fintech Engineering Handbook — Claude skill

A [Claude](https://claude.com/claude-code) **skill** that packages the patterns and domain
knowledge needed to build software that handles money: ledgers, payments, wallets, FX,
settlement, reconciliation, custody, and the compliance controls around them.

It turns the excellent [*Fintech Engineering Handbook*](https://w.pitula.me/fintech-engineering-handbook/)
by **Voytek Pitula** into a progressively-disclosed skill so an AI agent reads only the
layer it needs for the task at hand.

> **Scope.** Money-movement engineering (payments, ledgers, settlement, reconciliation,
> custody balances) is first-class. Capital-markets / trading systems (market-making, order
> books, options & derivatives, fixed income, FX/rates OTC, repo/swaps) are covered only at
> an **orientation + architecture-risk** level — the skill supplies the vocabulary and flags
> the structural risks, then defers the deep design to specialists. It is not a
> capital-markets handbook.

## What it does

When a task touches money, the skill helps Claude:

- Lead every decision with the handbook's **three principles** — *no invented data, no lost
  data, no trust*.
- Pull the right reference for the layer in play — representing money, the ledger, executing
  flows, the external world, controls/access, or testing.
- Look up finance/payments/trading/crypto/compliance terms via a glossary.
- See the patterns combined in three worked flows (crypto withdrawal, card deposit, in-app
  conversion).

## What this skill adds beyond the original

Most of the text is the original handbook, reorganized. A handful of sections are the
maintainer's own extensions — informed by community discussion of the handbook — that turn
soft or absolute prose into testable, decision-oriented defaults. All are flagged in
[`NOTICE.md`](./NOTICE.md):

1. **Money representation — a per-context default stance + decision matrix** (ledger /
   storage / API transport / compute / display): amounts on the wire as strings with an
   explicit scale and a rounding-policy reference; never a bare number or implied decimals;
   arbitrary-width integers for crypto.
2. **Ledger balances are projections, not "never stored"** — a stored balance is fine if it's
   recomputable, reconcilable, explainable, and drift-detectable; what's forbidden is the
   *uncomputable* balance mutated with no entry behind it.
3. **A judgment-point sweep** — several soft or absolute calls turned into conditional
   defaults: idempotency dedup window, circuit breakers, provider redundancy, reservation
   backstop, key-reuse fingerprint, per-balance linearizability, testing defaults vs menu.
4. **Data lineage & replayability** — an 8-field provenance checklist for every external
   fact, append-only versioned facts, and replay of any report/decision against the exact
   input versions it used.
5. **The truth hierarchy** — per-source, per-role authority (trigger / booking basis / final
   reconciliation authority) with a per-domain table; "don't trust the webhook" becomes a
   role assignment, not a law.
6. **The effectively-once review checklist** — eight testable questions tying idempotency and
   resumability together, each mapped to a test (the goal is effectively-once *processing*,
   not exactly-once *delivery*).
7. **The compliance boundary** — split every recommendation into *engineerable* vs *not
   engineering's to decide alone*, with trigger surfaces and how to handle the second column.
8. **An explicit scope boundary** — money-movement engineering is first-class; capital
   markets is orientation + architecture-risk-flagging only (see the Scope note above).

## Structure

```
fintech-engineering-handbook/
├── SKILL.md                 # entry point: scope, 3 principles, forced-output defaults + a "which file?" map
├── references/
│   ├── representing-money.md     # precision (default stance + decision matrix), rounding, currencies, FX rates
│   ├── ledger.md                 # double-entry, balances-as-projections, timestamps, audit trails, event sourcing, immutability/GDPR
│   ├── executing-money-flows.md  # invariants, funds reservation, overdrafts, idempotency, resumability, effectively-once checklist
│   ├── external-world.md         # consuming APIs, truth hierarchy, webhooks, data lineage & replay, outbox/CDC, reconciliation
│   ├── controls-and-access.md    # compliance boundary, segregation of duties, access control, SDLC change trail
│   ├── testing.md                # property-based, invariant/idempotency injection, golden, prod testing
│   ├── glossary.md               # Appendix A: domain vocabulary (+ capital-markets orientation) + further reading
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
