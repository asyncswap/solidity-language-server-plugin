# Report Formatting

## Report Path

Save the report to `assets/findings/{project-name}-lsp-security-review-{timestamp}.md` where `{project-name}` is the repo root basename and `{timestamp}` is `YYYYMMDD-HHMMSS` at scan time.

## Output Format

````
# Security Review — <ContractName or repo name>

---

## Scope

|                    |                                                        |
| ------------------ | ------------------------------------------------------ |
| **File(s)**        | `File1.sol` · `File2.sol`                              |
| **Solidity**       | 0.8.x                                                  |
| **Inherits**       | `ParentA` · `ParentB`                                  |
| **Agents**         | call-analysis · entrypoint-analysis · poc-generator     |
| **Date**           | YYYY-MM-DD                                              |

---

## Executive Summary

<2-3 sentences: what the contract does, how many permissionless entrypoints it exposes, total findings by severity, and the single most important risk.>

---

## Findings

[95] **1. <Title>**

`ContractName.functionName` · Line N · Confidence: 95

**Description**
<The vulnerable code pattern and why it is exploitable, in 1-2 short sentences.>

**Attack Path**
```
caller → function → state change → external call → impact
```

**Proof**
<Concrete values, traces, or POC test results demonstrating the bug. Include filler/user balance changes if applicable.>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```

---

[82] **2. <Title>**

`ContractName.functionName` · Line N · Confidence: 82

**Description**
<1-2 sentences.>

**Attack Path**
```
caller → function → state change → impact
```

**Proof**
<Concrete values or POC results.>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```

---

< ... all findings with confidence >= 80, each with Description + Attack Path + Proof + Fix >

---

[75] **3. <Title>**

`ContractName.functionName` · Line N · Confidence: 75

**Description**
<1-2 sentences.>

**Attack Path**
```
path
```

---

< ... all findings with confidence < 80, description + attack path only, no Fix block >

---

## Findings List

| # | Confidence | Title | Entrypoint | POC |
|---|---|---|---|---|
| 1 | [95] | <title> | `function()` | CONFIRMED |
| 2 | [82] | <title> | `function()` | MITIGATED |
| 3 | [75] | <title> | `function()` | — |

---

## Leads

_Vulnerability trails with concrete code smells where the full exploit path could not be completed. These are not false positives — they are high-signal leads for manual review. Not scored._

- **<Title>** — `Contract.function` — Code smells: <missing guard, unsafe arithmetic, etc.> — <1-2 sentence trail description>
- **<Title>** — `Contract.function` — Code smells: <...> — <1-2 sentence description>

---

## Entrypoint Surface

| Function | Line | Access | Payable | Parameters | Risk |
|---|---|---|---|---|---|
| `swap()` | 266 | permissionless | yes | key, amountIn, tick, ... | <highest finding severity> |
| `fill()` | 758 | permissionless | yes | order, fillAmount, ... | <highest finding severity> |

---

## External Call Surface

| Call | Depth | Parent | Target Type | Checked | Risk |
|---|---|---|---|---|---|
| `.call(transferFrom)` | 2 | `_fill → _deliverOutput` | [unresolved-token] | yes + balance | HIGH |
| `oracle.getPrice()` | 2 | `_fill → previewUsdSurplus` | [external-untrusted] | try/catch | MED |

---

> This review was performed by an AI assistant using LSP-augmented analysis (call hierarchy, symbol resolution, taint tracking). AI analysis cannot verify the complete absence of vulnerabilities. Team security reviews, bug bounty programs, and on-chain monitoring are strongly recommended.
````

## Rules

- Follow the template above exactly.
- **Only POC-validated findings appear in the Findings section.** Every finding MUST have a passing Foundry test as proof. No POC = Lead, no exceptions.
- Sort findings by confidence (highest first).
- Findings with confidence >= 80 get full treatment: Description + Attack Path + Proof + Fix.
- Findings with confidence < 80 (e.g., MITIGATED or PARTIALLY CONFIRMED by POC) get Description + Attack Path only.
- The **Proof** field must cite the test function name, the forge output (PASS/FAIL), and quantitative results (token amounts, percentages).
- The Entrypoint Surface and External Call Surface tables come from call-analysis and entrypoint-analysis agents — include them as appendices.
- Draft findings directly in report format — do not re-generate.
- Issues that look serious but could not be validated with a POC go in Leads with a note: "POC attempted — [reason it could not confirm]".
