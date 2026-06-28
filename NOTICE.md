# Attribution

This skill is an adaptation of:

> **Fintech Engineering Handbook — Patterns for building software that handles money**
> Author: **Voytek Pitula**
> Source: <https://w.pitula.me/fintech-engineering-handbook/>

The substance, structure, examples, and wording of the patterns originate with that
handbook. This repository reorganizes the material into the Claude skill format
(`SKILL.md` + `references/`) with progressive disclosure and light editorial reformatting
(imperative phrasing, navigation, "when to read this" pointers).

A few places additionally add the maintainer's own decision-oriented guidance that
*extends* the original — most notably the money-representation **default stance** and
per-context decision matrix in `references/representing-money.md`, the
*stored-balances-are-projections* framing in `references/ledger.md`, the **data lineage &
replayability** and **truth hierarchy** (per-source/per-role authority) sections in
`references/external-world.md`, the **effectively-once review checklist** in
`references/executing-money-flows.md`, the **compliance boundary** (engineerable vs
not-engineering's-to-decide) in `references/controls-and-access.md`, an explicit **scope
boundary** (money-movement first-class; capital markets orientation-only) in `SKILL.md` and
`references/glossary.md`, and a sweep turning several soft or absolute judgment calls into
conditional defaults. These are informed by community
discussion of the handbook. That added guidance is the maintainer's recommendation, not the
original author's words; the underlying principles and the bulk of the text remain Voytek
Pitula's.

All credit for the underlying ideas and most of the text belongs to the original author.
If you find this useful, read and share the original. The original is presented as a living
document that welcomes contributions — please direct substantive corrections upstream.

If you are the author and would like attribution adjusted, the content licensed differently,
or this taken down, please open an issue.
