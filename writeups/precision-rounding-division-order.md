# Precision, Rounding & Integer Division Order

*Solidity has no floats. Every division truncates. Where you divide, in what order, and which way you round decides whether value leaks.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

Fixed-point math in smart contracts is a minefield of truncation. Most rounding bugs net dust and don't matter; a few compound, cross a boundary, or favor the wrong party and become real. This is how to tell them apart.

## The core hazards

**1. Division before multiplication.** `a / b * c` truncates at `a / b` and loses precision; `a * c / b` preserves it. The classic ordering bug — grep for `/` followed later by `*` on the same quantity.

**2. Rounding direction.** Every truncation must favor the *protocol*, not the user (see the ERC-4626 writeup for the deposit/withdraw/mint/redeem directions). A truncation that rounds a user's payout *up* or their cost *down* leaks value.

**3. Precision loss at small scale.** With 6-decimal tokens (USDC) or small amounts, intermediate truncation can zero out a legitimate result (e.g. interest on a small balance rounds to 0 → free borrow, or fee rounds to 0 → free action).

**4. Scaling mismatches.** Mixing 6-decimal and 18-decimal quantities without normalizing. `WAD`/`RAY` conversions done in the wrong order. Hard-coded `1e18` against a non-18 token.

**5. Accumulated drift.** A per-operation rounding of a few wei is dust once — but a function callable thousands of times, each skimming in the attacker's favor, aggregates to real value. Reachability × frequency = severity.

## What to check

- [ ] Every `/` — is there a `*` that should come first? (`mulDiv` / `FullMath` avoids overflow while preserving order.)
- [ ] Every conversion — does it round in the protocol's favor?
- [ ] Small-amount / low-decimal paths — can a legitimate result truncate to 0 (free action) or can the attacker choose amounts that always round their way?
- [ ] Are all quantities normalized to a common scale before arithmetic?
- [ ] Is any rounding-favorable operation callable repeatedly with attacker-chosen amounts?

## The honest-severity part

This is the class most prone to **over-reporting**. A rounding error that leaks 1 wei per call, callable at a gas cost far exceeding the gain, is `NOT_EXPLOITABLE` — report it as informational at most. The real findings are: (a) truncation-to-zero that grants a free action, (b) wrong-direction rounding on a high-value or high-frequency path, (c) drift that a cheap loop can compound into meaningful value. Always compute the *actual* extractable amount vs gas before assigning severity.

## Testing it

Unit-test the arithmetic at boundary inputs: 1 wei, `type(uint).max`-adjacent, and low-decimal amounts. Assert the invariant (`sum of payouts ≤ sum of deposits`, `fee ≥ minimum`, `shares round down`). For drift: loop the operation N times and assert cumulative leak, then compare to N × gas cost.

## Takeaway

Grep every division for order and direction, focus on truncation-to-zero and repeatable attacker-chosen-amount paths, and — more than any other class — compute the real extractable value before calling it a finding. Most rounding "bugs" are dust; the few that aren't are worth finding precisely.
