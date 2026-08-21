# The Value of a Rigorous Zero-Finding Audit

*Why "no reportable findings" — reached through an evidence-gated, falsification-first process — is a legitimate and valuable audit outcome, not a failure.*

> Generic methodology writeup drawn from many completed engagements. No specific protocol or live finding is referenced.

## Zero is a result

A large fraction of serious audit work on mature, previously-audited protocols ends with **zero reportable findings**. This is not wasted effort and it is not a failure — provided the zero is *earned*: reached by generating real hypotheses and then rejecting each one with concrete evidence, not by a shallow pass that simply didn't look hard enough.

The difference between a lazy zero and a rigorous zero is the paper trail: a lazy zero has nothing behind it; a rigorous zero has a registry of candidates, each with a verdict and the exact reason it was rejected.

## The falsification-first loop

For each candidate vulnerability I run it through a fixed gauntlet and record where it dies:

1. **Scope gate** — is the precondition actually reachable by an untrusted actor? Many "critical" findings assume a privileged role (admin/multisig/relayer) doing something malicious. If exploitation requires a trusted role to misbehave, most programs treat it as out of scope (trusted-role assumption). Record it as `REJECTED_SCOPE`, don't inflate it.
2. **Refutation gate** — does the code actually permit the exploit? Reentrancy guard present? CEI respected? Replay salt + deadline enforced? Read the guard, don't assume its absence. Most candidates die here as `REFUTED`.
3. **Exploitability gate** — is there a concrete, profitable, reproducible path? A theoretical rounding error that nets dust, or requires infeasible market conditions, is `NOT_EXPLOITABLE`.
4. **Known-issue gate** — was this already reported in a published audit or is it a documented design tradeoff? If so it's a duplicate / out of scope, `NON_REPORTABLE`.

A candidate only becomes reportable if it survives all four. On hardened targets, most don't — and that's the correct answer.

## Common reasons strong-looking findings die

- **Privileged prerequisite.** "Admin can rug via a malicious mapping/upgrade." True, but that's the trust model, not a defect.
- **Guard actually present.** The suspected reentrancy/CEI/replay hole turns out to be covered by a `nonReentrant`, a `usedHashes[h]` set, or a `deadline`.
- **Infeasible economics.** The manipulation works on paper but the capital/market conditions to profit don't exist, or MEV/incentive checks neutralize it.
- **Intended behavior.** The "bug" is a documented tradeoff the protocol accepts.
- **Out of the assessed surface.** The vulnerable code is at an unaudited HEAD or in an out-of-scope dependency (report upstream).

## Why the honest zero matters

- **Credibility.** A researcher who submits five padded lows that all get rejected is trusted less than one who reports a clean, well-documented zero and one real high.
- **Signal to the protocol.** A rigorous zero with a candidate registry tells the team *what was checked and why it held* — which is genuinely useful assurance, not an empty result.
- **Reusable assets.** Every refuted hypothesis becomes a reusable check for the next target. The attack-pattern checklist grows with each engagement.

## What a rigorous zero looks like

- A candidate registry: ID, title, verdict, gate-of-death, one-line rationale.
- Explicit scope reasoning for anything rejected as trusted-role / out-of-scope.
- A statement of coverage: which contracts/surfaces were read, and any residual (the honest "~8–12% unassessed" rather than a false "100%").
- No PoC theater: if there's no exploit, there's no PoC, and that's stated plainly.

## Takeaway

Don't chase a finding that isn't there. Generate hypotheses aggressively, then try just as hard to kill them, and document the kills. A defensible zero-finding report — with a candidate registry and honest coverage — is a professional deliverable in its own right.
