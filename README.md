<div align="center">

# 🧬 self-evolve

**An autonomous self-improvement skill for Claude**
*Invoke once. Let it work for hours. Get production-ready code.*

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Skill](https://img.shields.io/badge/Claude-Skill-orange.svg)](https://claude.ai)

</div>

---

## What Is This?

`self-evolve` is a Claude skill that transforms the AI into a **self-directed senior engineering agent**. Once activated, Claude autonomously:

1. Audits your codebase across 6 engineering dimensions
2. Prioritizes and fixes issues in dependency order
3. Validates every fix with tests
4. Accumulates lessons learned across iterations
5. Keeps iterating — without you re-prompting — until the project converges

No babysitting. No follow-up prompts. Just invoke the skill and come back to improved code.

---

## How It Works

```
Bootstrap → Audit → Diagnose → Patch → Test → Reflect → (repeat)
```

| Depth | Scope |
|-------|-------|
| **Micro** | Bugs, missing guards, typos, trivial refactors |
| **Meso** | Module refactoring, test coverage, API contracts |
| **Macro** | Architecture, performance, security model |

Exit conditions:
- ✅ **CONVERGED** — score ≥ 9/10, zero critical/high issues
- 🚧 **BLOCKED** — one specific user decision is needed
- 📍 **CHECKPOINT** — context window limit reached (state saved for resumption)

---

## The 6 Audit Lenses

| Lens | What Gets Checked |
|------|------------------|
| **Correctness** | Logic errors, wrong assumptions, race conditions |
| **Robustness** | Unhandled errors, input validation, resource leaks |
| **Performance** | Algorithmic complexity, I/O patterns, memory usage |
| **Security** | Injection risks, secrets, auth gaps, unsafe ops |
| **Maintainability** | Coupling, dead code, missing types, naming |
| **Testability** | Coverage gaps, flaky tests, untested branches |

---

## Key Features

- **Zero re-prompting** — The agent continues autonomously between iterations
- **Context-first** — Reads project state from conversation before touching filesystem
- **Dependency-aware patching** — Fixes applied in topological order
- **Regression shield** — Any newly failing test fixed before iteration ends
- **Lessons learned ledger** — Accumulates knowledge to avoid repeating mistakes
- **Token budget management** — CHECKPOINT protocol for context resumption
- **Ecosystem-aware** — Python, TypeScript, Rust, Go, Shell, SQL

---

## Installation

1. Download [`self-evolve/SKILL.md`](self-evolve/SKILL.md)
2. Open [claude.ai](https://claude.ai) → Settings → Skills → Upload

---

## Usage

```
"Self-evolve my current project"
"Review and iteratively improve this codebase"
"Run a full audit and keep optimizing until it's production-ready"
"Evolve this code — I'll check back in an hour"
```

---

## License

MIT — use freely, modify freely, share freely.

---

<div align="center">
Made for the Claude skills ecosystem · Contributions welcome
</div>
