# Security Research Portfolio — aethertale

Independent security researcher. Smart contracts (EVM/Solidity) and cross-chain bridge protocols. Evidence-first: every finding ships with a runnable proof-of-concept and an honest severity assessment.

- **HackenProof:** [@aethertale](https://hackenproof.com/hackers/aethertale)
- **Focus areas:** cross-chain bridges · bridge lifecycle · signature/replay flaws · upgradeable (UUPS/proxy) safety · DeFi economic security
- **Primary tooling:** Foundry (`forge`/`cast`/`anvil`), Slither, Hardhat, fork & invariant testing

---

## Methodology

I run every engagement as an evidence-first pipeline. A finding does not exist until it is reproduced by a test that fails against the vulnerable code and passes against the fix.

1. **Surface scan** — protocol maturity, prior audits, upgrade paths, external dependencies.
2. **Deep dive** — full source read; map every external call, `delegatecall`, approval, and trust boundary.
3. **Hypothesize** — for each surface: *"what if the caller is malicious / the ordering is adversarial / the input is hostile?"*
4. **Build a runnable PoC** — a `forge test` that demonstrates the defect deterministically, plus a control test that isolates the root cause.
5. **Falsify honestly** — actively try to disprove my own finding before I believe it. Zero findings on a hardened target is a legitimate, reportable result.
6. **Report** — clean writeup + PoC + concrete fix recommendation + honest severity.

## Attack-pattern checklist (bridges & smart contracts)

- Reentrancy — CEI violations, callbacks before state update
- Access control — missing auth, merkle/signature bypass, malleability
- Signature safety — replay across chains/deployments, missing domain separation (EIP-712), missing deadline
- Nonce & ordering — monotonic-counter griefing, out-of-order relay, front-run of permissionless entrypoints
- Upgrade safety — uninitialized implementations, `_disableInitializers`, storage layout, proxy admin
- Oracle & pricing — spot vs TWAP, stale feeds, flash-loan-derived prices
- Token behavior — fee-on-transfer, rebase, weird decimals, hooks
- Cross-chain messaging — replay protection, validation, finality races
- Economic — fund-lock / griefing, liquidation incentive gaps, MEV surface

## Engagements

Bug-bounty and audit work is largely private or under coordinated disclosure. Representative work (details on request / under NDA):

- **Cross-chain bridge audit** *(private, pending coordinated disclosure)* — identified permissionless out-of-order deposit relay causing permanent fund-lock (griefing), missing EIP-712 domain separation enabling cross-deployment signature replay, and an uninitialized UUPS implementation. All reproduced with Foundry PoCs and a full regression suite.
- **Multiple protocol reviews** closed as *no-finding* after a full audit — reported honestly rather than padded with low-signal noise.

## Writeups

- [`writeups/bridge-nonce-ordering-pitfalls.md`](writeups/bridge-nonce-ordering-pitfalls.md) — how strictly-monotonic deposit nonces on a permissionless relay entrypoint become a permanent fund-lock griefing vector, and how to design them safely.

---

*Contact via HackenProof profile for engagements. Private PoCs and detailed reports available under NDA.*
