# Oracle Manipulation: Spot vs TWAP vs Stale Feeds

*How price-oracle dependencies become the single point of failure in DeFi, and how to tell a manipulable feed from a safe one.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

Most DeFi exploits that move real money route through a price. If an attacker can move the price the protocol reads — cheaply, atomically, or by waiting out a stale window — the downstream math (collateral value, liquidation threshold, mint/redeem rate) breaks.

## The three failure modes

**1. Spot-price from an AMM pool.** If the protocol reads `reserveB / reserveA` (or `slot0` on a concentrated-liquidity pool) as "the price," a flash loan can move that ratio within one transaction, let the protocol act on the manipulated value, then revert the swap. Classic flash-loan price manipulation.

**2. TWAP that's too short or too thin.** A time-weighted average resists single-block manipulation, but a short window (a few blocks) over a thin pool can still be pushed by an attacker willing to hold the position across the window, especially on low-liquidity pairs or L2s with cheap blocks.

**3. Stale / unbounded feeds.** A push oracle (Chainlink-style) can go stale. If the consumer doesn't check `updatedAt` freshness and `answeredInRound`, it may act on a price hours old — dangerous during volatility. Equally: missing min/max bounds means a mis-reported feed value is consumed verbatim.

## What to check in the consumer

- [ ] Is the price derived from **pool balances / spot** anywhere? (`getReserves`, `slot0`, `balanceOf`-ratios) → flash-loan surface.
- [ ] For TWAP: what is the window? How deep is the pool? Cost to move it across the window?
- [ ] For push feeds: is `updatedAt` checked against a staleness threshold? Is `answer > 0` and within sane bounds? Is `answeredInRound >= roundId`?
- [ ] Is there a single oracle, or cross-checked sources with deviation bounds?
- [ ] Decimals: does the consumer assume 8/18 decimals that the feed doesn't actually use?

## The honest-severity part

An oracle finding is only real if the manipulation is **economically profitable** after costs (flash-loan fee, gas, slippage, price impact). "The spot price can be moved" is not a finding; "moving it nets the attacker X after costs, draining vault Y" is. Model the economics before claiming severity. On deep, well-arbitraged pools, many theoretical manipulations aren't profitable — say so.

## Testing it

Fork mainnet, flash-borrow, swap to move the target pool, call the victim function, assert the protocol read the manipulated price and that the round-trip is profitable. If it isn't profitable, that's the answer.

## Takeaway

Trace every price back to its source. Spot-from-balances is the red flag; short/thin TWAPs are the yellow flag; unchecked staleness is the quiet killer. Then prove profitability before you assign a severity.
