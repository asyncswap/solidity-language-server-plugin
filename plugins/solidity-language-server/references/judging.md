# Finding Validation

**Core principle: Only findings with a passing Foundry POC are included in the final report.** Findings that cannot be demonstrated with a concrete test are demoted to leads. The POC is the proof — everything else is context.

Every finding passes four sequential gates plus a mandatory POC gate. Fail any gate → **rejected** or **demoted** to lead.

## Gate 1 — Refutation

Construct the strongest argument that the finding is wrong. Find the guard, check, or constraint that kills the attack — quote the exact line and trace how it blocks the claimed step.

- Concrete refutation (specific guard blocks exact claimed step) → **REJECTED** (or **DEMOTE** if code smell remains)
- Speculative refutation ("probably wouldn't happen") → **clears**, continue

## Gate 2 — Reachability

Prove the vulnerable state exists in a live deployment.

- Structurally impossible (enforced invariant prevents it) → **REJECTED**
- Requires privileged actions outside normal operation → **DEMOTE**
- Achievable through normal usage or common token behaviors → **clears**, continue

Content-addressed patterns (e.g., `orderId = keccak256(abi.encode(order))`) make input forgery structurally impossible — if a fake struct produces a different key that maps to zero state, the call reverts. This is a **valid refutation** for findings based on parameter forgery.

## Gate 3 — Trigger

Prove an unprivileged actor executes the attack.

- Only trusted roles can trigger → **DEMOTE**
- Costs exceed extraction → **REJECTED**
- Requires governance compromise (oracle config, fee changes) → **PARTIALLY CONFIRMED** with precondition noted
- Unprivileged actor triggers profitably → **clears**, continue

## Gate 4 — Impact

Prove material harm to an identifiable victim.

- Self-harm only → **REJECTED**
- Dust-level, no compounding → **DEMOTE**
- Material loss to identifiable victim → **clears**, continue

## Gate 5 — POC Validation (mandatory)

**Every finding that clears Gates 1-4 MUST have a Foundry POC test.** No exceptions.

- POC passes, demonstrates attacker profit or state corruption → **CONFIRMED** (cite test name, quantitative impact)
- POC passes, attack is attempted but reverts (defense holds) → **MITIGATED** (include as finding with downgraded severity, note which defense works)
- POC passes, impact only under governance compromise → **PARTIALLY CONFIRMED** (include with precondition noted)
- POC cannot be written (structural reason) → **DEMOTE** to lead with explanation
- POC fails / cannot reproduce → **DEMOTE** to lead

The POC test output (pass/fail, log values, balances) IS the proof field in the report.

## Confidence

Start at **100**, deduct:
- Partial attack path → **-20**
- Bounded non-compounding impact → **-15**
- Requires specific (but achievable) state → **-10**
- Requires governance compromise → **-15**
- CEI pattern followed (reentrancy finding) → **-20**
- POC confirms mitigation holds → **-25**

Confidence >= 80 gets Description + Attack Path + Proof + Fix.
Below 80 gets Description + Attack Path only.

## Safe patterns (do not flag)

- `unchecked` in 0.8+ (but verify reasoning)
- Explicit narrowing casts in 0.8+ (reverts on overflow)
- Content-addressed orderId preventing input forgery
- CEI pattern with state updated before external calls (but note as defense-in-depth recommendation if no reentrancy guard)
- `try/catch` on oracle calls with graceful degradation
- `onlyPoolManager` / `onlyProtocolOwner` guards
- SafeERC20 / balance-check-after-transfer patterns
- Two-step admin transfer

## Do Not Report

Linter/compiler issues, gas micro-opts, naming, NatSpec. Admin privileges by design. Missing events. Centralization without exploit path. Implausible preconditions (but fee-on-transfer, rebasing, blacklisting ARE plausible for contracts accepting arbitrary tokens).
