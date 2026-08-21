# MEV, Slippage & Sandwich Resistance

*Every value-moving transaction on a public mempool is visible to searchers before it lands. If a protocol doesn't defend the user, the user gets sandwiched.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

On a public chain, a pending swap/mint/liquidation is a broadcast of intent. Searchers front-run and back-run it (a "sandwich") to skim value, or reorder/censor it. Auditing for MEV is auditing whether the protocol leaves users exposed to this by design.

## Where MEV bites

**1. Unprotected swaps.** A swap with `minAmountOut = 0` (or a slippage bound the caller doesn't set) can be sandwiched: attacker buys before (pushing price up), user buys at the worse price, attacker sells after. The user's loss is the attacker's profit.

**2. Slippage defaults.** Does the protocol *require* a user-supplied `minOut`/`deadline`, or default them to unsafe values (0 / `type(uint).max` / `block.timestamp`)? A `deadline = block.timestamp` is no deadline — it's always "now," letting a validator hold the tx.

**3. Liquidations without protection.** A liquidation that dumps seized collateral on an AMM with no slippage bound both loses value and creates sandwich profit.

**4. Oracle update ordering.** If a price update and a dependent action can be ordered by a searcher (update → liquidate in the same block), that's an MEV-extractable sequence.

**5. Auction / mint ordering.** NFT mints, auctions, and first-come mechanisms are trivially front-run without commit-reveal or fair-ordering.

## What to check

- [ ] Does every swap/trade path enforce a **caller-supplied** `minAmountOut` and a real `deadline`?
- [ ] Are defaults safe, or does the protocol pass `0` / `block.timestamp` internally?
- [ ] Do liquidations/rebalances that hit an AMM have slippage protection?
- [ ] Is there any commit-reveal / batching / private-mempool assumption, and does the code actually enforce it?
- [ ] Can a searcher profit by ordering a protocol action relative to an oracle update?

## The honest-severity part

MEV/sandwich exposure is often rated **Low/Medium** because the loss is bounded by slippage and the user *can* protect themselves by setting parameters — unless the protocol hard-codes unsafe values or makes protection impossible. Distinguish "user could set minOut but the UI defaults it badly" (arguably not a contract finding) from "the contract ignores/overrides minOut" (a real finding). Also: MEV that harms users but doesn't break a protocol invariant may be out of scope for some programs — check the rules.

## Testing it

Fork, simulate the sandwich: attacker tx (front) → victim tx → attacker tx (back), all in one block, and measure victim loss vs attacker profit. For deadline: submit a tx and "hold" it (advance time) to show it still executes when it shouldn't.

## Takeaway

Check that every value-moving call takes a real, caller-controlled `minOut` and `deadline`, and that the protocol never substitutes unsafe defaults. Rate honestly: bounded, user-preventable slippage is usually low; a contract that ignores slippage protection is a real finding.
