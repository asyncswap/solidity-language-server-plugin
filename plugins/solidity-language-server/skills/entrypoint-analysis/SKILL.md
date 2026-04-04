---
name: entrypoint-analysis
description: Enumerate state-changing entrypoints in Solidity contracts, trace user-supplied parameters into state mutations and external calls, and flag unsanitized inputs. Use when auditing attack surface, reviewing access control, checking input validation, or scoping a security review.
argument-hint: <file-path> [function-name] [--scope state-changing|all]
allowed-tools: LSP Grep Glob Read
---

# Solidity Entrypoint & Unsanitized Input Analysis

Security-focused analysis of how user-controlled inputs flow through state-changing entrypoints into sensitive sinks (storage writes, external calls, token transfers, access control decisions).

## Input

- `$ARGUMENTS` — Parse the arguments:
  - **file-path** (required): Path to the Solidity file to analyze
  - **function-name** (optional): Specific function to analyze. If omitted, analyze all state-changing `external` and `public` functions.
  - **--scope** (optional): `state-changing` (default) or `all` (includes view/pure).

## Phase 1: Entrypoint Enumeration (LSP + Grep)

### Step 1: Discover all functions

Use `LSP documentSymbol` on the file to get all function symbols with their line numbers, visibility, and mutability.

### Step 2: Classify entrypoints

For each `external` or `public` function, read the function signature (2-3 lines) and classify:

- **Visibility**: `external`, `public`
- **Mutability**: state-changing (`payable`, default) vs read-only (`view`, `pure`)
- **Access control**: Check for modifiers (`onlyOwner`, `onlyPoolManager`, custom `require`/`if-revert` on `msg.sender` in the first 5 lines of the body)
- **Payable**: Whether it accepts ETH

Filter to state-changing functions unless `--scope all` was specified.

### Step 3: Map parameters

For each entrypoint, list every parameter including implicit ones:
- Named parameters from the function signature
- `msg.sender` — who is calling
- `msg.value` — ETH attached (if `payable`)
- `calldata`/`hookData` — opaque bytes that get decoded inside

Output a table:

```
## Entrypoints

| Function | Line | Visibility | Payable | Access Control | Parameters |
|---|---|---|---|---|---|
| swap() | 266 | external | yes | none (permissionless) | key, zeroForOne, amountIn, tick, minAmountOut, deadline, msg.value |
| fill() | 758 | external | yes | none (permissionless) | order, zeroForOne, fillAmount, msg.value |
```

## Phase 2: State Mutation Mapping (Grep + LSP)

### Step 4: Find all state mutations in entrypoint call chains

For each entrypoint, use `LSP outgoingCalls` (depth 2) to identify all internal functions called. Then grep the entrypoint body AND each called internal function for state mutation patterns:

```
Patterns to grep in the target file (restrict to function line ranges):
  \w+\[.*\]\s*=            — mapping/array write
  \w+\[.*\]\s*\+=          — mapping/array increment
  \w+\[.*\]\s*\-=          — mapping/array decrement
  delete\s+\w+             — storage deletion
  \w+\s*=\s*(?!.*==)       — state variable assignment (exclude comparisons)
  \.push\(                 — array push
  \.pop\(                  — array pop
  emit\s+\w+               — events (not mutations, but mark state-change intent)
```

For each mutation found:
1. Identify which state variable is being written
2. Trace backwards: which parameters flow into the written value?
3. Note whether the value is validated before the write

Output per entrypoint:

```
### swap() — State Mutations

| State Variable | Operation | Line | Derived From | Validated? |
|---|---|---|---|---|
| orderDeadline[orderId][zeroForOne] | write | 287 | deadline (param) | yes — checked > block.timestamp |
| balancesIn[orderId][zeroForOne] | += | 713 | amountIn (param) - feeAmount | yes — amountIn > 0 |
| balancesOut[orderId][zeroForOne] | += | 718 | _computeAmountOut(netInput, tick) | no — tick unchecked (by design) |
```

### Step 5: Find external calls in entrypoint chains

For each entrypoint, combine LSP outgoing calls with grep for low-level calls (`.call{`, `.call(`, `.staticcall(`, `.delegatecall(`). For each external call:

1. Is the **target address** derived from user input or from trusted storage?
2. Is the **call data** derived from user input?
3. Is the **call value** (ETH) derived from user input?
4. Is the call result checked?
5. Can the target re-enter?

Output per entrypoint:

```
### fill() — External Calls

| Call | Line | Target Source | Data Source | Value Source | Checked | Re-entrant? |
|---|---|---|---|---|---|---|
| token.call(transferFrom) | 740 | Currency.unwrap(outputCurrency) — from storage | filler (msg.sender), recipient (from order), amount (from state) | none | yes + balance | YES |
| POOL_MANAGER.transfer() | 832 | immutable POOL_MANAGER | order.swapper, inputCurrency.toId(), preview.userShare | none | trusted | no |
```

## Phase 3: Taint Analysis — Parameter to Sink

### Step 6: Trace each parameter to its sinks

For each user-supplied parameter in each permissionless entrypoint, trace forward through the function body and called internal functions to answer:

1. **Does it reach a state write?** Which variable, what transformation?
2. **Does it reach an external call?** As target address, call data, or value?
3. **What validation gates exist between the parameter and the sink?**
4. **Can the validation be bypassed?**

Classification for each parameter→sink path:

- **SANITIZED**: Parameter is validated with `require`/`if-revert` or bounded by trusted state before reaching the sink.
- **BOUNDED**: Parameter is not explicitly checked but is constrained by design (e.g., content-addressed orderId — wrong input → cache miss → revert).
- **UNCHECKED**: Parameter reaches the sink with no validation. Determine if this is by design or a bug.
- **PARTIAL**: Some paths are checked, others are not.

Output a taint flow table per entrypoint:

```
### swap() — Parameter Taint Flows

| Parameter | Sink | Path | Validation | Classification |
|---|---|---|---|---|
| tick | balancesOut (via _computeAmountOut) | swap → _processOrder → _computeAmountOut | minAmountOut slippage check | BOUNDED — user picks price, minAmountOut is backstop |
| key | pools[key.toId()] lookup | swap → router → beforeSwap → _processOrder | POOL_MISMATCH revert | SANITIZED — invalid pool reverts entire tx |
| amountIn | balancesIn write | swap → router → beforeSwap → _processOrder | require(amountIn > 0) | SANITIZED |
| msg.value | ETH settlement | swap → router → _settleExactInput | msg.value == amountIn | SANITIZED |
```

### Step 7: Cross-entrypoint state dependencies

Check whether state written by one entrypoint is read by another without re-validation:

- State written by `swap()` (balancesIn, balancesOut, orderDeadline, feeRemaining)
- Read by `fill()` and `cancelOrder()` — are the reads safe given any possible write values?

Flag cases where:
- Entrypoint A writes an unbounded value that Entrypoint B trusts without re-checking
- A storage slot can be manipulated by a permissionless caller to affect another user's execution

## Phase 4: Output Report

### Step 8: Assemble the report

Present the full analysis in this order:

1. **Entrypoints Table** — all state-changing entrypoints with access control (Phase 1, Step 3)
2. **State Mutations Per Entrypoint** — what each function writes and where values come from (Phase 2, Step 4)
3. **External Call Surface Per Entrypoint** — all outbound calls with target/data/value provenance (Phase 2, Step 5)
4. **Parameter Taint Flows** — each parameter traced to its sinks with validation classification (Phase 3, Step 6)
5. **Cross-Entrypoint Dependencies** — state coupling between entrypoints (Phase 3, Step 7)
6. **Findings Summary** — a prioritized list of findings

### Step 9: Findings summary

Produce a summary table of all findings, ordered by severity:

```
## Findings

| # | Severity | Entrypoint | Parameter/State | Finding | Classification |
|---|---|---|---|---|---|
| 1 | HIGH | fill() | outputCurrency (token.call) | Arbitrary ERC-20 transferFrom after state mutation — reentrancy possible | UNCHECKED target |
| 2 | MED | swap() | tick | User-chosen price with no oracle validation at swap time | BOUNDED (minAmountOut) |
| 3 | INFO | cancelOrder() | order.swapper | Controlled by caller but content-addressed — forgery produces revert | BOUNDED |
```

Severity classification:
- **CRITICAL**: Unsanitized input directly controls a state write or external call target with no validation, enabling fund theft or protocol corruption.
- **HIGH**: Input reaches a sensitive sink with incomplete validation. Exploitation requires specific conditions but is plausible.
- **MEDIUM**: Input is validated but the validation is weak, bypassable under edge cases, or relies on external trust assumptions (oracles, governance).
- **LOW**: Input reaches a sink but is bounded by design. Exploitation requires breaking a fundamental invariant.
- **INFO**: Noted for completeness. Input is unchecked but benign by design (user controls their own parameters).

## Phase 5: Security Review Report

### Step 10: Generate the security review report

After all analysis phases are complete, produce a structured security review report that consolidates everything into an auditor-readable document. The report should be self-contained — a reader who has not seen the intermediate tables should understand the contract's attack surface.

Format:

```
# Security Review: <ContractName>

**File**: <file-path>
**Solidity**: <pragma version>
**Inherits**: <parent contracts>
**Date**: <today>

## 1. Executive Summary

<2-3 sentence overview: what the contract does, how many state-changing entrypoints it exposes, the most significant risks found.>

## 2. Attack Surface Overview

<Total entrypoints, how many are permissionless vs access-controlled, whether the contract is payable, whether it holds tokens/ETH, whether it has callback re-entry points.>

### 2.1 Entrypoints

<The entrypoints table from Phase 1.>

### 2.2 Trust Boundaries

<List each trust boundary: who can call what, what external contracts are trusted (PoolManager, oracles, tokens), what addresses are user-controlled.>

## 3. Input Validation Analysis

For each permissionless entrypoint, present:

### 3.N <functionName>()

**Parameters**: <list with types>
**Access**: <permissionless | role-gated>

#### State Mutations
<State mutations table from Phase 2, Step 4>

#### External Calls
<External calls table from Phase 2, Step 5>

#### Parameter Taint Flows
<Taint flow table from Phase 3, Step 6>

#### Assessment
<1-3 sentences: is this entrypoint safe? What is the worst-case scenario for a malicious caller?>

## 4. Cross-Entrypoint State Coupling

<Cross-entrypoint dependency analysis from Phase 3, Step 7. Focus on: can a permissionless caller manipulate state that affects another user's execution path?>

## 5. Findings

<Findings table from Phase 4, Step 9, sorted by severity.>

For each HIGH or CRITICAL finding, add a detailed section:

### Finding N: <title>

- **Severity**: <CRITICAL|HIGH|MEDIUM|LOW|INFO>
- **Entrypoint**: <function>
- **Parameter/State**: <what is unsanitized>
- **Sink**: <where it ends up>
- **Impact**: <what an attacker can achieve>
- **Proof sketch**: <step-by-step attack scenario, or why exploitation is infeasible>
- **Recommendation**: <fix suggestion, if applicable>

## 6. Appendix: Raw Data

<Link back to the intermediate tables (state mutations, external calls, taint flows) for reference. Include them inline if the report is standalone.>
```

**Important formatting rules:**
- Use markdown throughout. Tables must be properly aligned.
- Every finding must have a proof sketch — either an attack scenario or an argument for why it is safe.
- Do not include findings that are purely informational noise. INFO-level findings should teach the reader something non-obvious about the contract's design.
- If the contract uses a content-addressed or commit-reveal pattern that makes input forgery revert, explicitly call this out as a design pattern in the Executive Summary.
