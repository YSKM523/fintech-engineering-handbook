# Glossary: know your domain

The hardest part of joining fintech is often not the code but the **vocabulary** — words
that sound ordinary but mean something precise, and acronyms everyone uses without ever
expanding. Use this as a lookup when a term is unclear. Terms a layperson already knows
(deposit, withdrawal, transfer, currency) and exotic corners are skipped; where the
handbook covers a concept properly, the entry points to that reference.

## Accounting & ledgers

- **Ledger** — the system of record for money movements; the source of truth balances are
  derived from. (See `ledger.md` → double-entry.)
- **General ledger vs sub-ledger** — the single consolidated book vs a detailed book for
  one domain (per user/product) that rolls up into it.
- **Debit / credit** — the two sides of every entry. Which one *increases* an account
  depends on the account's type, not on "money in vs out."
- **Posting** — committing an entry to the ledger; "posted" = recorded and, by convention,
  immutable.
- **Chart of accounts** — a catalogue of accounts that can be posted to; a system can have
  several (per legal entity, per book, per reporting standard).
- **Account type** — asset / liability / equity (plus revenue / expense), so the
  **accounting equation** holds and each account has a defined side on which it increases.
- **Receivable / payable** — money owed *to* you / money owed *by* you.
- **IOU** — informally, a liability. A user's balance on a custodial platform is an IOU
  from the platform to the user, which is why it sits on the liability side.
- **Accrual vs cash basis** — recognising money when earned/owed vs when it actually moves.
- **Trial balance** — a check that total debits equal total credits across the books.
- **Suspense / clearing account** — a temporary holding account for money in transit or not
  yet attributable.
- **Write-off** — booking a balance you no longer expect to recover as a loss.
- **Commingling** — mixing company funds with user funds; a regulatory red flag.
- **Reconciliation break** — a single unmatched discrepancy surfaced by reconciliation.

## Money & FX

- **Money (as a type)** — an amount paired with a currency.
- **Minor units** — the smallest indivisible unit of a currency; amounts often stored as
  integers of these (€12.34 → `1234`).
- **Basis point (bp / "bip")** — one hundredth of a percent (0.01%); fees and rates are
  routinely quoted in these.
- **Notional** — the face value a calculation is based on, which may be far larger than the
  cash that actually changes hands.
- **Fiat vs crypto** — state-issued currency vs blockchain-native asset.
- **Stablecoin** — a token pegged to a reference asset, usually a fiat currency like USD.
- **Pegged / wrapped / bridged** — representations connected to an underlying asset but
  *not* equivalent to it.
- **Bid / ask / spread** — the buy price, the sell price, and the gap between them.
- **Mid-market rate** — the midpoint between bid and ask; a reference point, not a price you
  can actually trade at.
- **Reference rate** — a rate used for valuation/equivalence (holdings value, a tax base),
  not for an actual trade.
- **Mark-to-market** — revaluing a holding at the current market rate rather than the price
  it was acquired at.

## Transactions, timing & settlement

- **Value date / booking date / settlement date** — when it happened / when we recorded it
  / when money actually moved.
- **T+X** — something (e.g. settlement) happens X business days after the value date (T+2).
- **Clearing vs settlement** — agreeing who owes what vs actually transferring the money.
- **Cut-off time** — the daily deadline after which a transaction rolls into the next
  settlement window.
- **Float** — money that, mid-transfer, appears to exist in both systems at once (or in
  neither).
- **Netting** — offsetting many obligations into a single net transfer instead of settling
  each gross.
- **Backdating** — assigning a value date earlier than the booking date.
- **Reversal / correction** — negating a posting in full as if it never happened
  economically vs booking the difference between what was recorded and what should have
  been.

## Payments, rails & cards

- **Payment rail** — the underlying network a payment travels over (SEPA, SWIFT, ACH, card
  networks, a blockchain).
- **IBAN / SWIFT / SEPA / ACH / FPS / CHAPS / wire** — identifiers and networks for moving
  fiat between banks.
- **Originator / beneficiary** — who sends / who receives a transfer (also remitter /
  payee, payer / payee).
- **PSP (payment service provider)** — a vendor that connects you to one or more rails.
- **Nostro / vostro account** — "our money held at their bank" / "their money held at ours";
  the accounts that make cross-bank transfers work.
- **Omnibus account** — one pooled account holding many users' funds together, with per-user
  balances tracked internally. Legitimate pooling, as opposed to commingling.
- **FBO ("for benefit of") account** — an account a company holds on behalf of its users.
- **Sweep** — automatically moving balances between accounts on a schedule (e.g. into cold
  storage or an interest-bearing account).
- **Chargeback** — a forced reversal of a card payment, initiated by the cardholder's bank.
- **Issuer / acquirer** — the cardholder's bank / the merchant's bank.
- **Interchange** — the fee the acquirer pays the issuer on each card transaction; the bulk
  of card-processing cost.
- **Authorization vs capture** — placing a hold on funds vs actually charging them; the card
  world's version of funds reservation.
- **Dunning** — the retry-and-notify process for recovering a failed recurring payment.

## Trading & markets

- **Order book** — the live list of outstanding buy (bid) and sell (ask) orders at each
  price level.
- **Market vs limit order** — execute now at the best available price (takes liquidity) vs
  only at a set price or better (rests on the book, provides liquidity).
- **Maker / taker** — maker adds a resting order, taker crosses the spread and removes one;
  fees usually differ.
- **Slippage** — the difference between the expected price and the price actually filled.
- **Liquidity / depth** — how much can be traded before the price moves; thin books move
  more.
- **Spot** — buying or selling the asset itself for immediate delivery.
- **Derivative** — a contract whose value derives from an underlying asset rather than the
  asset itself.
- **Futures / perpetual (perp)** — a contract to trade at a future date / one with no
  expiry, kept near spot by a funding rate.
- **Funding rate** — periodic payment between longs and shorts that tethers a perp's price
  to the index.
- **Long / short** — a position that profits when the price rises / falls.
- **Leverage / margin** — controlling a position larger than your capital / the collateral
  posted to open and maintain it.
- **Liquidation** — forced closing of a position when its margin falls below the maintenance
  requirement.
- **Haircut** — a discount applied to the value of collateral to cushion against price
  moves.
- **Counterparty** — the other side of a trade or contract; "counterparty risk" is the risk
  they fail to deliver.
- **AUM / AUC** — assets under management (actively managed, fee-bearing mandate) vs assets
  under custody (merely safekept). The same holdings can be one, the other, or both.

## Custody & crypto

- **Custody** — who controls the assets: self-custody (the user holds the keys) vs custodial
  (the platform/custodian holds them).
- **Hot / cold wallet** — keys kept online for quick access vs offline for security.
- **Private key / public key / address** — the secret that authorises spending / its derived
  public identifier / where funds are received.
- **Seed phrase** — the human-readable backup that reconstructs a wallet's keys.
- **Multisig / MPC** — requiring several keys (or key-shares) to authorise a transfer, so no
  single device can move funds alone.
- **Gas / network fee** — the fee paid to get a transaction included on a blockchain.
- **Confirmation / finality** — a block built on top of the one containing your transaction
  / the point at which it can no longer be reversed. More confirmations = more finality.
- **Reorg** — a blockchain reorganisation that can undo recently confirmed transactions; the
  reason finality isn't instant.
- **Mempool** — the pool of pending transactions waiting to be included in a block.
- **UTXO vs account model** — Bitcoin-style "spend whole coins, receive change" vs
  Ethereum-style running balances; the two demand different accounting.
- **Token vs coin** — an asset issued on top of a chain (e.g. an ERC-20) vs a chain's native
  asset (BTC, ETH).
- **Dust** — an amount so small the network fee to move it exceeds its value.
- **Address whitelisting (allow-listing)** — restricting withdrawals to a set of
  pre-approved addresses.

## Compliance & regulation

- **KYC** — Know Your Customer; verifying a user's identity.
- **AML / CFT** — Anti-Money-Laundering / Countering the Financing of Terrorism; controls to
  detect and prevent illicit funds.
- **Sanctions screening** — checking parties against sanctioned-entity lists.
- **PEP** — politically exposed person; a higher-risk category requiring extra diligence.
- **SoF / SoW** — source of funds / source of wealth; evidence of where a customer's money
  came from.
- **Travel Rule** — the requirement to share originator and beneficiary information on
  transfers above a threshold.
- **VASP** — Virtual Asset Service Provider; the regulatory label for a crypto business such
  as an exchange or custodian.
- **MiCA** — the EU's Markets in Crypto-Assets regulation.
- **Segregation of duties** — splitting a sensitive action across people so no single person
  can complete it alone. (See `controls-and-access.md`.)
- **Four-eyes / maker-checker / dual control** — requiring a second person to approve a
  sensitive action before it takes effect.
- **Least privilege / RBAC** — granting each actor the minimum access needed / managing it
  through roles rather than per-person grants.
- **Change management** — the controlled, traceable process (review, approval, deployment)
  by which code and config reach production.
- **Audit / audit trail** — external scrutiny that the books and controls reflect reality /
  the recorded history that lets any balance or decision be reproduced.

## Further resources

No single book covers money systems end to end. Grouped by layer:

**Accounting & ledgers**
- *Accounting for Computer Scientists* (Martin Kleppmann, free essay) — maps double-entry
  onto a graph/data model, for engineers.
- *The Accounting Game: Basic Accounting Fresh from the Lemonade Stand* — accounting from
  first principles; assumes no finance background.
- Modern Treasury, *How to Scale a Ledger* (free article series) — building a production
  ledger from a software-engineering angle.

**Payments & cards**
- *Payments Systems in the U.S.* — reference-style tour of how money moves between banks;
  US-centric, thorough, dry.
- *The Anatomy of the Swipe* — what happens between tapping a card and the money arriving;
  for builders and beginners.

**Markets & trading**
- *Trading and Exchanges: Market Microstructure for Practitioners* — where order books,
  makers/takers, and spreads come from; a long practitioner reference.

**Crypto**
- *Mastering Bitcoin* and *Mastering Ethereum* — engineering-level references for the two
  models a fintech engineer keeps meeting (UTXO and account).

**The engineering half**
- *Designing Data-Intensive Applications* — idempotency, logs, consistency, and the failure
  modes this handbook keeps returning to, from the systems side.

**KYC & AML** (written for compliance professionals, not engineers)
- *Anti-Money Laundering in a Nutshell* — short, awareness-level intro.
- *Mastering Anti-Money Laundering and Counter-Terrorist Financing* — heavier practitioner
  guide with checklists and example documents.
