# Access Control & Privilege Escalation Audit Notes

*Most "critical" bridge/DeFi findings reduce to a role assumption. Knowing which privilege abuses are in scope — and which are the trust model — is half the job.*

> Generic methodology writeup. No specific protocol or live finding is referenced.

## Build the role matrix first

Before hunting escalation, enumerate:

- [ ] Every privileged role (`owner`, `admin`, `MULTISIG_ROLE`, `RELAYER_ROLE`, `EMERGENCY_ROLE`, `PAUSER`, `UPGRADER`, …).
- [ ] Every function each role can call.
- [ ] Who holds each role at deploy, and how roles are granted/revoked (`grantRole`, `transferOwnership`, two-step vs one-step).

A single table of *role → functions* surfaces most problems immediately.

## Escalation patterns to check

- [ ] **Missing modifier.** A sensitive function lacks `onlyRole`/`onlyOwner`, or uses the wrong role. Grep every state-changing external function against the matrix.
- [ ] **Initialization capture.** `initialize()` sets `owner = msg.sender` and can be called by anyone (see uninitialized-proxy notes) → attacker becomes admin.
- [ ] **Role admin cycles.** Can role A grant itself role B, transitively reaching a role it shouldn't? Check `getRoleAdmin` wiring.
- [ ] **One-step ownership transfer.** `transferOwnership` to a wrong/zero address with no accept step = bricked or hijacked admin.
- [ ] **Signature-gated as pseudo-role.** A relayer signature that authorizes an action is a privilege; is the signer set current and threshold-enforced?
- [ ] **Default-admin left set.** `DEFAULT_ADMIN_ROLE` still held by a deployer EOA rather than governance.

## The scope line (this is where findings live or die)

Bug bounty programs almost universally treat **"a trusted role does something malicious"** as *out of scope* — it's the protocol's trust model, not a code defect. So:

- **In scope:** an *untrusted* actor gains a privilege they shouldn't (missing modifier, init capture, escalation cycle). This is a real finding.
- **Out of scope:** "the multisig can rug," "the owner can set a malicious oracle," "admin can upgrade to a stealing implementation." True, but assumed. Report as `REJECTED_SCOPE` and don't inflate.

The interesting middle: a *config* that a trusted role plausibly sets by mistake, enabling an untrusted exploit. Report it honestly with the privileged prerequisite stated explicitly, at a severity that reflects the prerequisite — not as a bare critical.

## Testing it

For each escalation hypothesis, write a test where a non-privileged address attempts the action and assert it *should* revert. If it succeeds, you have a finding. For init capture: call `initialize()` from a random address on a freshly deployed (proxy or implementation) and assert admin capture.

## Takeaway

Draw the role→function matrix, grep every external function against it, and separate untrusted-gains-privilege (in scope, real) from trusted-role-misbehaves (out of scope, trust model). Calibrate severity to the prerequisite, and state the prerequisite plainly.
