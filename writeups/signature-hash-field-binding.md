# Signature-Hash Field Binding: The Confused-Deputy Bridge Bug

*When a signed message omits a field that distinguishes two otherwise-identical operations, one authorization can be redirected to another — the confused-deputy pattern in cross-chain bridges.*

> Generic methodology writeup. No specific protocol, deployment, or live finding is referenced.

## The pattern

Bridges authorize cross-chain actions with an operator signature over a hash of the request:

```solidity
bytes32 h = keccak256(abi.encodePacked(
    sender, toAddress, targetTokenAddress, amount,
    originChainId, targetChainId, deadline, salt
));
require(_recover(h, sig) == relayer);
```

This *looks* complete — sender, destination, token, amount, both chain IDs, a deadline, and a replay salt. But the safety of the scheme depends on one question: **does the signed set of fields uniquely identify the exact operation the signer intended to authorize?**

If two distinct mappings/routes can share the same `(targetTokenAddress, originChainId, targetChainId)` while differing in a field that is **not** in the hash — say a `mapId` or the `originTokenAddress` — then a signature issued for one is byte-for-byte valid for the other. The signer authorized deputy A; an attacker points it at deputy B.

## Why it hides

The omitted field is usually an *internal* identifier that feels like "just a lookup key," not "security-relevant input." Developers hash the user-facing parameters (who, what token, how much, where) and forget that the contract's own routing table can contain two entries the user-facing parameters don't disambiguate. The hash is complete with respect to the *user*, incomplete with respect to the *contract's state*.

## How to find it

For every signature-gated entrypoint, build a table:

1. List every field the handler *reads* to decide what to do (including values looked up from storage via an ID).
2. List every field that goes *into the signed hash*.
3. **The difference is your attack surface.** Any decision-relevant field not in the hash is a candidate for redirection.

Then ask: can two valid configurations exist that agree on all hashed fields but differ on an un-hashed decision field? If yes, and if reaching that second configuration doesn't require a trusted role, it's exploitable.

## The trusted-prerequisite caveat (be honest)

Often the second configuration can only be created by a privileged role (e.g. a multisig registering a duplicate mapping). If so, the "attack" reduces to *trusted admin misconfigures the system* — which most programs treat as out of scope (trusted-role assumption), **not** a pure code defect. Report it honestly as a hardening recommendation with a clear statement of the privileged prerequisite, rather than inflating it to a critical. Credibility is worth more than a rejected submission.

That said, the fix is cheap and correct regardless of exploitability today:

## The fix

Bind the signature to a domain and to *every* decision-relevant field:

```solidity
bytes32 h = _hashTypedDataV4(keccak256(abi.encode(
    TYPEHASH, mapId, originTokenAddress, sender, toAddress,
    targetTokenAddress, amount, originChainId, targetChainId,
    deadline, salt
)));  // EIP-712: also binds chainId + address(this)
```

Now a signature is valid only for the exact route, on the exact contract, on the exact chain it was produced for.

## Takeaway

A signed hash is only as safe as it is *complete*. Enumerate what the handler reads versus what the signer signed; the gap is where confused-deputy bugs live. And when the gap is only reachable by a trusted role, say so plainly.
