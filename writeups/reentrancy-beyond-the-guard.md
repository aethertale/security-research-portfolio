# Reentrancy Beyond the Guard

*Reentrancy didn't die with `nonReentrant`. Read-only reentrancy, cross-function reentrancy, and cross-contract callbacks still bite.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

Everyone adds `nonReentrant` to state-changing functions now, so the naive "withdraw calls me back before balance update" bug is rarer. But the guard only protects the functions it's on, and only against re-entering the *same* guarded set. Several live variants slip past it.

## Variants that survive a naive guard

**1. Read-only reentrancy.** A `view` function (no guard — why would a getter need one?) reads state that is mid-update during an external call. A second protocol that trusts that getter as an oracle reads a corrupted intermediate value. The vulnerable contract never gets "reentered" in the classic sense; the *consumer* is the victim. Guards on state-changing functions don't help.

**2. Cross-function reentrancy.** Function A is guarded and makes an external call mid-execution. During the callback the attacker calls function B — also guarded, but by a *different* lock, or reading state A hasn't finished updating. If A and B don't share the same reentrancy lock and touch overlapping state, the guard is bypassed.

**3. Cross-contract reentrancy.** Contracts X and Y share state (or Y trusts X's balances). X is guarded; the callback re-enters Y, which acts on X's not-yet-updated state.

**4. ERC-777 / hooks / ERC-1155 callbacks.** Any token with transfer hooks (ERC-777 `tokensReceived`, ERC-1155 `onERC1155Received`, ERC-721 `onERC721Received`) gives the counterparty execution during a transfer — a callback you may not have realized existed.

## How to hunt it

- [ ] List every external call (token transfer, low-level `call`, callback) and ask: what state is not yet finalized at that point?
- [ ] Check whether `view` functions read mid-update state that any external protocol consumes as truth (read-only reentrancy).
- [ ] Map which functions share a reentrancy lock. Functions touching the same state must share the *same* lock.
- [ ] Flag any token with transfer hooks in scope.
- [ ] Verify strict CEI (checks-effects-interactions): are *all* state writes done before *every* external call?

## Testing it

Write an attacker contract that implements the relevant callback/hook and re-enters the target (or a consumer of the target) during the window. Assert the invariant break (double withdrawal, stale-read profit). For read-only reentrancy, assert the getter returns a corrupted value mid-call.

## Takeaway

`nonReentrant` protects a set of functions against re-entry into that set. It does not protect getters, differently-locked functions, or external consumers of your mid-update state. Enumerate every external call and every callback-capable token, then ask who can act while your state is inconsistent.
