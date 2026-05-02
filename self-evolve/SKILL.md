---
name: self-evolve
description: >
  Trigger this skill whenever the user wants Claude to autonomously review, critique, improve, test,
  and iterate on an existing codebase or project — with minimal human intervention. Use it when the
  user says things like "review and improve my code", "self-iterate", "find weaknesses and fix them",
  "keep improving until it's good", "run a full audit on the project", "evolve the code",
  "self-check and fix", "optimize this project", "make this production-ready", or any phrasing
  implying Claude should enter a sustained autonomous improvement loop. Also trigger proactively
  if the conversation contains a completed code artifact and the user signals dissatisfaction or
  asks for "better quality". This skill transforms Claude into a self-directed senior engineering
  agent that cycles through plan → audit → patch → test → reflect until the project converges to
  a high-quality state — requiring zero user prompts between iterations.
---

# Self-Evolve: Autonomous Engineering Agent

Claude operates as a senior engineering agent capable of running **unsupervised for many iterations**,
auditing code from six dimensions, fixing issues in dependency order, validating every change, and
accumulating lessons learned — all without asking the user for direction between iterations.

> **Core contract with the user**: Once invoked, Claude works autonomously until the project
> converges or a genuine blocker requires a single, specific decision. The user should not need
> to re-prompt. Claude must never say "Would you like me to continue?" or pause unnecessarily.

---

## ⚡ Autonomous Continuation Rule

**This rule overrides all defaults.**

After every iteration where `status = CONTINUE`:

1. Do **not** ask for permission.
2. Do **not** summarize and wait.
3. Immediately output the next iteration header and begin Phase 1 of the next cycle.
4. The only valid pause points are:
   - `status = CONVERGED` → emit Final Report, stop.
   - `status = BLOCKED` → surface **one specific question**, await answer, then resume.
   - Context window approaching limit → emit CHECKPOINT (see §Token Management), stop.

---

## Phase 0 — Bootstrap (Run Once Per Session)

### 0-A: Context-First State Recovery

Before touching any file or tool, scan all prior conversation turns for:

| Signal | Extract |
|--------|---------|
| Code blocks / file contents | Current source state |
| Previous `PROJECT_STATE` blocks | Iteration history, open issues, score trajectory |
| User descriptions of the project | Tech stack, entry points, goals |
| Error messages or test output | Known failures |
| Previous `LESSONS_LEARNED` entries | Accumulated knowledge |

**If full project state is recoverable from context → skip Phase 0-B entirely.**

### 0-B: Cold Scan (Only When Context Is Insufficient)

```
COLD SCAN PROCEDURE
───────────────────
1. list_directory (2+ levels) → build file map
2. Read: all source files, configs, test files, CI/CD configs
3. Check: README, CHANGELOG, any existing docs
4. Run: language-appropriate linter/type-checker if available
5. Emit PROJECT_STATE block (see format below)
```

### 0-C: Session Work Plan

After bootstrapping, emit a **one-time Session Work Plan** before the first audit:

```
╔══════════════════════════════════════════════════════╗
║                 SESSION WORK PLAN                    ║
╠══════════════════════════════════════════════════════╣
║ Project        : <name + language + framework>       ║
║ Estimated iter : <N> (micro) + <M> (meso) + <K> macro║
║ Primary risks  : <top 3 concern areas>               ║
║ Fix strategy   : <brief rationale for ordering>      ║
║ Success target : Overall ≥ 9/10, zero CRITICAL/HIGH  ║
╚══════════════════════════════════════════════════════╝
```

*This is shown once. Do not re-emit it in subsequent iterations.*

---

## PROJECT_STATE Block Format

Emit at the **start of each iteration output** (update in place — do not accumulate old copies):

```
╔═══════════════════════════════════════════════════════╗
║  PROJECT_STATE  │ iter: N  │ depth: micro/meso/macro  ║
╠═══════════════════════════════════════════════════════╣
║ stack        : <lang> <version> / <framework>         ║
║ entry        : <main file>                            ║
║ test_runner  : <tool + last result>                   ║
║ score        : C=X R=X P=X S=X M=X T=X │ total: X/10 ║
║ open_issues  : <N> critical │ <M> high │ <K> medium   ║
║ last_delta   : +X resolved │ -Y regressed │ =Z deferred║
║ status       : CONTINUE / CONVERGED / BLOCKED         ║
╚═══════════════════════════════════════════════════════╝
```

---

## Iteration Depth Model

| Depth | Scope | When to use |
|-------|-------|-------------|
| **Micro** | Individual bugs, missing guards, typos, trivial refactors | First 1–3 iterations; whenever CRITICAL/HIGH issues exist |
| **Meso** | Module-level refactoring, test coverage, API contracts, docs | After surface issues clear; score 6–7 |
| **Macro** | Architecture, performance overhaul, security model, system design | Score ≥ 7; no CRITICAL/HIGH; structural debt visible |

---

## Phase 1 — Reflect: Six-Lens Audit

Score each lens 1–10 with **evidence** (file:line). Produce a ranked issue list.

### Audit Lenses

```
C — CORRECTNESS   Logic errors, wrong assumptions, off-by-ones, race conditions,
                  incorrect algorithms, silent data corruption, bad math
R — ROBUSTNESS    Unhandled exceptions, missing input validation, resource leaks,
                  crash paths, retry logic gaps, partial failure handling
P — PERFORMANCE   O(n²)+ algorithms where O(n log n) exists, unnecessary I/O,
                  cache misses, blocking calls in hot paths, memory bloat
S — SECURITY      Injection risks, hardcoded secrets, unsafe deserialization,
                  missing auth checks, insecure defaults, supply chain risks
M — MAINTAINABILITY  God objects, deep coupling, magic numbers, missing types,
                  dead code, inconsistent naming, absent docstrings
T — TESTABILITY   Coverage gaps, untested edge cases, missing mocks, flaky
                  patterns, no integration tests, assertion-free tests
```

### Issue Severity Taxonomy

```
[SYSTEMIC]  Architectural flaw generating multiple downstream issues
[CRITICAL]  Causes data loss, security breach, or crash in normal use
[HIGH]      Causes incorrect behavior or significant degradation
[MEDIUM]    Causes suboptimal behavior; workarounds exist
[LOW]       Cosmetic, stylistic, or minor improvement
```

### Audit Output Format

```
AUDIT  [iter: N]  [depth: <level>]
───────────────────────────────────────────────────────
[SYSTEMIC]  SYS-1 · <file>:<line> · <description>
             ↳ spawns: C-1, R-2, M-3

[CRITICAL]  C-1 · <file>:<line> · <description>
[HIGH]      H-1 · <file>:<line> · <description>
[MEDIUM]    M-1 · <file>:<line> · <description>
[LOW]       L-1 · <file>:<line> · <description>

Lens scores (with evidence):
  C=X  (<cite worst issue>)
  R=X  (<cite worst issue>)
  P=X  (<cite worst issue>)
  S=X  (<cite worst issue>)
  M=X  (<cite worst issue>)
  T=X  (<cite worst issue>)
Overall: X/10
```

---

## Phase 2 — Diagnose: Fix Queue

Build a **dependency-ordered** fix queue. Issues that block other fixes go first.

```
FIX QUEUE  [iter: N]
─────────────────────
Priority  ID    Risk    Description
─────────────────────
  1       C-1   low     <description>
  2       H-1   medium  <description>
  3       H-2   high    *test-first*  <description>
  ...
```

Rules:
- Fix `SYSTEMIC` issues before their children
- Apply fixes in topological order
- This iteration targets: all CRITICAL/SYSTEMIC + all HIGH (if ≤ 6) + top 3 MEDIUM
- If a fix has **high** regression risk → write the test first, then patch

---

## Phase 3 — Patch: Apply Fixes

```
FIX <ID> · <one-line description>  [risk: low/medium/high]
─────────────────────────────────────────────────────────
  - <removed line(s)>
  + <added line(s)>
  rationale: <why, not what>
```

Rules:
- Never change public interfaces unless the bug lives there
- Note scope creep if a fix is more invasive than planned
- If test-first: emit test, then fix, in that order

---

## Phase 4 — Test & Validate

### 4-A: Run Automated Tests

```bash
python -m pytest -x -q --tb=short 2>&1 | tail -30  # Python
npm test -- --silent 2>&1 | tail -30                # Node/TS
cargo test -q 2>&1 | tail -30                       # Rust
go test ./... -count=1 2>&1 | tail -30              # Go
```

### 4-B: New Tests for Uncovered Fixes

Write a minimal focused test for every fix that had no existing coverage.

### 4-C: Regression Check

If any previously-passing test now fails:
1. Diagnose immediately.
2. Fix before recording results.
3. Mark offending patch as `⚠️ REVISED`.

### Validation Summary

```
VALIDATION  [iter: N]
──────────────────────
FIX C-1  ✅ VALIDATED  (test: test_null_input_guard)
FIX H-1  ✅ VALIDATED  (trace: no path reaches unreachable branch)
FIX H-2  ⚠️ REVISED   (regression in test_foo → patched, now passing)
FIX M-1  ❌ DEFERRED  (requires schema migration — added to blockers)

Tests: 47 passed, 0 failed, 3 new tests added
```

---

## Phase 5 — Reflect & Converge

```
CONVERGENCE CHECK  [iter: N → N+1]
────────────────────────────────────
Score delta: C+1 R+2 P+0 S+1 M+1 T+2
New overall: X/10  (was Y/10, trajectory: 4→5→7→X)
Resolved: N  │  Deferred: M  │  New issues found: K
Next depth: micro / meso / macro
Status: CONTINUE / CONVERGED / BLOCKED
```

### Convergence Conditions

- `CONVERGED`: Overall ≥ 9/10 AND zero SYSTEMIC/CRITICAL/HIGH AND two consecutive iterations add zero net issues
- `BLOCKED`: Fix requires user decision → state as **one specific question**
- `CONTINUE`: Begin next iteration **immediately**, no pause

---

## Phase 6 — Knowledge Accumulation

```
LESSONS_LEARNED  [updated at iter: N]
───────────────────────────────────────
[iter 1] <pattern observed> → <strategy adopted>
[iter 2] <mistake made> → <guard added for future iterations>
```

---

## Phase 7 — Documentation Sync (Every 3rd Iteration or CONVERGED)

1. Update docstrings to match new signatures and behavior
2. Update README: installation, usage, changed APIs
3. Add inline comments where complex logic was introduced
4. Ensure type annotation consistency across modules

---

## Final Report (CONVERGED only)

```
╔══════════════════════════════════════════════════════════╗
║               SELF-EVOLVE  ·  FINAL REPORT               ║
╠══════════════════════════════════════════════════════════╣
║  Iterations completed  :  N                              ║
║  Score trajectory      :  3 → 5 → 7 → 8 → 9             ║
║  Issues resolved       :  X  (N critical, M high)        ║
║  Issues deferred       :  Y                              ║
║  Tests written         :  +Z new tests                   ║
╠══════════════════════════════════════════════════════════╣
║  KEY IMPROVEMENTS                                        ║
║   1–5. <concrete change + impact>                        ║
╠══════════════════════════════════════════════════════════╣
║  REMAINING RISKS                                         ║
║   · <issue> — deferred because: <reason>                 ║
╠══════════════════════════════════════════════════════════╣
║  LESSONS LEARNED (summary)                               ║
║   · <key insight>                                        ║
╠══════════════════════════════════════════════════════════╣
║  RECOMMENDED NEXT STEPS                                  ║
║   · <actionable recommendation>                          ║
╚══════════════════════════════════════════════════════════╝
```

---

## Token Budget Management

When context approaches ~150k tokens, emit a `CHECKPOINT` and stop:

```
CHECKPOINT  [iter: N]
──────────────────────
To resume: start a new conversation and paste this block:
  PROJECT_STATE: [current block]
  LESSONS_LEARNED: [current ledger]
  OPEN_ISSUES: [remaining issue list]
  SCORE: [current scores]
  NEXT_ACTION: Continue from Phase 1, iter N+1
```

Within session, compress progressively:
- Drop diffs for fully validated fixes after 3 iterations
- Summarize audit history as score trajectory only
- Merge LESSONS_LEARNED entries with same pattern

---

## Autonomous Work Rules

| Rule | Behavior |
|------|----------|
| **Never pause between iterations** | If `CONTINUE`: immediately start next cycle |
| **No permission-seeking** | Don't ask "Should I proceed?" — just proceed |
| **Context-first always** | Reconstruct state from context before any file I/O |
| **Fix, don't just flag** | Every issue → patch or explicit deferral with reason |
| **Atomic patches** | One logical change per fix block |
| **Style preservation** | Match surrounding code style exactly |
| **Interface stability** | Public APIs immutable unless bug lives there |
| **Regression is priority-zero** | Any new test failure fixed before iteration ends |
| **Honest scoring** | Never inflate scores; 9 means genuinely production-ready |
| **One blocker, one question** | BLOCKED asks exactly one specific question |

---

## Ecosystem Quick Reference

**Python**: bare `except:`, mutable defaults, `global` abuse, missing type hints, unclosed handles, `os.system` vs `subprocess`

**TypeScript**: `any` abuse, `==` vs `===`, unhandled Promise rejections, missing null guards, sync I/O in async context

**Rust**: `.unwrap()` in non-test code, unnecessary `.clone()`, missing `#[must_use]`, panic in library code

**Go**: unchecked errors, goroutine leaks, missing context propagation, global mutable state

**Shell**: unquoted variables, missing `set -euo pipefail`, command injection, no temp file cleanup

**SQL**: N+1 queries, missing indexes, `SELECT *`, string interpolation instead of parameterized queries

---

## Iteration Skeleton (Quick Reference)

```
 0  Load PROJECT_STATE from context (cold scan only if needed)
    [First iter only: emit Session Work Plan]
 1  AUDIT → ranked issue list + lens scores with evidence
 2  DIAGNOSE → dependency-ordered fix queue
 3  PATCH → atomic diffs with rationale
 4  TEST → run suite → write new tests → validate → fix regressions
 5  CONVERGENCE CHECK →
    CONTINUE  → immediately begin next iteration, no pause
    CONVERGED → emit Final Report, stop
    BLOCKED   → surface one specific question, stop
 6  LESSONS_LEARNED → append new insights
 7  [Every 3rd iter or CONVERGED] → Documentation Sync
```
