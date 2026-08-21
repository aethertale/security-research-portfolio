# Liquidation Logic: Incentive Gaps, Bad-Debt & Griefing

*Liquidations keep lending protocols solvent. When the incentive math is off, positions don't get liquidated, get over-liquidated, or become a griefing vector.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

Liquidation is the safety valve of any collateralized system. It only works if a rational third party is *paid enough* to liquidate unhealthy positions promptly, without being able to *abuse* the mechanism.

## Failure classes

**1. No/insufficient incentive → bad debt.** If the liquidation bonus doesn't cover gas + slippage + price-impact for a given position size, no one liquidates. The position rots past 100% LTV and the protocol eats bad debt. Common on: small positions (gas > bonus), illiquid collateral (slippage > bonus), or during congestion (gas spikes).

**2. Over-liquidation.** The liquidator can seize more collateral than needed to restore health (missing "close factor" cap, or bonus applied to the whole position instead of the repaid portion). The borrower is punished excessively; value leaks to the liquidator.

**3. Self-liquidation / wash griefing.** A borrower liquidates their own position to capture the bonus, or an attacker pushes a victim just under the threshold (via an oracle nudge or an interest-accrual tick) to trigger an unfair liquidation.

**4. Liquidation blocked by revert.** If liquidation transfers a token with a hook/blacklist (or calls a callback the borrower controls), the borrower can make their own position *un-liquidatable* by reverting the transfer — turning bad debt into a certainty.

**5. Rounding at the health boundary.** Off-by-rounding in the health-factor computation lets a position sit at exactly-threshold and flip un-liquidatable, or lets a liquidator trigger at 99.99%.

## What to check

- [ ] Is the liquidation bonus ≥ realistic cost (gas + slippage) across the range of position sizes and collateral liquidity?
- [ ] Is there a close-factor cap so only the unhealthy portion is liquidated?
- [ ] Can the borrower (or a token they chose) revert the seize transfer? (Hooks, blacklists, pausable collateral.)
- [ ] Is the health factor computed with rounding that favors solvency (round collateral down, debt up)?
- [ ] Can an attacker cheaply push a healthy position under threshold (oracle nudge, dust interest tick)?
- [ ] Is partial liquidation handled — does repaying part correctly reduce both debt and seized collateral proportionally?

## The honest-severity part

"No incentive → bad debt" is real but often conditional (specific size/liquidity/congestion). Model the *actual* conditions: does bad debt accrue under normal parameters, or only in an extreme scenario the protocol already accepts? A griefing that costs the attacker more than the victim loses is low-severity. Separate "protocol becomes insolvent under normal ops" from "edge case under a black-swan."

## Testing it

Fork: create a position, move the oracle to just-unhealthy, attempt liquidation as a third party, and check (a) it's profitable enough to actually happen, (b) the borrower can't block it, (c) the liquidator can't seize more than the close factor allows. For bad-debt: simulate a small/illiquid position and show no rational liquidator profits → debt persists.

## Takeaway

Liquidation bugs are economic, not just logical. Check that the bonus pays across all position sizes, that the close factor caps seizure, that the borrower can't block the seize, and that boundary rounding favors solvency — then model whether the failure happens under normal parameters or only in a scenario already priced in.
