# Governance, Timelocks & Emergency Powers

*Governance is a privileged path by design. The bugs are in the gaps: missing timelocks, flash-loan voting, and emergency functions that skip the guardrails.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

Governance and emergency mechanisms are *supposed* to be powerful. Auditing them isn't about "admin can do X" (that's the trust model) — it's about whether the *process* protecting that power can be subverted, or whether an emergency shortcut bypasses a safety users rely on.

## Attack surfaces

**1. Flash-loan governance.** If voting power = token balance snapshotted at the *current* block (not a past snapshot), an attacker flash-borrows the governance token, votes, and returns it in one transaction. Defense: snapshot voting power at proposal-creation block (checkpointing), not at vote time.

**2. Missing / bypassable timelock.** A timelock exists so users can exit before a malicious/mistaken change lands. Check: does *every* sensitive action route through it, or is there a path (an "emergency" function, a second admin role) that skips it? A timelock with a bypass is theater.

**3. Emergency powers too broad.** `pause()` is reasonable. An `emergencyWithdraw` that lets a role drain user funds "for safety," or an emergency upgrade with no delay, is a rug vector dressed as safety. Assess what the emergency role can *take*, not just *stop*.

**4. Proposal execution mismatch.** Does the executed calldata match what was voted on? If a proposal stores a hash and executes arbitrary matching calldata, is the binding tight? Can a passed proposal be executed with different parameters?

**5. Quorum / threshold gaming.** Low quorum + concentrated tokens = capture. Time-of-check for delegation vs time-of-vote. Double-voting via delegation cycles.

## What to check

- [ ] Is voting power from a **past snapshot** (checkpoint) or live balance? Live = flash-loan surface.
- [ ] Does every privileged state change go through the timelock, with no emergency bypass that skips the delay?
- [ ] What can the emergency role *take* (not just pause)? Is emergency scope bounded?
- [ ] Is executed calldata cryptographically bound to what was voted?
- [ ] Are quorum/threshold and delegation mechanics resistant to same-block manipulation?

## The scope line

"Governance can pass a malicious proposal if it controls enough votes" is usually the trust model — out of scope. **In scope:** governance being subverted by someone *without* legitimate voting power (flash-loan voting, timelock bypass reachable by a non-admin, calldata mismatch letting a passed-but-benign proposal execute something malicious). Draw that line explicitly in the report.

## Testing it

Flash-loan the gov token and attempt to pass/execute a proposal in one tx (assert it should fail due to snapshotting). Attempt a sensitive change via any non-timelock path (assert none exists). Vote on proposal A's calldata, then attempt to execute different calldata (assert binding holds).

## Takeaway

Don't report "admin is powerful." Report snapshot-less voting, timelock bypasses reachable without privilege, unbounded emergency withdrawal, and calldata-binding gaps — the places where the *process* fails, not where the trust model simply grants power.
