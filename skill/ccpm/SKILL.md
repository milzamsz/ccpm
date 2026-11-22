---
name: ccpm
description: "CCPM - spec-driven project management: PRD → Epic → GitHub Issues → parallel agents → shipped code. Use this skill for anything in the software delivery lifecycle: writing a PRD ('write a PRD for X', 'let's plan X', 'scope this out'), parsing a PRD into an epic, decomposing an epic into tasks, syncing to GitHub ('sync the X epic', 'push tasks to github'), starting work on an issue ('start working on issue N', 'let's work on issue N'), analyzing parallel work streams, running standups ('standup', 'run the standup'), checking status ('what's next', 'what's blocked', 'what are we working on'), closing issues, merging an epic, choosing who builds vs. reviews ('/ccpm roles'), or reviewing a builder's task PR. Use ccpm any time the user is talking about shipping a feature, managing work, or tracking progress — even if they don't say 'ccpm' or 'PRD'. Do NOT use for: debugging code, writing tests, reviewing PRs with no CCPM task, or raw GitHub issue/PR operations with no delivery context."
---

# CCPM - Claude Code Project Manager

A spec-driven development workflow: PRD → Epic → GitHub Issues → Parallel Agents → Shipped Code.

## Core Philosophy

Requirements live in files, not heads. Every feature starts as a PRD, becomes a technical epic, decomposes into GitHub issues, and gets executed by parallel agents with full traceability.

## File Conventions

Before doing anything, read `references/conventions.md` for path standards, frontmatter schemas, and GitHub operation rules. These apply to all phases.

## Role Mode — Who Builds, Who Reviews

Each project picks a role mode in `.claude/ccpm/roles.json` (missing file = `agent-builds`). Read it before starting an issue, opening or reviewing a PR, or merging.

- **`agent-builds`** (default): this agent implements on task branches, opens PRs, requests review from the configured reviewer (e.g. `@codex review`), fixes findings on the same PR, and merges only under the merge gate in `references/sync.md`.
- **`reviewer-agent`**: an external builder (e.g. Codex) implements; this agent writes build requests, verifies every head SHA locally, reviews it, asks the builder for fixes, and merges only under the gate in `references/review.md`.

The repository owner gives the final merge permission in both modes. Show or switch the mode with `/ccpm roles` or `/ccpm roles agent-builds|reviewer-agent` (**Read**: `references/review.md`, "Showing and Switching the Mode"). A PR keeps the mode it started with (`CCPM-Role-Mode:` line in its body).

## The Five Phases

### 1. Plan — Capture requirements
**When**: User wants to define a new feature, product requirement, or scope of work.
**Read**: `references/plan.md`
**Covers**: Writing PRDs through guided brainstorming, converting PRDs to technical epics.

### 2. Structure — Break it down
**When**: An epic exists and needs to be decomposed into concrete tasks.
**Read**: `references/structure.md`
**Covers**: Epic decomposition into numbered task files with dependencies and parallelization.

### 3. Sync — Push to GitHub
**When**: Local epic/tasks need to become GitHub issues, progress needs to be posted as comments, or a bug is found and needs a linked issue created.
**Read**: `references/sync.md`
**Covers**: Epic sync (epic + tasks → GitHub issues), issue sync (progress comments), Draft PR → review → merge via GitHub PR (never a direct push to main), closing issues/epics, bug reporting against completed issues.
**Mode**: in `reviewer-agent`, also read `references/review.md` for reviewing the builder's PRs and the merge gate; post-merge steps stay in `sync.md`.

### 4. Execute — Start building
**When**: User wants to start working on one or more GitHub issues with parallel agents.
**Read**: `references/execute.md`
**Covers**: Issue analysis (parallel work stream identification), launching parallel agents, coordinating worktrees.
**Mode**: in `reviewer-agent`, read `references/review.md` instead: starting an issue sends a build request to the external builder rather than launching agents.

### 5. Track — Know where things stand
**When**: User asks for status, standup report, what's blocked, what's next, or needs to validate state.
**Read**: `references/track.md`
**Covers**: Status, standup, search, in-progress, next priority, blocked items, validation.

---

## Script-First Rule

For deterministic operations — anything that reads and reports without needing reasoning — always run the bash script directly rather than doing the work manually:

| What the user wants | Script to run |
|---|---|
| Project status | `bash references/scripts/status.sh` |
| Standup report | `bash references/scripts/standup.sh` |
| List all epics | `bash references/scripts/epic-list.sh` |
| Show epic details | `bash references/scripts/epic-show.sh <name>` |
| Epic status | `bash references/scripts/epic-status.sh <name>` |
| List PRDs | `bash references/scripts/prd-list.sh` |
| PRD status | `bash references/scripts/prd-status.sh` |
| Search issues/tasks | `bash references/scripts/search.sh <query>` |
| What's in progress | `bash references/scripts/in-progress.sh` |
| What's next | `bash references/scripts/next.sh` |
| What's blocked | `bash references/scripts/blocked.sh` |
| Validate project state | `bash references/scripts/validate.sh` |
| Neon branch for an epic (optional) | `bash references/scripts/neon-branch.sh create <epic>` |
| Remove that Neon branch on merge | `bash references/scripts/neon-branch.sh cleanup <epic>` |

Use the LLM for work that requires reasoning: writing PRDs, analyzing parallelism, launching agents, synthesizing updates.

---

## Quick Reference

```
Plan a feature:     "I want to build X" or "create a PRD for X"
Parse to epic:      "turn the X PRD into an epic"
Decompose:          "break down the X epic into tasks"
Sync to GitHub:     "push the X epic to GitHub"
Start an issue:     "start working on issue 42"
Check status:       "what's our status" / "standup"
What's next:        "what should I work on next"
Merge epic:         "merge the X epic" (GitHub PR; needs review of latest SHA, green checks, owner approval)
Role mode:          "/ccpm roles" (show) / "/ccpm roles reviewer-agent" (external builder, this agent reviews)
Review a task PR:   "review the PR for issue 42" (reviewer-agent mode)
Report a bug:       "found a bug in issue 42" / "testing issue 42 revealed X"
```
