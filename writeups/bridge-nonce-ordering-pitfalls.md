# Bridge Deposit-Nonce Ordering Pitfalls

*How a strictly-monotonic deposit nonce on a permissionless relay entrypoint turns into a permanent fund-lock griefing vector — and how to design it safely.*

> Generic methodology writeup. No specific protocol, deployment, or live finding is referenced.

## The pattern

Many token bridges settle inbound deposits on the destination chain through a contract that:

1. accepts a deposit message signed by a threshold of bridge operators (e.g. 3-of-5),
2. is callable by **anyone** (a permissionless relayer submits the signed bundle),
3. protects against replay with a **strictly-monotonic nonce**: `require(depositNonce > lastDepositNonce)`.

Individually each choice is reasonable. Combined, they can create a denial-of-service that permanently locks user funds.

## Why the monotonic gate is dangerous here

Operators typically **pre-sign several deposits ahead** (a signing pipeline of depth N) and gossip the signed bundles on a public channel so any relayer can post them. That means: for a window of N deposits, valid threshold-signed bundles for nonces `k, k+1, …, k+N-1` are **all publicly available at the same time**, before any of them has been posted on-chain.

Because the entrypoint is permissionless and enforces only `depositNonce > lastDepositNonce`, an adversary can take a *genuinely signed* higher-nonce bundle and post it **first**:

```
lastDepositNonce = k-1
attacker posts nonce k+3  (real, threshold-signed)  ->  lastDepositNonce = k+3
honest relayer posts nonce k   (real, threshold-signed)  ->  REVERT: nonce not > k+3
        "                k+1  ->  REVERT
        "                k+2  ->  REVERT
```

Deposits `k, k+1, k+2` are valid, fully signed, and unspent — but can never be admitted. If nonces are deterministic (derived from a canonical ordering off-chain) they cannot be re-issued at a higher value, and if the contract has no gap-fill or recovery path, the funds those deposits represent (already locked/burned on the source chain) are **permanently stuck**.

Key point: **the attacker forges nothing.** Every signature is real. The only adversarial act is *ordering* — which the contract leaves unprotected. It doesn't even require malice: a natural race (a higher nonce reaching threshold and being posted before a lower one) triggers the same brick.

## Root cause

The replay defense and the ordering policy are conflated into one mechanism. The monotonic counter is **redundant** for replay (a per-message `processedDeposits[hash]` set already prevents double-processing), but it **adds** an ordering constraint the contract cannot safely enforce against adversarial or out-of-order submission.

## Safe designs

- **Per-message replay only.** Track `processedDeposits[keccak256(message)]` and drop the monotonic gate entirely. Order becomes irrelevant; each signed deposit is admitted exactly once, in any order.
- **Gap-tolerant admission.** If ordering must be preserved for accounting, accept any nonce ≥ `lastDepositNonce` and record a bitmap/set of processed nonces, rather than hard-rejecting everything below the high-water mark.
- **Owner/operator gap-fill.** As a defense-in-depth backstop, provide an authenticated recovery entrypoint to admit a stuck lower nonce — so a brick is recoverable without a full contract upgrade.
- **Bind signatures to a domain.** Independently, sign an EIP-712 payload including `chainId` and `address(this)` so a bundle can never be replayed against another deployment sharing the operator set.

## Testing it

A minimal Foundry PoC:

1. Deploy the inbox with a known operator set.
2. Produce genuinely-signed bundles for nonces 1, 2, 3.
3. Submit nonce 3 first, then attempt 1 and 2 — assert both revert.
4. Control: submit 1, 2, 3 in order — assert all succeed. This isolates *ordering* (not signature validity) as the root cause.

If step 3 reverts and step 4 succeeds, the brick is real and mechanically demonstrated rather than argued.

## Takeaway

On a permissionless entrypoint fed by pre-signed, publicly-gossiped bundles, **a strictly-monotonic nonce is a liability, not a safety feature.** Use per-message replay protection, tolerate gaps, and keep a recovery path. Never let submission *order* be a precondition for funds not getting stuck.
