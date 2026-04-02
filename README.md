# solidity-language-server

A Claude Code plugin that provides Solidity language server support via [solidity-language-server](https://github.com/asyncswap/solidity-language-server).

## Prerequisites

Install the Solidity language server:

```sh
curl -fsSL https://asyncswap.org/lsp/install.sh | sh
```

## Installation

1. Add the marketplace:

```
/plugin marketplace add asyncswap/solidity-language-server-plugin
```

2. Install the plugin:

```
/plugin install solidity-language-server@asyncswap
```

## Features

- **Go to Definition / Declaration** — jump to any symbol across files, including qualifier segments in qualified type paths
- **Find References** — all usages of a symbol across the project, with interface/implementation equivalence
- **Rename** — project-wide symbol rename with prepare support
- **Hover** — signatures, NatSpec docs, function/error/event selectors, `@inheritdoc` resolution
- **Completions** — scope-aware with two modes (fast cache vs full recomputation)
- **Document Links** — clickable imports, type names, function calls
- **Document Symbols / Workspace Symbols** — outline and search
- **Formatting** — via `forge fmt`
- **Diagnostics** — from `solc` and `forge lint`
- **Signature Help** — parameter info on function calls, event emits, and mapping access
- **Inlay Hints** — parameter names at call sites
- **Go to Implementation** — jump from interface/abstract declarations to concrete implementations
- **Call Hierarchy** — navigate call graphs across contracts and libraries (incoming/outgoing calls)
- **Code Actions** — quickfix engine (e.g., "Remove unused import" via forge-lint diagnostics)
- **File Operations** — template scaffolding on create, import updates on rename/delete
- **Semantic Tokens** — full, range, and delta semantic highlighting
- **Folding Ranges** — contracts, functions, structs, enums, blocks, comments, imports
- **Selection Ranges** — smart expand/shrink selection
- **Execute Commands** — `solidity.clearCache`, `solidity.reindex`
- **Update Check** — notifies when a newer version is available

## License

MIT
