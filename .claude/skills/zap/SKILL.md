---
name: zap
description: Fix a bug using competitive multi-agent investigation. 3 parallel recon agents survey the codebase, 5 agents investigate different theories, top 3 implement fixes, validates with 3 testers, retries on partial failure, then commits and creates a PR. Use when a bug is persistent, non-obvious, or has resisted prior fix attempts.
argument-hint: "<bug description> [--single-recon]"
---

# Multi-Agent Bug Fix

Fix the following bug using competitive multi-agent investigation:

**Bug:** $ARGUMENTS

## Phase 0: Reconnaissance

If `$ARGUMENTS` contains `--single-recon`, use **single-agent recon** (below). Otherwise, use **parallel recon** (default).

### Default: Parallel recon (3 agents, fast)

Spawn **3 agents in parallel**, each in an isolated worktree (`isolation: "worktree"`). Use `model: "sonnet"`.

Each agent has a distinct recon responsibility:

**Agent 1 — Code reader:**
- The bug description
- Instructions to:
  - Find and read source files likely related to the bug
  - Map the relevant code paths, entry points, and data flow
  - Identify suspicious patterns, error handling gaps, or logic issues
  - Return a **structured report**: relevant files (with brief descriptions), code path analysis, suspicious areas
- A **5-minute time budget** — breadth over depth

**Agent 2 — Git archaeologist:**
- The bug description
- Instructions to:
  - Run `git log` on files and directories likely related to the bug
  - Run `git blame` on the most suspect sections
  - Identify recent changes that could have introduced the bug
  - Check commit messages and PR descriptions for related context
  - Return a **structured report**: recent changes (commits, authors, dates, diffs), suspect commits, relevant history
- A **5-minute time budget** — breadth over depth

**Agent 3 — Test runner:**
- The bug description
- Instructions to:
  - Identify and run existing tests related to the bug area
  - Note which tests pass and which fail, with output
  - Search for related TODOs, FIXMEs, HACKs, or known-issue comments
  - Check for disabled/skipped tests that might be relevant
  - Return a **structured report**: test results (pass/fail with output), related comments and TODOs, test coverage gaps
- A **5-minute time budget** — breadth over depth

Wait for all 3 agents to complete. Merge their reports into a single **recon report**: combine relevant files, recent changes, test results, and initial observations. Deduplicate and cross-reference findings. This becomes the shared artifact that every agent in every subsequent phase receives as baseline context.

### Alternative: Single-agent recon (`--single-recon`)

Spawn **1 agent** in an isolated worktree (`isolation: "worktree"`). Use `model: "sonnet"`.

This agent does a fast survey of the codebase related to the bug. Its prompt must include:
- The bug description
- Instructions to:
  - Read files likely related to the bug
  - Run `git log` on suspect files/directories to find recent changes
  - Run existing tests if available, note which pass/fail
  - Check for related issues in comments, TODOs, or commit messages
  - Return a **structured report**: relevant files (with brief descriptions), recent changes (commits, authors, dates), test results, initial observations
- A **5-minute time budget** — breadth over depth

Wait for the agent to complete. Save its report — this becomes the **recon report**, a shared artifact that every agent in every subsequent phase receives as baseline context.

## Phase 1: Investigate (5 agents, fast)

Read the recon report. Generate **5 distinct theories** about the root cause, grounded in the recon findings. Theories should cover different layers of the stack (e.g., data flow, timing/async, state management, rendering/UI, configuration). One theory must always be a **git history theory**: "The bug was introduced by a recent change — identify which commit and why." Avoid overlapping theories.

Spawn **5 agents in parallel**, each in an isolated worktree. Use `model: "sonnet"`.

Each agent's prompt must include:
- The bug description
- The recon report
- Its assigned theory (be specific)
- Instructions to: read relevant code, **run builds/tests/commands as needed** to confirm or deny the theory, add temporary diagnostic logging if it helps, write a brief verdict (LIKELY / UNLIKELY / CONFIRMED) with evidence (file paths, line numbers, command output, git commits)
- A **5-minute time budget** — focused investigation, not exhaustive

Wait for all 5 to complete. Synthesize all 5 reports into a **curated summary**: what was confirmed, what was ruled out, key evidence, files involved. Display a summary table:

```
| Agent | Theory | Verdict | Key Evidence |
```

## Phase 2: Deep dive (top 3, medium effort)

Select the **top 3** theories (CONFIRMED > LIKELY > UNLIKELY, break ties by evidence quality).

Spawn **3 agents in parallel**, each in an isolated worktree. Use `model: "opus"` — these agents need strong reasoning to identify root causes and write correct fixes. Each agent gets:
- The bug description
- The **orchestrator's curated summary** of all Phase 1 findings (not just their own theory)
- Its confirmed/likely theory as primary focus
- Instructions to:
  - Implement a fix
  - Write a regression test **if feasible** — "If you can write a test that fails without your fix and passes with it, do so. If the bug isn't practically testable (visual, timing, config), skip this and explain why."
  - Run whatever validation is available and appropriate for this project — build, compile, test suite, linter, type checker, or simply verify the code is syntactically correct
  - Report: files changed, approach summary, whether a test was written, what validation was run and results
- A **10-minute time budget** — implement the fix, don't over-engineer

Wait for all 3 to complete. For each, note:
- What validation did it run, and did it pass?
- What files were changed?
- Was a regression test written?
- How invasive is the fix?
- Does the approach make sense?

## Phase 3: Select winner

Evaluate the 3 fixes as orchestrator. Pick the **winner** based on:
1. Correctness — does it actually address the root cause?
2. Minimality — fewest lines changed, least risk of side effects
3. Completeness — handles edge cases the others miss
4. Validation confidence — an agent that ran a full test suite gets more trust than one that only checked syntax

If multiple fixes address different aspects of the same bug (compounding issues), **combine them** into a single fix. Otherwise, take the single best fix.

If no fix is satisfactory, explain why and ask the user for guidance.

## Phase 4: Implement winner

If the winning fix is already complete in its worktree, cherry-pick or manually apply it to a new branch off main. If combining fixes, apply them to a new branch.

Create a feature branch: `fix/<short-kebab-description>`

Run available validation on the feature branch (build, tests, linter — whatever the project supports).

## Phase 5: Validate (3 testers, 2/3 must pass)

Spawn **3 testing agents in parallel**, each in an isolated worktree based on the feature branch. Use `model: "sonnet"`.

Each tester gets:
- The bug description
- The orchestrator's curated summary of findings
- The fix that was applied (summary of changes)
- Whether a regression test was written in Phase 2
- A different validation angle:
  - **Tester 1: Functional** — Does the fix resolve the reported bug? Trace the code path. Run available validation. If a regression test was written, verify it fails on main and passes on the fix branch.
  - **Tester 2: Regression** — Could this fix break anything else? Check callers, dependents, edge cases. Run available validation.
  - **Tester 3: Code quality** — Is the fix clean? Any issues with error handling, memory, thread safety, naming? Does it follow project conventions?

Each tester must return a **PASS** or **FAIL** verdict with justification.

Wait for all 3.

## Phase 6: Retry or ship

### 2/3 or 3/3 testers pass → Ship it

1. Commit the fix on the feature branch with a clear commit message
2. Push the branch
3. Create a PR to main using `gh pr create` with:
   - Title: concise bug fix description
   - Body: Summary of the bug, root cause found, fix applied, and validation results

Report the PR URL to the user.

### 1/3 testers pass → Competitive retry

Spawn **2 new Opus fix agents in parallel**, each in an isolated worktree. Each gets:
- The bug description
- The orchestrator's curated summary
- The previous fix and what it got right
- The failure feedback: which testers failed, why, their justifications
- Instructions to revise the fix to address the failures

Wait for both to complete. Pick the better revised fix (same criteria as Phase 3). Apply to the feature branch.

Re-run Phase 5 validation. If **2/3 or 3/3 pass**, ship it. If it fails again, stop and report all findings to the user.

### 0/3 testers pass → Stop

The approach is fundamentally wrong. Report all findings, failure reasons, and evidence gathered across all phases. Ask the user for guidance.
