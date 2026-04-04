---
name: call-analysis
description: Analyze incoming and outgoing calls for Solidity functions using LSP call hierarchy with recursive depth tracing and manual grep for low-level calls. Use when analyzing call graphs, tracing call chains, auditing external calls, or checking reentrancy surfaces in Solidity contracts.
argument-hint: <file-path> [function-name] [--depth N] [--direction both|incoming|outgoing]
allowed-tools: LSP Grep Glob Read
---

# Solidity Call Analysis

Perform a comprehensive call graph analysis for Solidity functions using LSP call hierarchy plus manual grep for calls the LSP cannot resolve.

## Input

- `$ARGUMENTS` — Parse the arguments:
  - **file-path** (required): Path to the Solidity file to analyze
  - **function-name** (optional): Specific function to analyze. If omitted, analyze all `external` and `public` functions in the file.
  - **--depth N** (optional): Maximum recursion depth, 1-7. Default: 3.
  - **--direction** (optional): `incoming`, `outgoing`, or `incoming,outgoing` (default: `incoming,outgoing`).

## Phase 1: LSP Call Hierarchy (Structured Calls)

For each target function, recursively trace calls using LSP.

### Step 1: Locate the target functions

Use `LSP documentSymbol` on the file to get all function symbols with their line numbers. If a specific function name was given, filter to that function. Otherwise, select all `external` and `public` functions (skip `view`/`pure` where not interesting — but include them if they call external contracts).

### Step 2: Recursive LSP tracing

For each target function, perform recursive outgoing/incoming call tracing:

```
function traceOutgoing(filePath, line, character, currentDepth, maxDepth, visited):
    if currentDepth > maxDepth: return "... (depth limit reached, may have deeper calls)"
    
    key = filePath + ":" + line
    if key in visited: return "(circular — already visited)"
    visited.add(key)
    
    results = LSP outgoingCalls(filePath, line, character)
    for each call in results:
        record: call.name, call.filePath, call.line, currentDepth
        traceOutgoing(call.filePath, call.line, call.character, currentDepth + 1, maxDepth, visited)
```

Apply the same pattern for `incomingCalls` when direction includes `incoming`.

**Important rules:**
- Track visited nodes by `filePath:line` to prevent infinite loops on circular/recursive calls.
- Stop at `maxDepth` (default 3, max 7) and note: `"... recursive calls may continue beyond depth limit"`.
- Make LSP calls in parallel where possible — batch all calls at the same depth level.
- When tracing into dependency files (e.g., `v4-core`, `openzeppelin`), continue tracing but mark them as `[external-dep]`.

### Step 3: Classify each discovered call

Tag every call node with one of:
- `[internal]` — same contract, private/internal function
- `[inherited]` — parent contract function
- `[external-dep]` — call into a dependency (OpenZeppelin, Uniswap, etc.)
- `[external-untrusted]` — call to an address that could be user-supplied or arbitrary (ERC-20 tokens, oracles, callbacks)
- `[interface]` — call through an interface (resolve to concrete type if possible)
- `[leaf]` — function has no further outgoing calls

## Phase 2: Manual Grep for Unresolved Calls

LSP cannot resolve dynamic/low-level calls. Grep the target file and all internal contract files for these patterns:

### Step 4: Grep for low-level calls

Search the target file and all files discovered in Phase 1 that are in `src/`:

```
Patterns to grep (in .sol files under src/):
  \.call\{                    — low-level call with value
  \.call\(                    — low-level call
  \.staticcall\(              — static call
  \.delegatecall\(            — delegate call
  abi\.encodeWithSelector\(   — often paired with .call()
  abi\.encodeWithSignature\(  — often paired with .call()
  abi\.encodeCall\(           — typed call encoding
```

For each match:
1. Read the surrounding context (5 lines before/after) to determine what address is being called
2. Determine if the target address is:
   - A known contract (e.g., `POOL_MANAGER`, `address(rewardToken)`) — tag `[resolved]`
   - A function parameter or storage variable that could be arbitrary — tag `[unresolved-dynamic]`
   - A `Currency.unwrap()` or similar pattern indicating a token address — tag `[unresolved-token]`
3. Trace what data is being sent (function selector if visible)
4. Note if the call result is checked

### Step 5: Grep for callback patterns

Search for patterns that indicate the contract can be called back:

```
Patterns to grep:
  receive\(\)                 — ETH receive callback
  fallback\(\)                — fallback function
  onERC721Received            — NFT callback
  unlockCallback              — Uniswap V4 callback
  on.*Callback                — generic callback pattern
```

## Phase 3: Assemble the Full Call Tree

### Step 6: Merge LSP and grep results

Combine the LSP call tree with the grep-discovered calls. Insert grep-discovered calls at the correct nesting level (the function where the `.call()` was found).

### Step 7: Output the call tree

Format the output as an indented tree with annotations:

```
functionName() [ContractName:line]
├── childCall() [internal] [ContractName:line]
│   ├── deeperCall() [external-dep] [Dependency:line]
│   │   └── LEAF (pure math)
│   └── .call{value}(transferFrom) [unresolved-token] [ContractName:line]
│       Target: Currency.unwrap(inputCurrency) — arbitrary ERC-20
│       Selector: IERC20.transferFrom.selector
│       Return: checked (reverts on failure)
├── externalCall() [external-untrusted] [OracleInterface:line]
│   └── ... (depth limit reached, may have deeper calls)
└── POOL_MANAGER.unlock() [external-dep] [IPoolManager:line]
    └── unlockCallback() [inherited] [ContractName:line]  <- callback re-entry
        ├── POOL_MANAGER.burn() [external-dep] [IPoolManager:line] -> LEAF
        └── POOL_MANAGER.take() [external-dep] [IPoolManager:line] -> LEAF
```

### Step 8: Generate the external call surface table

After the tree, produce a summary table:

```
| External Call | Depth | Parent Function | Target Type | Checked | Risk |
|---|---|---|---|---|---|
| IERC20.transferFrom() | 2 | _deliverOutput | [unresolved-token] | yes | HIGH — reentrancy |
| oracle.getPrice() | 3 | previewSurplus | [external-untrusted] | try/catch | MED — manipulation |
| .call{value}("") | 1 | withdrawNative | [resolved: param] | yes | MED — recipient |
```

Risk classification:
- **HIGH**: Calls to unresolved/untrusted addresses that can modify state or re-enter
- **MED**: Calls to untrusted addresses that are view-only or properly guarded
- **LOW**: Calls to known/trusted contracts (PoolManager, own token, etc.)
- **INFO**: Pure library calls, leaf math functions

### Step 9: Note recursion beyond depth limit

If any branch hit the depth limit, add a section:

```
## Branches Truncated at Depth Limit

The following call chains were cut off at depth {N} and may have deeper calls:
- `functionA() -> functionB() -> functionC() -> ...` (last seen in DependencyFile.sol:123)
  Recommendation: increase --depth or manually inspect DependencyFile.sol:functionC()

To trace deeper, re-run with: /call-analysis <file> <function> --depth 7
```

## Output Formatting

Present the full analysis in this order:
1. **Target Functions** — list of functions being analyzed
2. **Call Trees** — one tree per target function (Phase 3, Step 7)
3. **Unresolved Dynamic Calls** — grep results (Phase 2) integrated into trees
4. **External Call Surface** — summary table (Phase 3, Step 8)
5. **Truncated Branches** — depth limit warnings (Phase 3, Step 9)
6. **Callback Entry Points** — functions that external contracts can call back into (Phase 2, Step 5)

## Structured Output (when called by orchestrator)

When this skill is invoked as an agent by the `security-review` orchestrator, **also** append structured FINDING and LEAD blocks after the normal output. These blocks enable deduplication across agents.

For each HIGH or CRITICAL risk in the External Call Surface table:

```
FINDING | contract: Name | function: func | line: N | bug_class: kebab-tag | group_key: Contract | function | bug-class
path: caller → function → state change → external call → impact
proof: concrete call chain with node classifications
description: one sentence
fix: one-sentence suggestion
```

For each MEDIUM or LOW risk:

```
LEAD | contract: Name | function: func | line: N | bug_class: kebab-tag | group_key: Contract | function | bug-class
code_smells: what you found (e.g., unresolved-token call, missing reentrancy guard)
description: one sentence explaining the risk and what remains unverified
```
