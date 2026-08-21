# Weird ERC-20s: Fee-on-Transfer, Rebasing & Non-Standard Tokens

*The ERC-20 interface is a suggestion. Protocols that assume "amount in = amount received" break on tokens that don't play by the happy-path rules.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

Integrations routinely assume every ERC-20 behaves like a textbook one. Real tokens don't. Each deviation is an accounting bug waiting to happen.

## The non-standard behaviors

**1. Fee-on-transfer.** `transfer(to, 100)` delivers 95; 5 is skimmed. A protocol that credits the sender `100` (the passed amount, not the received amount) is now over-credited by 5 — repeatable, drains the pool.

**2. Rebasing / elastic supply.** Balances change out from under the contract (stETH, aTokens, elastic tokens). A vault storing a *share count* derived from a past balance mis-accounts after a rebase; one storing raw balances double-counts yield or loses it.

**3. Missing return value.** Some tokens (USDT being the famous one) don't return a `bool` from `transfer`/`approve`. A contract using `require(token.transfer(...))` reverts against them; must use `SafeERC20`.

**4. Approval race / non-zero approval.** Some tokens require allowance be set to 0 before a new non-zero approval. `approve(x)` then `approve(y)` reverts.

**5. Weird decimals.** Not everything is 18. USDC/USDT are 6; WBTC is 8; some are 0 or 2. Hard-coded `1e18` scaling corrupts value math.

**6. Blocklists / pausable / hooks.** USDC can freeze an address; ERC-777 gives the counterparty a callback (reentrancy surface, see the reentrancy writeup). A blocked recipient can brick a withdrawal queue.

**7. Double-entry / rebasing-to-zero / upgradeable tokens.** The token contract itself can change behavior via upgrade.

## What to check

- [ ] Does the protocol credit the **passed amount** or the **actual received** (`balanceAfter - balanceBefore`)? Only the latter is safe against fee-on-transfer.
- [ ] Does it store share counts or raw balances, and is that consistent with rebasing collateral?
- [ ] `SafeERC20` (or equivalent) for all transfers/approves? Or does it assume a `bool` return?
- [ ] Does it zero allowance before re-approving?
- [ ] Are decimals read from the token, or hard-coded?
- [ ] Can a blocklist/pause on an in-scope token brick a critical path (withdrawals, liquidations)?

## The honest-severity part

The finding's severity depends on **which tokens are actually in scope**. If the protocol only ever handles a fixed whitelist of standard 18-decimal tokens, "doesn't support fee-on-transfer" is not a finding. If it accepts *arbitrary* user-supplied tokens (permissionless listing, generic vault), these become real and often high-impact. Check the token-admission model before rating.

## Testing it

Deploy a mock fee-on-transfer / rebasing / no-return / 6-decimal token, run it through deposit/withdraw/liquidation, and assert the accounting invariant (`sum of user credits == actual contract balance`) holds. It usually won't on the happy-path assumption.

## Takeaway

Never assume "amount in = amount received," never assume 18 decimals, never assume a `bool` return, and always ask which tokens can actually reach the code. The bug is real only for tokens the protocol actually admits — check that first.
