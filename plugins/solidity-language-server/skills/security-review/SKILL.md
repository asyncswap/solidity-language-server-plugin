---
name: security-review
description: Orchestrated LSP-augmented security review of Solidity contracts. Spawns call-analysis, entrypoint-analysis, and poc-generator agents in parallel, deduplicates findings, validates through judging gates, and produces a formatted audit report. Use when asked to audit, review security, or do a full security analysis of Solidity files.
argument-hint: <file-path ...> [--file-output] [--skip-poc]
allowed-tools: LSP Grep Glob Read Write Edit Bash Agent
---

# Solidity Security Review (Orchestrator)

You are the orchestrator of a parallelized, LSP-augmented smart contract security review. You coordinate three specialized agents that use the Solidity Language Server for structural analysis, then merge their results into a single validated report.

## Input

- `$ARGUMENTS` — Parse the arguments:
  - **file-path** (required): One or more Solidity file paths to review. If none given, scan all `.sol` files in `src/` excluding `interfaces/`, `libraries/`, `test/`, and `*.t.sol`.
  - **--file-output** (optional, off by default): Also write the report to a markdown file.
  - **--skip-poc** (optional): Skip the POC generation phase (faster, no test execution).

## Turn 1 — Discover

Make these parallel tool calls in one message:

1. **Glob** `src/**/*.sol` to find all in-scope Solidity files (exclude interfaces/, test/).
2. **Glob** for `**/references/report-formatting.md` under the plugin directory — extract the `references/` directory path as `{ref_path}`.
3. **Read** the target file(s) pragma and inheritance line (first 40 lines) to get Solidity version and parent contracts.

Print the scope summary:

```
## Scope
Files: <list>
Solidity: <version>
Inherits: <parents>
```

## Turn 2 — Prepare References

Read in parallel:
1. `{ref_path}/report-formatting.md`
2. `{ref_path}/judging.md`
3. `{ref_path}/shared-rules.md`

These references guide the final deduplication, validation, and formatting. Do not inline them into agent prompts — you (the orchestrator) apply them in Turn 4.

## Turn 3 — Spawn Agents

Launch **three agents in parallel** using the Agent tool. Each agent runs the corresponding skill against the target file(s).

### Agent 1: Call Analysis

```
prompt: |
  Run the call-analysis skill on <file-path>.
  Analyze all external and public functions.
  Use LSP outgoingCalls with depth 3, direction outgoing. Then use LSP incomingCalls with depth 1 for callback entry points.
  Grep for low-level calls (.call, .staticcall, .delegatecall, abi.encodeWithSelector).
  Grep for callback patterns (receive, fallback, unlockCallback, on*Callback).

  For each function, produce a call tree with node classifications:
  [internal], [inherited], [external-dep], [external-untrusted], [interface], [leaf]

  Output structured blocks per shared-rules.md:
  - One FINDING block per HIGH/CRITICAL external call risk
  - One LEAD block per MEDIUM/LOW risk
  - An External Call Surface table
  - A Callback Entry Points table

  Be precise about which calls are to trusted contracts (PoolManager, known dependencies)
  vs untrusted targets (arbitrary ERC-20 tokens, user-supplied addresses, governance-configured oracles).
```

### Agent 2: Entrypoint Analysis

```
prompt: |
  Run the entrypoint-analysis skill on <file-path>.
  Scope: state-changing functions only.

  For each permissionless entrypoint:
  1. Map all parameters (including msg.sender, msg.value)
  2. Find all state mutations using grep + LSP outgoingCalls (depth 2)
  3. Find all external calls with target/data/value provenance
  4. Trace each parameter to its sinks (state writes, external calls)
  5. Classify: SANITIZED, BOUNDED, UNCHECKED, PARTIAL
  6. Check cross-entrypoint state coupling

  Output structured blocks per shared-rules.md:
  - One FINDING block per UNCHECKED parameter→sink path with material impact
  - One LEAD block per PARTIAL or BOUNDED path worth investigating
  - Entrypoints Table (function, line, access control, parameters)
  - State Mutations Table per entrypoint
  - Parameter Taint Flow Table per entrypoint

  Pay special attention to:
  - Parameters that reach external call targets (address controlled by user)
  - Parameters that reach state writes without validation
  - Asymmetric enforcement (user-side vs filler-side surplus capture)
  - Content-addressed patterns that prevent forgery (note as BOUNDED, not a finding)
```

### Agent 3: POC Generator (unless --skip-poc)

```
prompt: |
  Run the poc-generator skill for <file-path>.

  Wait to see the findings from the other agents in the conversation context.
  If findings are available, generate POC tests for all HIGH and CRITICAL findings.
  If no findings are available yet, analyze the file yourself and generate POCs for:
  - Any reentrancy surface (external calls after state mutations)
  - Any oracle-dependent value distribution
  - Any permissionless function with unchecked parameters reaching sensitive sinks

  Rules:
  - Search test/ for existing test base (SetupHook, BaseTest, etc.) and REUSE it
  - Search test/ for existing POC/reentrancy tests — don't duplicate
  - Follow directionality rules for oracle manipulation (higher tick = token0 more valuable for zeroForOne=true)
  - Always include a control test (fair parameters, no exploit)
  - Use percentage-based assertions: assertLt(received, available * 80 / 100)
  - Emit log_named_uint for intermediate values
  - Run with: forge test --match-path test/<file>.t.sol --no-cache -vvv

  Output:
  - The test file path and contents
  - Forge output (pass/fail per test)
  - Verdict per finding: CONFIRMED / MITIGATED / PARTIALLY CONFIRMED / INVALID
  - Quantitative impact (% of tokens drained, balances before/after)
```

**Agent execution strategy:**
- **Phase A**: Spawn Agents 1 and 2 in parallel (no dependency). Wait for results.
- **Phase B**: Parse FINDING and LEAD blocks from Agents 1 and 2. Extract all findings that cleared initial assessment (HIGH, CRITICAL, or MEDIUM with concrete attack path).
- **Phase C**: Spawn Agent 3 (POC Generator) with the merged findings list. Agent 3 generates and runs Foundry tests for each finding.

**The POC is the selection filter.** Only findings with a passing POC test make it into the final report as findings. Everything else becomes a lead. This means Agent 3 is NOT optional — it is the validation gate that determines what ships.

If `--skip-poc` is set, all findings are reported as leads (no confirmed findings without POC proof).

## Turn 4 — Build Report Around POC Results

The POC results are the source of truth. Everything else is supporting context.

### Step 1: Start from POC verdicts

Parse Agent 3's output. For each POC test:

| POC Verdict | Report Treatment |
|---|---|
| **CONFIRMED** (test passes, demonstrates exploit) | → FINDING in report. Cite test name, forge output, quantitative impact. |
| **MITIGATED** (test passes, attack reverts) | → FINDING at reduced confidence (note which defense holds). Valuable because it proves a defense-in-depth recommendation. |
| **PARTIALLY CONFIRMED** (passes under specific conditions) | → FINDING with preconditions clearly stated. |
| **INVALID** (could not reproduce) | → LEAD if code smell exists, otherwise drop. |

### Step 2: Enrich with Agent 1 & 2 context

For each POC-validated finding, pull supporting data from Agents 1 and 2:
- **Call tree** from Agent 1 ��� shows the full call chain from entrypoint to sink
- **Taint flow** from Agent 2 — shows which parameter reaches the vulnerable sink
- **External call classification** from Agent 1 — shows trust level of the call target
- **State mutations** from Agent 2 — shows what state changes before the external call

This context goes into the finding's **Attack Path** and **Description** fields.

### Step 3: Demote unvalidated findings to leads

Any FINDING or LEAD from Agents 1 or 2 that was NOT covered by a POC test goes to the **Leads** section. Annotate each with:
- Why a POC was not generated (e.g., "requires governance compromise to set up", "view-only call, no state impact")
- Whether it warrants manual review

### Step 4: Deduplicate

Group by `group_key` (`Contract | function | bug-class`). If multiple agents flagged the same issue AND the POC covers it, merge into one finding with the POC as the authoritative proof. Annotate with explicit agent names, e.g. `[agents: call-analysis, entrypoint-analysis]`. Never use "both" — always list agent names.

### Step 5: Compute confidence

For each finding, start at 100 and deduct per `judging.md` rules. The POC verdict is the strongest signal:
- POC CONFIRMED → no deduction for "partial attack path" (it's proven)
- POC MITIGATED → deduct 25 (defense holds, but pattern is worth noting)
- POC PARTIALLY CONFIRMED → deduct 15 (preconditions required)

### Step 6: Format report

Format per `report-formatting.md`. The report centers on POC-validated findings:

1. **Scope** table
2. **Executive Summary** — lead with the POC results: "N findings confirmed by Foundry POC tests, M mitigated, K leads for manual review"
3. **Findings** — POC-validated only, sorted by confidence descending
   - Confidence >= 80: Description + Attack Path + Proof (POC output) + Fix
   - Confidence < 80: Description + Attack Path only
4. **Findings List** — summary table with POC test name and verdict
5. **Leads** — unvalidated issues from Agents 1 & 2 that could not be POC'd
6. **Entrypoint Surface** table (from Agent 2)
7. **External Call Surface** table (from Agent 1)
8. **Disclaimer**

If `--file-output`: also write to `assets/findings/{project-name}-lsp-security-review-{timestamp}.md`.

## Banner

Before doing anything else, print this exactly:

```
 ╔═══════════════════════════════════════════════════════════╗
 ║     LSP Security Review — Solidity Language Server        ║
 ║     call-analysis · entrypoint-analysis · poc-generator   ║
 ╚═══════════════════════════════════════════════════════════╝
```
