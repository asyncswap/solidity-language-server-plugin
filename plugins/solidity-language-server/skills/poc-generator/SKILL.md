---
name: poc-generator
description: Generate Foundry proof-of-concept tests that validate HIGH and CRITICAL security findings from entrypoint analysis. Produces runnable Solidity test files with attack contracts, setup scaffolding, and pass/fail assertions. Use after running entrypoint-analysis or call-analysis when findings need validation.
argument-hint: <file-path> [--finding N] [--all-high]
allowed-tools: LSP Grep Glob Read Write Edit Bash
---

# Solidity POC Generator

Generate Foundry proof-of-concept tests that validate HIGH and CRITICAL security findings. Each POC is a self-contained Foundry test that demonstrates whether a finding is exploitable or safely mitigated.

## Input

- `$ARGUMENTS` — Parse the arguments:
  - **file-path** (required): Path to the Solidity file that was analyzed
  - **--finding N** (optional): Generate a POC for finding #N only. If omitted, generate POCs for all HIGH and CRITICAL findings.
  - **--all-high** (optional): Include MEDIUM findings as well.

If no findings are provided in the arguments, check the conversation history for the most recent entrypoint-analysis or call-analysis output and extract findings from there.

## Phase 1: Finding Extraction

### Step 1: Identify findings to validate

From the conversation context or the target file, identify all HIGH and CRITICAL findings. For each finding, extract:

- **Finding ID and title**
- **Severity**
- **Entrypoint** — the function being attacked
- **Attack vector** — what the attacker controls (parameter, external call, callback)
- **Sink** — where the unsanitized input ends up
- **Proof sketch** — the attack scenario from the analysis

### Step 2: Classify POC type

For each finding, determine the POC approach:

| Attack Vector | POC Type | Requires |
|---|---|---|
| Reentrancy via ERC-20 | Malicious token contract with reentrant `transferFrom`/`transfer` | Custom ERC-20 with callback hook |
| Reentrancy via ETH receive | Malicious contract with reentrant `receive()` | Attack contract deployed as swapper/recipient |
| Oracle manipulation | Adversarial oracle returning controlled prices | Mock oracle implementing the protocol's oracle interface |
| Access control bypass | Direct call from unauthorized address | `vm.prank()` with attacker address |
| Input validation bypass | Crafted parameters that skip checks | Specific parameter values that reach sinks |
| State corruption | Cross-entrypoint state manipulation | Multi-step scenario: write state in tx1, exploit in tx2 |
| Arithmetic overflow/underflow | Edge-case inputs near bounds | Large values, zero values, max uint values |
| Front-running / sandwich | Mempool-observable tx ordering | Multi-tx scenario with `vm.prank` for attacker/victim |

## Phase 2: Scaffold Discovery

### Step 3: Find existing test infrastructure

Search the test directory for:

```
Patterns to find:
  - Test base contracts (SetupHook, BaseTest, etc.)
  - Helper functions (_makeOrder, _swap, _netInput, etc.)
  - Existing mock contracts (MockERC20, MockOracle, etc.)
  - Deployment patterns (deployCodeTo, vm.etch, etc.)
  - Import conventions
```

Use `Glob test/**/*.sol` and `Grep` to find:
1. The test base contract that sets up the hook + pool + tokens
2. Existing attack contracts (ReentrantToken, MaliciousOracle, etc.) — reuse if applicable
3. The import style and pragma version used in existing tests
4. Existing POC or reentrancy tests — check whether a finding is already covered before writing a duplicate

**Critical**: Reuse existing test infrastructure. Do NOT duplicate setup logic that already exists in a base contract. Inherit from the base test contract and use its helpers (`_swap`, `_makeOrder`, `_netInput`, etc.).

### Step 4: Read the target contract's interfaces

For each finding, identify which interfaces the POC needs to interact with:
- The target contract's function signatures (from LSP documentSymbol or reading the file)
- Any oracle interfaces the protocol uses (for oracle manipulation POCs)
- ERC-20 interface (for token-based reentrancy)
- Callback interfaces (for callback-based attacks)

## Phase 3: POC Generation

### Step 5: Generate attack contracts

For each finding, generate the necessary attack contracts. Follow these patterns:

#### Malicious ERC-20 (reentrancy)
```solidity
contract MaliciousToken {
    // Minimal ERC-20 with reentrant hook in transferFrom
    // Must: track balances, implement approve/transfer/transferFrom
    // Attack: in transferFrom, call back into the target contract
    // Control: arm() function to set attack parameters
    // Safety: disarm after one re-entry to prevent infinite loops
    // Tracking: reentrancyCalls counter for assertion verification
}
```

#### Malicious ETH Receiver (reentrancy)
```solidity
contract MaliciousReceiver {
    // Contract with receive() that re-enters on ETH receipt
    // Must: store attack parameters, disarm after one attempt
    // Attack: in receive(), call fill() or other entrypoint
    // Tracking: reentrancyCalls counter for assertion verification
}
```

#### Adversarial Oracle (manipulation)
```solidity
contract AdversarialOracle {
    // Implements the protocol's oracle interface exactly
    // Exposes setter functions: setPrice(), setQuote(), setQuoteFromTick()
    // Can be configured to return: normal prices, extreme prices, stale timestamps, or revert
    // Keep state in simple mappings — no complex logic
}
```

**Rules for attack contracts:**
- Keep them minimal — only implement what the POC needs
- Include an `arm()` function to configure attack parameters for reentrancy contracts
- Include a disarm mechanism (set `armed = false` after first re-entry) to prevent infinite loops
- Track re-entry count so assertions can verify behavior
- Follow the same Solidity version (`pragma`) as the project
- For oracles: implement the exact interface the protocol imports (read the interface file first)

### Step 6: Generate the test contract

For each finding, generate a test function that follows this structure:

```solidity
/// @notice [Finding #N] <title>
/// @dev Validates: <what the POC proves>
///      Expected: <revert | succeed with profit | state corruption>
function test_poc_finding_N_<descriptive_name>() public {
    // 1. SETUP — deploy attack contracts, configure state
    //    Reuse base contract helpers where possible
    
    // 2. PRECONDITIONS — create the vulnerable state
    //    (e.g., create an order that will be attacked)
    
    // 3. SNAPSHOT — record balances/state before attack
    
    // 4. ATTACK — execute the exploit
    //    Use vm.prank() for attacker context
    //    Use vm.expectRevert() if the attack should fail
    
    // 5. ASSERTIONS — verify the outcome
    //    Did the attacker profit? Did state corrupt?
    //    Compare pre/post balances
    //    Check invariants that should hold
}
```

**Test naming convention**: `test_poc_finding_<N>_<snake_case_description>()`

#### Directionality rules for swap/DEX protocols

**This is critical and easy to get wrong.** Before writing any oracle manipulation or price-related POC, work out the directionality on paper first:

For a swap protocol with `zeroForOne` direction:
- `zeroForOne=true` means: user **sells** token0, **buys** token1
- The oracle price (e.g., `sqrtPriceX96 = sqrt(token1/token0)`) determines the "fair" exchange rate
- **User is disadvantaged** when the oracle says the user's input token is worth MORE than the order price implies
  - For `zeroForOne=true`: a **higher** oracle price (higher tick) means token0 is more valuable → user overpaid
  - For `zeroForOne=false`: a **lower** oracle price (lower tick) means token1 is more valuable → user overpaid
- **Filler is disadvantaged** when the oracle says the user got a better deal than fair
  - This is the reverse direction from above

**Before writing the test:** write a 1-line comment: "Oracle at tick X means token0 is worth Y relative to token1, so the user [overpaid/underpaid]." If you can't write this comment confidently, read the `_computeInputForOutputAtSqrtPrice` function to understand the math.

For USD-denominated oracle manipulation:
- `inputPriceX18 > outputPriceX18` (adjusted for decimals) means user's input is worth more → user overpaid
- The ratio determines the surplus magnitude

#### Assertion strategy

- **Use percentages of the available amount**, not absolute values. This makes assertions robust across different swap sizes:
  ```solidity
  assertLt(fillerReceived, inputAvailable * 80 / 100, "filler lost >20%");
  assertGt(userRebate, inputAvailable * 10 / 100, "user rebate >10%");
  ```
- **Always verify the accounting invariant** — all tokens must be accounted for:
  ```solidity
  assertEq(fillerReceived + userRebate + protocolSurplus, inputAvailable, "all input accounted for");
  ```
- **Emit intermediate values** for debugging. When a test fails, these logs (visible with `-vvv`) are the fastest way to diagnose:
  ```solidity
  emit log_named_uint("Input available", inputAvailable);
  emit log_named_uint("Filler received", fillerReceived);
  emit log_named_uint("User rebate", userRebate);
  ```

#### Always include a control test

For every adversarial POC, write a matching **control test** that uses honest/fair parameters. This proves:
1. The baseline behavior is correct (filler gets ~100% with fair oracle)
2. The adversarial test's deviation is caused by the attack, not a setup bug

```solidity
function test_poc_control_fair_<description>() public {
    // Same setup as adversarial test, but with honest oracle/parameters
    // Assert: filler gets ~100%, no surplus drained
}
```

#### Edge case tests

For oracle manipulation findings, also test boundary values:
- Maximum and minimum possible oracle values (e.g., `TickMath.MAX_TICK`, `TickMath.MIN_TICK`)
- Zero prices
- Stale timestamps

**Important**: edge cases are directional too. Work out what each extreme value means for the specific swap direction before asserting.

#### Asymmetric enforcement patterns

Some protocols enforce surplus capture only in one direction (e.g., user-side but not filler-side). When writing POCs:
- Check whether `preview.active` is true for both directions or only one
- If filler-side is "informational only" (not enforced), the adversarial oracle POC should test the user-disadvantaged direction
- Write a separate test confirming filler-side detection is informational (no tokens redirected)

### Step 7: Generate the test file

Combine all POCs into a single test file. Structure:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity <same as project>;

// Imports — match existing test conventions
import {Test} from "forge-std/Test.sol";
// ... existing base contracts, deployers, types

// ============================================================
// Attack Contracts
// ============================================================

/// @notice Attack contract for Finding #N: <title>
contract <AttackContractName> {
    // ...
}

// ============================================================
// POC Tests
// ============================================================

/// @title POC Validation Tests
/// @notice Generated from entrypoint-analysis findings
contract PocTest is <BaseTestContract> {
    // ... setup if needed beyond base

    // ================================================================
    // Finding #N: <title>
    // ================================================================

    /// @notice Finding #1: <title>
    function test_poc_finding_1_<name>() public {
        // ...
    }

    // ================================================================
    // Control tests
    // ================================================================

    /// @notice Control: fair parameters, no exploit
    function test_poc_control_<name>() public {
        // ...
    }
}
```

## Phase 4: Validation

### Step 8: Run the tests

Execute the generated test file with Forge. **Always use `--no-cache`** to ensure recompilation picks up new/changed test files:

```bash
forge test --match-path test/<TestFile>.t.sol --no-cache -vvv
```

If Forge reports "No tests found" or "No files changed, compilation skipped", the file was not picked up. Use `--no-cache` or `--force` to force recompilation.

For each test:
- If it **passes**: The POC confirms the finding's assessment (exploitable, or safely mitigated)
- If it **reverts unexpectedly**: Debug — read the revert reason from the `-vvv` trace output, check setup, fix the POC
- If it **fails assertions**: The finding may be wrong — re-analyze

### Step 9: Debug failing tests

When a test fails, follow this procedure:

1. **Read the revert reason** from the `-vvv` output. Common issues:
   - `"SLIPPAGE"` — set `minAmountOut = 0` in setup swaps
   - `"UNKNOWN_POOL"` — pool not initialized or wrong poolId
   - `"ORDER_ALREADY_FILLED"` — order already consumed, check orderId derivation
   - `"FILL_AMOUNT_TOO_SMALL"` — fillAmount must be >= 50% of remaining
   - Missing `approve()` — filler must approve the hook for ERC-20 transfers
   - Missing `vm.prank()` — wrong `msg.sender` context

2. **Check directionality** — if an assertion like "user should appear disadvantaged" fails, the oracle price direction is probably wrong. Re-read the directionality rules above and swap the oracle values.

3. **Check the preview** — if `preview.active` is false when you expect true, log the preview fields:
   ```solidity
   emit log_named_uint("claimShare", preview.claimShare);
   emit log_named_uint("fairShare", preview.fairShare);
   emit log_named_uint("deviationBps", preview.deviationBps);
   ```
   Common causes: oracle is stale (`maxAge` exceeded), deviation is within `maxDeviationBps` tolerance, or the filler is disadvantaged (detection-only, `active=false`).

4. **Fix and re-run** with `--no-cache`. Repeat up to 3 times before marking as "needs manual review".

### Step 10: Interpret results

For each POC, produce a verdict:

```
## POC Results

| Finding | Test | Result | Verdict |
|---|---|---|---|
| #1 HIGH — Reentrancy via ERC-20 | test_poc_finding_1_reentrancy_erc20 | PASS (re-entry reverted) | MITIGATED by CEI — downgrade to LOW |
| #2 HIGH — Oracle manipulation | test_poc_finding_2_oracle_manipulation | PASS (surplus skewed) | CONFIRMED — filler lost 67% of input share |
```

Verdicts:
- **CONFIRMED**: The finding is exploitable. The POC demonstrates profit, state corruption, or invariant violation. Include the quantitative impact (e.g., "filler lost 67% of input share").
- **MITIGATED**: The attack is attempted but fails due to existing defenses. The finding's severity should be downgraded. Note which defense stopped it (e.g., "CEI pattern — state updated before external call").
- **PARTIALLY CONFIRMED**: The attack succeeds under specific conditions (e.g., only with certain token types, only if governance is compromised). State the preconditions clearly.
- **INVALID**: The POC could not reproduce the scenario. The finding may be incorrect.

## Output

Present the full output in this order:
1. **Findings Being Validated** — table of findings with severity and POC type
2. **Generated Test File** — full path and contents written
3. **Forge Output** — test results with pass/fail per POC, including log output for confirmed findings
4. **Verdict Table** — finding-by-finding assessment with:
   - Quantitative impact (% of tokens drained, gas cost, etc.)
   - Preconditions required for exploitation
   - Which defense (if any) mitigates it
   - Severity adjustment recommendation
5. **Recommendations** — for each confirmed finding, suggest a concrete fix with code. For each mitigated finding, note what defense is working and whether defense-in-depth is warranted.
