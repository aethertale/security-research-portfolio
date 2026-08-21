# Cross-Chain Message Replay & Finality Races

*Cross-chain messaging turns one signed intent into a value transfer on another chain. Miss replay protection or a finality check and that intent gets executed more than once — or on a fork.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

A bridge or cross-chain app takes a message proven/signed on chain A and executes it on chain B. The safety of the whole system reduces to: **each valid message executes exactly once, on the right chain, after it's truly final.**

## Replay across executions

- [ ] Is there a per-message uniqueness record (`processedMessages[hash] = true`) checked *before* execution and set *before* any external call?
- [ ] Is the message hash **complete** — does it bind source chain ID, destination chain ID, a unique nonce/message-id, sender, recipient, and payload? A hash missing `destChainId` can be replayed on a sibling deployment.
- [ ] Is the nonce/id space per-source-chain, or global? A global counter shared across sources can collide.

## Replay across chains / deployments

- [ ] If the same validator/operator set secures multiple deployments, is a message for deployment 1 rejected by deployment 2? (This is the EIP-712 `chainId` + `address(this)` binding problem, applied to bridges.)
- [ ] On a chain fork/reorg, can a message proven pre-fork be replayed on both branches?

## Finality races

- [ ] How many confirmations does the relayer wait before treating a source event as final?
- [ ] Can a source-chain reorg un-do the deposit *after* the destination has already minted? (Deposit on A → mint on B → A reorgs the deposit away → B minted against nothing.)
- [ ] For optimistic systems: is the challenge/fraud window respected before finalizing withdrawals?
- [ ] For light-client / proof systems: is the proof bound to a finalized header, not a speculative one?

## Validation completeness

- [ ] Does the destination re-validate *everything* it depends on, or does it trust a field the source didn't actually attest to?
- [ ] Are signature thresholds enforced (m-of-n), and is the validator set the *current* one (not a stale set an attacker can produce signatures for)?

## The honest-severity part

Finality-race findings often require the ability to cause or exploit a reorg, which on high-finality chains is expensive or infeasible — model that before rating. Replay findings, by contrast, are usually cleanly demonstrable and high-impact when the missing binding is real. Separate "theoretically missing check" from "here's the double-spend."

## Testing it

Reconstruct a valid message, execute it once (assert success), execute the identical message again (assert revert on replay). For cross-deployment: execute the same message against a second instance sharing the validator set (assert it should reject but doesn't → finding). For finality: simulate source-event → dest-execute → source-rollback and assert the dest is now under-collateralized.

## Takeaway

Every cross-chain message needs: a complete, chain-bound hash; a one-shot processed-set written before external calls; a current validator set with enforced threshold; and a finality wait that survives reorgs. Enumerate which of these is missing, then prove the double-execution rather than asserting it.
