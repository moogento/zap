![zap banner](assets/banner.png)

# zap

A Claude Code plugin that fixes bugs using competitive multi-agent investigation. When a bug resists your first few attempts, zap throws a squad of agents at it — recon, parallel theories, competing fixes, independent validation — and ships a PR.

## How It Works

![zap investigation flow](assets/flow.png)

**7 phases, fully autonomous:**

| Phase | What happens | Agents | Model |
|-------|-------------|--------|-------|
| **P0 Recon** | Survey codebase, git history, run tests | 3 (or 1) | Sonnet |
| **P1 Investigate** | 5 different theories tested in parallel | 5 | Sonnet |
| **P2 Deep dive** | Top 3 theories get full fix implementations | 3 | Opus |
| **P3 Select** | Orchestrator picks the best fix | — | — |
| **P4 Apply** | Winner applied to a feature branch | — | — |
| **P5 Validate** | 3 testers check functional, regression, quality | 3 | Sonnet |
| **P6 Ship/Retry** | PR on pass, competitive retry on partial fail | 0–2 | Opus |

Every agent runs in an isolated git worktree. No agent can break your working tree.

## Usage

```
/zap <describe the bug>
```

Options:

| Flag | Effect |
|------|--------|
| `--single-recon` | Use 1 recon agent instead of 3 parallel (cheaper, slower) |

Examples:

```
/zap thumbnails don't update after importing a 3D model

/zap race condition in WebSocket reconnect — messages dropped on fast network switches

/zap build fails on CI but passes locally, something about asset paths

/zap --single-recon off-by-one in pagination
```

## What Makes It Different

**Parallel recon before theories.** Three specialized agents — code reader, git archaeologist, test runner — survey the codebase simultaneously before anyone starts guessing. Theories are grounded in evidence, not vibes.

**Agents compete, not collaborate.** 5 independent theories, 3 independent fixes. No groupthink. The best fix wins on merit.

**Cross-pollination without contamination.** Each phase receives a curated summary of all prior findings. Agents know what was ruled out, not just what was confirmed.

**Validation is adversarial.** 3 independent testers try to break the fix from different angles. 2 of 3 must pass. No rubber-stamping.

**Retry is competitive.** If validation partially fails, 2 new agents compete to revise the fix using failure feedback. One more shot before giving up.

## Install

```bash
claude plugins add github:moogento/zap
```

Then restart Claude Code. `/zap` will be available in all projects.

## When to Use It

- Bug has resisted your first couple of fix attempts
- Root cause is unclear or could be multiple things
- Bug spans multiple files or layers of the stack
- You want a fix validated before you commit to it
- You're stuck and want fresh (parallel) perspectives

## When Not to Use It

- Typo fixes, obvious one-liners
- You already know the root cause and just need to write the fix
- The bug is in a file you haven't read yet (read it first, you might not need zap)

## Cost

A full run spawns ~15 agents across all phases. Expect roughly:
- **Typical run:** ~14 agents (3 recon + 5 investigate + 3 fix + 3 test)
- **With retry:** ~19 agents (add 2 fix + 2–3 retest)
- **Model mix:** Sonnet for investigation/testing, Opus for fix implementation

Use it for bugs that justify the investment.

## Requirements

- Claude Code CLI
- Git repository (worktrees require it)
- That's it — no dependencies, no config, no build step

## License

MIT
