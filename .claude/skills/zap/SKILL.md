---
name: zap
description: Fix a bug using competitive multi-agent investigation. 3 recon agents survey the codebase, 3 agents investigate theories (with early termination on confirmed root cause), 1-2 implement fixes, 3 testers validate, retries on partial failure, then commits and creates a PR. Use when a bug is persistent, non-obvious, or has resisted prior fix attempts.
argument-hint: "<bug description> [--single-recon] [--no-security] [--no-cleanup]"
---

# Multi-Agent Bug Fix

Fix the following bug using competitive multi-agent investigation:

**Bug:** $ARGUMENTS

## Argument Parsing

Parse `$ARGUMENTS` to extract:
1. **Bug description**: Everything except flags
2. **Flags**:
   - `--single-recon`: Use single-agent recon instead of parallel (see Phase 0)
   - `--no-security`: Skip security review in quality gates (Phase 6)
   - `--no-cleanup`: Skip code simplification & beautification in quality gates (Phase 6)

## Phase 0: Reconnaissance

### Code index detection

Before spawning recon agents, check `.mcp.json` (project root, then `~/.claude/.mcp.json`) for a code search MCP. Look for server entries matching any of these patterns:

| MCP server | Key / command contains | Key search tools |
|------------|----------------------|------------------|
| code-index-mcp | `code-index`, `code-index-mcp` | `search_code_advanced`, `get_file_summary`, `find_files` |
| claude-context | `claude-context` | `search_code`, `search_symbols` |
| Code-Index-MCP (ViperJuice) | `code-index-mcp`, `viperjuice` | `search`, `lookup_symbol` |

If any code search MCP is detected, include this instruction in **every recon and investigation agent prompt**:

> "A code search MCP is available. Prefer its search tools over grep/glob for finding relevant code — they return more targeted results with less noise. Fall back to grep/glob for simple exact-string matches or if the MCP tools don't return useful results."

If no code search MCP is detected, proceed normally (agents use grep/glob). After Phase 0 completes, include a one-line note in the final report: **"Tip: Install a code search MCP (e.g. `code-index-mcp`) for faster, more accurate recon."**

---

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
  - Return a **structured report**: relevant files (paths + line ranges + one-line descriptions), code path analysis, suspicious areas
  - **Keep the report under 300 lines. Return file paths and line ranges, NOT full file contents. Downstream agents can read files themselves.**
- A **3-minute time budget** — breadth over depth

**Agent 2 — Git archaeologist:**
- The bug description
- Instructions to:
  - Run `git log` on files and directories likely related to the bug
  - Run `git blame` on the most suspect sections
  - Identify recent changes that could have introduced the bug
  - Check commit messages and PR descriptions for related context
  - Return a **structured report**: recent changes (commit hashes, authors, dates, one-line summaries), suspect commits, relevant history
  - **Keep the report under 300 lines. Summarize diffs — do not paste full diffs. Include commit hashes so downstream agents can inspect if needed.**
- A **3-minute time budget** — breadth over depth

**Agent 3 — Test runner:**
- The bug description
- Instructions to:
  - Identify and run existing tests related to the bug area
  - Note which tests pass and which fail, with output
  - Search for related TODOs, FIXMEs, HACKs, or known-issue comments
  - Check for disabled/skipped tests that might be relevant
  - Return a **structured report**: test results (pass/fail with brief output), related comments and TODOs, test coverage gaps
  - **Keep the report under 300 lines. For failing tests, include the assertion/error message only, not full stack traces. For passing tests, just list them.**
- A **3-minute time budget** — breadth over depth

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
  - Return a **structured report**: relevant files (paths + line ranges + one-line descriptions), recent changes (commit hashes, dates, one-line summaries), test results, initial observations
  - **Keep the report under 400 lines. Return file paths and line ranges, NOT full file contents. Summarize diffs. Downstream agents can read files themselves.**
- A **3-minute time budget** — breadth over depth

Wait for the agent to complete. Save its report — this becomes the **recon report**, a shared artifact that every agent in every subsequent phase receives as baseline context.

## Phase 1: Investigate (3 agents, fast)

Read the recon report. Generate **3 distinct theories** about the root cause, grounded in the recon findings. Theories should cover different layers of the stack (e.g., data flow, timing/async, state management, configuration). One theory must always be a **git history theory**: "The bug was introduced by a recent change — identify which commit and why." Avoid overlapping theories.

Spawn **3 agents in parallel**, each in an isolated worktree. Use `model: "sonnet"`.

Each agent's prompt must include:
- The bug description
- The recon report
- Its assigned theory (be specific)
- Instructions to: read relevant code, **run builds/tests/commands as needed** to confirm or deny the theory, add temporary diagnostic logging if it helps, write a brief verdict (LIKELY / UNLIKELY / CONFIRMED) with evidence (file paths, line numbers, command output, git commits)
- A **5-minute time budget** — focused investigation, not exhaustive

Wait for all 3 to complete. Synthesize all 3 reports into a **curated summary**: what was confirmed, what was ruled out, key evidence, files involved. Display a summary table:

```
| Agent | Theory | Verdict | Key Evidence |
```

### Early termination

If any investigation agent returns **CONFIRMED** with strong evidence (specific file, line, and reproduction), skip directly to Phase 2 with **only 1 agent** for the confirmed theory. Do not spawn agents for unconfirmed theories — the evidence is sufficient.

## Phase 2: Deep dive (top theories, medium effort)

**If early termination triggered (1 CONFIRMED theory):** Spawn **1 agent** in an isolated worktree. Use `model: "opus"`.

**Otherwise:** Select the **top 2** theories (CONFIRMED > LIKELY > UNLIKELY, break ties by evidence quality). Spawn **2 agents in parallel**, each in an isolated worktree. Use `model: "opus"` for the strongest theory and `model: "sonnet"` for the second.

**Context trimming:** Before writing P2 prompts, distill the recon report + P1 findings into a **focused brief** for each agent. Include only: the bug description, the relevant theory with its key evidence (files, lines, commits), and a list of files to read. Do NOT forward the full recon report or full P1 reports — agents can read files themselves.

Each agent gets:
- The bug description
- The **orchestrator's focused brief**: trimmed findings relevant to this agent's theory (not the full recon report or all P1 reports)
- Its confirmed/likely theory as primary focus
- Instructions to:
  - Implement a fix
  - Write a regression test **if feasible** — "If you can write a test that fails without your fix and passes with it, do so. If the bug isn't practically testable (visual, timing, config), skip this and explain why."
  - Run whatever validation is available and appropriate for this project — build, compile, test suite, linter, type checker, or simply verify the code is syntactically correct
  - Report: files changed, approach summary, whether a test was written, what validation was run and results
- A **10-minute time budget** — implement the fix, don't over-engineer

Wait for all fix agents to complete. For each, note:
- What validation did it run, and did it pass?
- What files were changed?
- Was a regression test written?
- How invasive is the fix?
- Does the approach make sense?

## Phase 3: Select winner

**If only 1 fix agent was spawned (early termination):** Skip selection — use that fix directly.

**If 2 fix agents were spawned:** Evaluate both fixes as orchestrator. Pick the **winner** based on:
1. Correctness — does it actually address the root cause?
2. Minimality — fewest lines changed, least risk of side effects
3. Completeness — handles edge cases the others miss
4. Validation confidence — an agent that ran a full test suite gets more trust than one that only checked syntax

If both fixes address different aspects of the same bug (compounding issues), **combine them** into a single fix. Otherwise, take the single best fix.

If no fix is satisfactory, explain why and ask the user for guidance.

## Phase 4: Implement winner

If the winning fix is already complete in its worktree, cherry-pick or manually apply it to a new branch off main. If combining fixes, apply them to a new branch.

Create a feature branch: `fix/<short-kebab-description>`

Run available validation on the feature branch (build, tests, linter — whatever the project supports).

## Phase 5: Validate (3 testers, 2/3 must pass)

Spawn **3 testing agents in parallel**, each in an isolated worktree based on the feature branch. Use `model: "haiku"`.

Each tester gets:
- The bug description (1-2 sentences)
- A **brief fix summary**: what was changed, which files, and the approach (not the full investigation history)
- Whether a regression test was written in Phase 2
- A different validation angle:
  - **Tester 1: Functional** — Does the fix resolve the reported bug? Trace the code path. Run available validation. If a regression test was written, verify it fails on main and passes on the fix branch.
  - **Tester 2: Regression** — Could this fix break anything else? Check callers, dependents, edge cases. Run available validation.
  - **Tester 3: Code quality** — Is the fix clean? Any issues with error handling, memory, thread safety, naming? Does it follow project conventions?

Each tester must return a **PASS** or **FAIL** verdict with justification.

Wait for all 3.

## Phase 6: Retry or ship

### 2/3 or 3/3 testers pass → Quality gates & ship

Run quality gates on the feature branch before committing, then ship.

#### Quality gates

Run these **in parallel** on the feature branch. Each agent works in an isolated worktree based on the feature branch. After both complete, apply any changes from both agents to the feature branch (cherry-pick or manually merge — resolve conflicts if any).

**Security Review & Fix** (skip if `--no-security`):

Spawn 1 agent in an isolated worktree. Use `model: "haiku"`.

The agent gets:
- The bug description
- The output of `git diff main` showing all changes on the feature branch
- Instructions to:
  - Review all changed code for security vulnerabilities
  - Check OWASP Top 10: injection (SQL, command, code), XSS, insecure deserialization, broken access control, sensitive data exposure, security misconfiguration
  - Check for: path traversal, unsafe regex, hardcoded secrets or credentials, missing input validation, unsafe file operations
  - If any issues found, **fix them directly** in the code
  - Run the project's test suite or linter on changed files if available, to verify fixes don't break anything
  - Report: issues found and fixed, or "no security issues found"
- A **5-minute time budget**

**Code Simplification & Beautification** (skip if `--no-cleanup`):

Spawn 1 agent in an isolated worktree. Use `model: "haiku"`.

The agent gets:
- The output of `git diff main` showing all changes on the feature branch
- Instructions to:
  - Detect and run the project's linter/formatter if one is configured (e.g., prettier, black, rustfmt, gofmt, eslint --fix, phpcs/phpcbf, clang-format)
  - Remove dead code and unused variables/imports introduced by the fix
  - Simplify unnecessarily complex conditionals or nested logic
  - Remove comments that state the obvious
  - Ensure consistent naming and style with the surrounding code
  - Do **not** refactor code outside the scope of the bug fix
  - Report: simplifications applied, formatting fixes, linter results
- A **5-minute time budget**

#### Commit and create PR

1. Commit the fix on the feature branch with a clear commit message
2. Push the branch to the remote
3. Create a PR using `gh pr create` with a title and body that summarise the bug, root cause, and fix approach

Then proceed to **Phase 7**.

### 1/3 testers pass → Competitive retry

Spawn **2 new Opus fix agents in parallel**, each in an isolated worktree. Each gets:
- The bug description
- The orchestrator's curated summary
- The previous fix and what it got right
- The failure feedback: which testers failed, why, their justifications
- Instructions to revise the fix to address the failures

Wait for both to complete. Pick the better revised fix (same criteria as Phase 3). Apply to the feature branch.

Re-run Phase 5 validation. If **2/3 or 3/3 pass**, follow the "Quality gates & ship" steps above. If it fails again, stop and report all findings to the user.

### 0/3 testers pass → Stop

The approach is fundamentally wrong. Report all findings, failure reasons, and evidence gathered across all phases. Ask the user for guidance.

## Phase 7: PR Feedback Loop

After the PR is created and pushed, wait for automated review bots (linters, CI checks, code review bots) to post their feedback before reporting to the user.

1. **Check for CI/review bots**: Run `ls .github/workflows/ 2>/dev/null` and `gh api repos/{owner}/{repo}/hooks --jq length 2>/dev/null` to detect if the repo has CI workflows or webhooks configured. If **neither exists**, skip the wait and go directly to step 2. Otherwise, **wait 3 minutes** for automated review bots to post comments (run `sleep 180`).

2. **Read all PR comments** using both of these commands (replace `{PR_NUMBER}` with the actual PR number, and `{owner}/{repo}` with the repo details from `gh repo view --json nameWithOwner`):
   ```
   gh api repos/{owner}/{repo}/pulls/{PR_NUMBER}/reviews
   gh api repos/{owner}/{repo}/pulls/{PR_NUMBER}/comments
   gh pr view {PR_NUMBER} --comments --json comments
   ```

3. **For each comment or suggestion**, evaluate whether it is a valid, actionable suggestion:
   - Read the referenced file and line using the file path and position info in the comment
   - Assess whether the suggestion would genuinely improve the code (correctness, quality, security, style consistency)
   - If the suggestion is incorrect, inapplicable, or would make the code worse, mark it as **rejected** and record a brief reason
   - If the suggestion is valid and beneficial, **implement the fix** directly on the feature branch

4. **Push any fixes** to the same branch with a follow-up commit:
   ```
   git add <changed files>
   git commit -m "address PR review feedback"
   git push
   ```

5. **Compile a feedback summary** listing every comment reviewed:
   - **Accepted**: what was changed and why it was a good suggestion
   - **Rejected**: what was suggested and why it was declined

If no comments are found after waiting, proceed directly to the final report.

## Final Report

Report to the user:

- **Branch**: the feature branch name
- **PR**: the PR URL
- **Fix summary**: root cause identified, approach taken, files changed
- **Validation**: which testers passed/failed and what they checked
- **Quality gates**: security issues found/fixed (or skipped), code cleanup applied (or skipped)
- **PR feedback**: accepted and rejected suggestions (if any)
- **Next step**: remind the user they can use `/zap-cleanup` to merge and clean up
