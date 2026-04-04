# Shared Agent Rules

## Reading

Use LSP tools (documentSymbol, outgoingCalls, incomingCalls, goToDefinition) as the primary navigation method. Use Grep for patterns LSP cannot resolve (low-level calls, state mutation patterns, callback patterns). Use Read for context when LSP points you to a location.

When matching function names, check both `functionName` and `_functionName` (Solidity internal convention).

## Cross-contract patterns

When you find a bug in one contract, **weaponize that pattern across every other contract in scope.** Search by function name AND by code pattern. Finding an unsanitized input in `ContractA.fill()` means you check every other contract's fill-like functions.

After scanning: escalate every finding to its worst exploitable variant (DoS may hide fund theft). Then revisit every function where you found something and attack the other branches.

## Directionality (DEX protocols)

For swap protocols with `zeroForOne` direction:
- `zeroForOne=true`: user sells token0, buys token1
- Oracle sqrtPriceX96 = sqrt(token1/token0)
- **Higher** oracle tick → token0 more valuable → user overpaid (user-disadvantaged)
- **Lower** oracle tick → token0 cheaper → filler overpaid (filler-disadvantaged)
- Always work out which direction makes which party disadvantaged BEFORE writing findings

## Do not report

Admin-only functions doing admin things. Standard DeFi tradeoffs (MEV, rounding dust). Self-harm-only bugs. "Admin can rug" without a concrete extraction mechanism.

## Output

Return structured blocks only — no preamble, no narration.

FINDINGs have concrete, unguarded, exploitable attack paths. LEADs have real code smells with partial paths — default to LEAD over dropping.

**Every FINDING must have a `proof:` field** — concrete values, traces, or state sequences from the actual code. No proof = LEAD, no exceptions.

**One vulnerability per item.** Same root cause = one item. Different fixes needed = separate items.

```
FINDING | contract: Name | function: func | line: N | bug_class: kebab-tag | group_key: Contract | function | bug-class
path: caller → function → state change → impact
proof: concrete values/trace demonstrating the bug
description: one sentence
fix: one-sentence suggestion

LEAD | contract: Name | function: func | line: N | bug_class: kebab-tag | group_key: Contract | function | bug-class
code_smells: what you found
description: one sentence explaining trail and what remains unverified
```

The `group_key` enables deduplication: `ContractName | functionName | bug_class`.
