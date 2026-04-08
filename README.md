# Rotifer Protocol Specification

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC_BY_SA_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)
[![Status: Frozen](https://img.shields.io/badge/Status-Frozen-blue.svg)]()

The Rotifer Protocol is an **open-source evolution framework for AI agents**. It defines how software "genes" — discrete, composable units of capability — are discovered, composed, executed, evaluated, and evolved through competitive natural selection in an Arena environment.

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Gene** | Atomic unit of agent capability with a typed phenotype |
| **Arena** | Competitive environment where genes are evaluated via fitness function F(g) |
| **Agent** | Autonomous entity assembled from a genome (set of genes) |
| **Binding** | Platform-specific implementation of the protocol |
| **URAA** | Universal Rotifer Autonomous Architecture — 5-layer stack (L0–L4) |

## Specification

The executive summary is available in [`rotifer-protocol-specification.md`](./rotifer-protocol-specification.md) — covering the core architecture, design principles, and key abstractions.

**Status:** Frozen — changes are now driven solely by implementation feedback, proposed through the ADR process.
**Full specification:** Available upon request — contact dev@rotifer.dev or file an issue.

## Architecture (URAA Five-Layer Stack)

```
L4 ── Collective Immunity ── Species Memory
L3 ── Competition & Exchange ── Selection & Transfer
L2 ── Calibration ── Immune System
L1 ── Synthesis ── Protein Synthesis
L0 ── Kernel ── Genetic Code
```

## Related Repositories

| Repository | Description |
|-----------|-------------|
| [**rotifer-playground**](https://github.com/rotifer-protocol/rotifer-playground) | CLI tool for gene development, Arena competition, protocol simulation |
| [**rotifer-papers**](https://github.com/rotifer-protocol/rotifer-papers) | Published articles and philosophy whitepaper |

## Contributing

The specification is open under CC BY-SA 4.0. To propose changes:

1. Read the current specification thoroughly
2. File an issue describing the motivation and proposed change
3. Reference relevant ADRs from [public ADRs](https://github.com/rotifer-protocol/rotifer-playground/blob/main/docs/architecture-decisions.md)

## License

This specification is licensed under [Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/).

## Links

- **Website:** [rotifer.dev](https://rotifer.dev)
- **Playground:** [npm install -g @rotifer/playground](https://www.npmjs.com/package/@rotifer/playground)
