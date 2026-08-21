# Upgradeable Proxy Safety Checklist (UUPS / Transparent)

*A field checklist for auditing upgradeable smart contracts. Each item is a concrete thing to read in the code, not a vibe.*

> Generic methodology writeup. No specific protocol, deployment, or live finding is referenced.

Upgradeable contracts trade immutability for flexibility, and every mechanism that enables the upgrade is also an attack surface. This is the checklist I run on any proxy-based system.

## 1. Uninitialized implementation

The logic (implementation) contract behind a proxy must never be left initializable.

- [ ] Does the implementation have a `constructor() { _disableInitializers(); }`?
- [ ] If not: can anyone call `initialize()` **directly on the implementation** and become its `owner`?
- [ ] For UUPS specifically: an attacker-owned implementation still holds `_authorizeUpgrade` — assess what that lets them do (historically, selfdestruct-to-brick, though EIP-6780 + OZ v5 largely neutralize that path today).

**Honest severity note:** post-Dencun (EIP-6780) and with OZ v5, the classic "brick the proxy by selfdestructing a hijacked implementation" escalation is mostly dead. So an uninitialized implementation is often **Low / best-practice** rather than critical. Report it as hardening with the correct fix, and don't overstate the impact.

## 2. Initializer hygiene

- [ ] Is `initialize()` protected by the `initializer` modifier (single-shot)?
- [ ] Any `reinitializer(n)` — is the version monotonic and intended?
- [ ] Can `initialize()` be front-run at deploy time? (Deploy + initialize should be atomic — e.g. proxy constructor calls it via `initData`.)
- [ ] Do parent contracts' `__X_init` chains all get called exactly once?

## 3. Storage layout

- [ ] Does an upgrade **append** new variables only, never reorder/insert/retype existing slots?
- [ ] Is there a `uint256[N] __gap;` in each upgradeable base to reserve slots for future variables?
- [ ] For `struct`s in storage: are new fields appended, not inserted?
- [ ] Do inherited contracts preserve linearization order across the upgrade?

A storage collision silently corrupts state after upgrade — one of the highest-impact, easiest-to-miss proxy bugs.

## 4. Upgrade authorization

- [ ] UUPS: is `_authorizeUpgrade(address)` overridden and gated (`onlyOwner` / `onlyRole`)? An unguarded `_authorizeUpgrade` = anyone upgrades = total compromise.
- [ ] Transparent: is the `ProxyAdmin` owner a multisig/timelock, not an EOA?
- [ ] Is there a timelock between proposing and executing an upgrade?
- [ ] Can the upgrade path be reached without the intended governance process?

## 5. Function-selector & admin-collision (Transparent proxies)

- [ ] Does any implementation function collide with the proxy's admin selectors?
- [ ] Is the admin address unable to accidentally call through to implementation logic?

## 6. `delegatecall` & context

- [ ] Does the implementation ever `delegatecall` untrusted targets?
- [ ] Are `immutable`/`constant` values (set in the implementation's constructor) correctly *not* expected to persist in proxy storage? (Common confusion: `immutable` lives in bytecode, not storage.)

## How to test

- Fork or unit-test the **upgrade transition**: deploy vN, write state, upgrade to vN+1, assert every prior storage variable still reads correctly.
- Attempt `initialize()` directly on the implementation address in a test; assert it either reverts (disabled) or is harmless.
- Attempt an upgrade from a non-authorized address; assert revert.

## Takeaway

Read the constructor (`_disableInitializers`), the `_authorizeUpgrade` guard, and the storage layout / `__gap` before anything else. Those three cover the majority of real proxy vulnerabilities. And calibrate severity honestly — several proxy issues are best-practice hardening, not criticals.
