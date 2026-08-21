# ERC-4626 Vault Rounding & The First-Depositor Inflation Attack

*Share-based vaults live and die by rounding direction. Round the wrong way, or leave the vault empty, and depositors lose funds.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

Tokenized vaults (ERC-4626 and lookalikes) convert between assets and shares with `convertToShares` / `convertToAssets`. Two recurring bug classes come from *how* that conversion rounds and *what happens when the vault is empty*.

## The first-depositor inflation attack

Classic sequence on an empty vault:

1. Attacker deposits 1 wei of asset → mints 1 share. Now `totalShares = 1`, `totalAssets = 1`.
2. Attacker **donates** a large amount of the asset directly to the vault (raw `transfer`, bypassing `deposit`). Now `totalAssets = 1 + BIG`, `totalShares = 1`. One share is worth `1 + BIG`.
3. Victim deposits `X`. Shares minted = `X * totalShares / totalAssets = X * 1 / (1+BIG)`, which **rounds down to 0** if `X ≤ BIG`.
4. Victim gets 0 shares for a real deposit; attacker's single share now owns the victim's assets too.

## Rounding direction

The invariant: **round in the protocol's favor, never the user's.**

- `deposit` (assets → shares): round **down** (user gets no more shares than earned).
- `withdraw` (shares → assets): round **down** (user withdraws no more assets than owed).
- `mint` (shares → assets to pay): round **up** (user pays enough).
- `redeem` (shares → assets out): round **down**.

A single flipped rounding direction lets an attacker repeatedly deposit/withdraw to skim dust that compounds — or leaves the vault insolvent by fractions that accumulate.

## What to check

- [ ] Is there **virtual shares / virtual assets** (OZ ERC-4626 offset) or a **minimum initial deposit** / dead-shares mint to block the first-depositor attack?
- [ ] Does the vault use `totalAssets()` = `balanceOf(this)`? If so, raw donation inflates it → attack surface.
- [ ] Is every conversion's rounding direction in the protocol's favor? Check all four of deposit/mint/withdraw/redeem.
- [ ] Division before multiplication anywhere? (`a / b * c` loses precision vs `a * c / b`.)
- [ ] Fee-on-transfer / rebasing assets: does the vault use *actual* received balance (`balanceAfter - balanceBefore`) or the passed `amount`?

## The honest-severity part

Many rounding findings net only **dust** per operation and aren't economically meaningful even if technically real — `NOT_EXPLOITABLE` in practice. The first-depositor attack, by contrast, is high-impact but only if the vault can actually be reached in an empty state (many are seeded at deploy). Check the *reachability* of the empty-vault precondition before rating it critical.

## Testing it

Fork/unit-test: empty vault → attacker 1-wei deposit → donation → victim deposit → assert victim shares round to 0 (inflation), or loop deposit/withdraw and assert vault balance drifts (rounding skim). Confirm the profit exceeds gas.

## Takeaway

For any share-based vault, first check the empty-vault defense (virtual offset / dead shares / seed), then verify all four rounding directions favor the protocol. Then check whether the leak is dust or drainage before assigning severity.
