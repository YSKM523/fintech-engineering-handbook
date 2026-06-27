# Representing money

Before you can move or record money, you have to represent it: how a monetary value is
modeled, stored, computed, and converted. Get these wrong and every layer above inherits
the error.

## Precision handling

There are four primary ways to represent a monetary amount. The only hard rule is **never
use floating point**.

1. **Floating-point.** Built-in `float`/`double`. Unpredictable precision loss; almost
   never a good idea. Fastest and most memory-efficient, needs no libraries — but not
   worth it for money.
2. **Arbitrary precision.** Types like Java's `BigDecimal` let you control precision
   exactly. Predictable; you decide where and how rounding happens. Fits intermediate
   work like FX or pricing math where many operations chain together.
3. **Minor-units precision.** For most fiat, keep a fixed precision — the same the
   connected central-banking system uses. The number of digits is given by **ISO 4217**
   (don't assume 2 — it isn't always). Store the amount as an integer in its smallest
   unit: €12.34 → `1234`. Crypto uses the same integer-smallest-unit idea (satoshis for
   BTC, wei for ETH) with two twists: precision is **per-asset** and defined by the token
   itself (e.g. an ERC-20's `decimals`, often 18), and the magnitudes overflow 64-bit
   integers, so you need arbitrary-width integers.
4. **Rational numbers.** When no precision loss is acceptable. Most powerful, but: slower;
   cannot be converted to other formats without losing precision; usually needs a custom
   type or library.

Selection depends on the system's class and responsibilities — there's no rule of thumb
beyond "not floats." These are **not mutually exclusive**: how you *store* an amount and
how you *compute* with it are separate decisions, and systems combine them (e.g. integer
storage with `BigDecimal` for intermediate computation).

The same care applies to **serialization**. A bare JSON number is an IEEE-754 double in
most parsers, so serializing money as a number reintroduces the floating-point problem at
the edge no matter how careful you were internally. Send money either as a **string**
(`"12.34"`) or as an **integer in its smallest unit**.

**Principles touched** — *No lost data:* the wrong representation silently drops precision
that can never be recovered.

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
