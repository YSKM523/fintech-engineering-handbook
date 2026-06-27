# Representing money

Before you can move or record money, you have to represent it: how a monetary value is
modeled, stored, computed, and converted. Get these wrong and every layer above inherits
the error.

## Precision handling

Money representation is the most consequential low-level decision in a money system: get it
wrong and every layer above silently inherits the error. The mistake to avoid is picking
*one* representation and forcing it everywhere. **Representation is per-context** — how you
store an amount, compute with it, and put it on the wire are separate decisions, and the
right default differs for each.

### Default stance (apply unless you have a specific, written reason not to)

These are the house rules. They are deliberately opinionated because the failure here is
silent — a wrong amount that balances and looks fine until an audit or a customer finds it.

1. **A ledger posting must explicitly carry `amount + asset/currency + scale (precision) +
   a reference to the rounding policy` that produced it.** A number without its scale and
   currency is ambiguous; a rounded number without a pointer to *why* it was rounded can't
   be audited or reproduced.
2. **Never put money on a JSON/API boundary as a bare number.** A bare JSON number is an
   IEEE-754 double in most parsers, which reintroduces float error exactly where you
   control it least. Send the amount as a **string**.
3. **Every cross-system boundary must declare its scale explicitly.** Do not rely on
   *implied decimals* — that the receiver "knows" USD has 2 and this token has 18. Implied
   scale is how you ship a 100× or 10¹²× incident.
4. **Computation may use decimal or rational at full precision, but persisting to the
   ledger must pass through an explicit rounding policy.** Full precision in flight; one
   deliberate rounding decision at the boundary.
5. **Rounding residuals must be booked or attributed — never silently dropped.** A
   fraction that vanishes is either invented or lost money (see Rounding strategies).

### The building blocks

Four ways to represent an amount. The only hard rule is **never floating point**.

1. **Floating-point.** Built-in `float`/`double`. Unpredictable precision loss; almost
   never a good idea. Fastest and most memory-efficient, needs no libraries — but not
   worth it for money.
2. **Arbitrary precision** (`BigDecimal` / decimal). Control precision exactly;
   predictable; you decide where and how rounding happens. Fits intermediate work like FX
   or pricing math where many operations chain together.
3. **Minor-units integer.** For most fiat, keep a fixed precision — the same the connected
   central-banking system uses. The number of digits is given by **ISO 4217** (don't
   assume 2 — JPY is 0, BHD is 3). Store the amount as an integer in its smallest unit:
   €12.34 → `1234`. Crypto uses the same integer-smallest-unit idea (satoshis for BTC, wei
   for ETH) with two twists: precision is **per-asset** and defined by the token itself
   (an ERC-20's `decimals`, often 18; USDC is 6), and the magnitudes overflow 64-bit
   integers, so you need arbitrary-width integers.
4. **Rational numbers.** When no precision loss is acceptable. Most powerful, but: slower;
   cannot be converted to other formats without losing precision; usually needs a custom
   type or library.

### Which representation, by context

Pick per layer; these combine in one system (e.g. integer storage + `BigDecimal`
computation + string on the wire).

| Context | Default | Why |
| --- | --- | --- |
| **Ledger posting** (source of truth) | integer in the asset's smallest unit, stored *with* currency/asset, scale, and a rounding-policy reference | exact, no float; the scale + currency make the integer unambiguous; the policy ref makes it reproducible under audit |
| **Database storage** | integer smallest-unit in a wide-enough type, or fixed-scale `DECIMAL(p, s)`; arbitrary-width for crypto; always store currency + scale alongside | exactness without float; 18-decimal wei overflows `int64`, so int64 is unsafe for crypto |
| **API / cross-system transport** | amount as a **string**, plus an **explicit** currency/asset **and scale** | a bare number is a double; a bare minor-unit integer is ambiguous across currencies/tokens and overflows JS `Number` |
| **Computation (intermediate)** | `BigDecimal` / decimal, or rational when zero loss is required | predictable; keep full precision and round exactly once, at the edge |
| **Report / display** | rounded to the currency's display precision via the explicit rounding policy | a presentation concern, *derived* from the ledger — never a source of truth |

### Serialization & interchange pitfalls

The wire is where carefully-kept internal precision most often leaks away. Note especially
that **minor-units integer is an excellent *storage* form but a treacherous *interchange*
form** unless the scale travels with it:

- **Bare JSON number → double.** `{"amount": 12.34}` is parsed as IEEE-754 in most stacks,
  re-importing the float problem at the edge.
- **Bare minor-unit integer is ambiguous.** `1234` is €12.34, ¥1234, or 1234 wei depending
  on a scale the integer doesn't carry. ISO 4217 decimals differ between currencies, crypto
  `decimals` differ between tokens, and stablecoins / wrapped or bridged assets can imply
  *different* decimals for the "same" logical asset across chains. An interchange integer
  without an explicit scale is a bug waiting for the next currency you add.
- **Large integers overflow JS `Number`.** 18-decimal wei exceeds 2⁵³, so even
  integer-in-JSON silently corrupts in any JavaScript consumer. A string is the safe
  carrier.

Resolution (the default rule above): on a boundary, send `amount` as a **string** alongside
an **explicit currency/asset and scale**, so the receiver never has to guess.

**Principles touched** — *No lost data:* the wrong representation, or an undeclared scale at
a boundary, silently drops precision that can never be recovered. *No invented data:* an
implied or mismatched scale can multiply an amount by a power of ten — money conjured from a
parsing mismatch, not from any posting.

## Rounding strategies

1. **Rounding is inevitable** and should be **explicit**: any division, currency
   conversion, fee, interest/rate application, or move between precisions may require it.
2. **It's a business decision.** Strategies have different implications — be conservative
   and round down (don't spend what you don't have), or use half-even when statistical
   effects matter. Who gets the fraction can have legal/tax implications.
3. **Round as seldom as possible.** The longer you keep full precision, the more options
   you have to decide correctly in context. Round on boundaries — before persisting or
   displaying.
4. **Rounding breaks sums.** Split a number into parts and round, and the parts may no
   longer sum to the original. Sometimes this needs explicit handling, e.g. a dedicated
   **rounding account**.

**Principles touched** — *No lost data:* residuals must be tracked, not dropped.
*No invented data:* rounding must never mint money that wasn't there.

## Currency handling

Money is never a number alone — it comes paired with a currency.

1. **Pack amount and currency together.** A `Money` newtype (struct/class/record)
   minimizes errors.
2. **No cross-currency arithmetic.** Prohibit adding two amounts in different currencies.
   Conversion happens explicitly, with a strictly controlled rate.
3. **Use a controlled currency set.** A config entry, the JDK database, a dedicated
   service — never accept arbitrary currency codes; validate at the system boundary.
4. **Codes identify fiat only.** Currency codes are unique identifiers only for fiat. For
   crypto you need something richer, e.g. `(network, contract address)`.
5. **Currencies carry metadata** (symbol, precision, name). Usually needed for display,
   rarely for business logic.
6. **Pegged is not the underlying.** Pegged, bridged, and wrapped crypto are *not*
   equivalent to the assets they track.

**Principles touched** — *No trust:* validate currency against the controlled set at the
boundary. *No invented data:* treating distinct currencies/assets as interchangeable
conjures value.

## FX rates

FX (foreign-exchange) rates convert money between currencies.

1. **A rate is always directional.** EUR/USD is not the inverted USD/EUR. On an exchange,
   buying and selling are different orders at different prices (the **bid/ask spread**),
   so the two directions don't simply invert.
2. **The time of the rate is critical.** You can use a rate from any point in time; the
   most common are:
   - **Current-time rate** — value current holdings, or a transaction as if it happened now.
   - **Value-date rate** — compute change in value or a tax amount.
3. **Two kinds of rate matter for conversion:**
   1. **Transactional rate** — the rate a real conversion happened at. You don't store it
      directly; it falls out of the original and result amounts.
   2. **Reference rate** (mid-market or central bank) — used for valuation and equivalence
      (what holdings are worth now, a tax base at the value date), not a price anyone
      actually trades at.
4. **There is no canonical rate.** Rates come from markets and vary by venue and method.
   Central-bank rates are the closest to canonical and usable only as a reference rate —
   and even there, alternative sources are just as valid.

**Principles touched** — *No lost data:* keep the amounts (and, for reference rates, a way
back to the source). *No trust:* there's no canonical rate, so the source should be part
of the data.
