# Flash-Loan Attack Surface Mapping

*Flash loans don't create vulnerabilities — they remove the capital barrier to exploiting ones that were already there. Audit for what becomes possible when an attacker temporarily controls unlimited capital.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

A flash loan gives an attacker millions of dollars for the duration of one transaction, repaid before it ends. It doesn't introduce new bugs; it makes previously "you'd need to be a whale" attacks free. Every audit should ask: *what invariant assumes no single actor has this much capital in one block?*

## What flash loans unlock

**1. Price manipulation.** Move an AMM pool the protocol reads as an oracle, act on the skewed price, revert the swap. (See the oracle writeup — flash loans are the delivery mechanism.)

**2. Governance capture.** Borrow the governance token, vote, return it — if voting power is live-balance rather than snapshotted. (See the governance writeup.)

**3. Collateral / liquidation games.** Inflate collateral value momentarily to over-borrow, or crash a victim's collateral to trigger a profitable self-directed liquidation.

**4. Share/rate manipulation.** In a vault whose exchange rate depends on `balanceOf(this)`, flash-donate to spike the rate, mint/redeem at the favorable rate, repay.

**5. Reserve-ratio / curve exploits.** AMMs and bonding curves that assume gradual liquidity changes can be pushed to an extreme point where the invariant math misbehaves.

## The audit question for every function

For each state-changing function, ask:

- [ ] Does it read any value that a single transaction can move — pool reserves, spot price, `balanceOf`, total supply, a rate derived from these?
- [ ] Does any check assume an attacker can't afford a large position?
- [ ] Is there an atomic path: borrow → manipulate → act on manipulated state → restore → repay, that nets profit?
- [ ] Are there per-block or same-transaction guards (e.g. TWAP, snapshotting, "can't deposit and withdraw same block")?

## Defenses to look for (and whose absence is the finding)

- Snapshot/checkpoint values from a *past* block, not the current one.
- TWAP over a window deep enough that moving it costs more than the exploit yields.
- Use *virtual* reserves / offsets rather than raw balances.
- Explicit same-block / flash-loan guards where appropriate.

## The honest-severity part

The finding is only real if the full atomic round-trip is **net profitable** after flash-loan fee, gas, slippage, and price impact. Many "flash-loan manipulations" cost more to execute than they extract on deep, liquid, well-arbitraged markets. Build the profit model. "This could be flash-loan manipulated" without a profitability model is a hypothesis, not a finding.

## Testing it

Fork mainnet, take a real flash loan from a live provider, execute the manipulation + victim call + repayment in one transaction, and assert the attacker ends with more than they started (after all costs). If the transaction reverts on repayment or nets a loss, the invariant holds — report that honestly.

## Takeaway

Flash loans turn "needs a whale" into "needs a transaction." For every function, find the values a single block can move and the checks that assume capital scarcity — then prove the round-trip profits before calling it a finding.
