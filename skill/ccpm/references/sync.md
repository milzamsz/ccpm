# Sync — Push to GitHub & Track Progress

This phase covers pushing local epics/tasks to GitHub as issues, syncing progress as comments, and closing issues when work is done.

---

## Repository Safety Check

**Always run this before any GitHub write operation:**

```bash
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$remote_url" == *"automazeio/ccpm"* ]]; then
  echo "❌ Cannot sync to the CCPM template repository."
  echo "Update remote: git remote set-url origin https://github.com/YOUR/REPO.git"
  exit 1
fi
REPO=$(echo "$remote_url" | sed 's|.*github.com[:/]||' | sed 's|\.git$||')
```

---

## Epic Sync — Push Epic + Tasks to GitHub

**Trigger**: User wants to push a local epic and its tasks to GitHub as issues.

### Preflight
- Verify `.claude/epics/<name>/epic.md` exists.
- Verify numbered task files exist — if none: "❌ No tasks to sync. Decompose the epic first."

### Process

Strip frontmatter with this `awk` command. Do not use `sed '1,/^---$/d; 1,/^---$/d'`: on GNU sed it deletes the whole file and the issue is created with an empty body.

```bash
strip_frontmatter() { awk 'NR==1 && $0!="---" {fm=2} fm<2 && $0=="---" {fm++; next} fm>=2' "$1"; }
```

`gh issue create` has no `--json` flag. It prints the new issue URL, so take the number from the end of it.

**Step 1 — Create epic issue:**

```bash
body_file=$(mktemp)
strip_frontmatter .claude/epics/<name>/epic.md > "$body_file"
epic_url=$(gh issue create \
  --repo "$REPO" \
  --title "Epic: <name>" \
  --body-file "$body_file" \
  --label "epic,epic:<name>,feature")
epic_number=$(echo "$epic_url" | grep -oE '[0-9]+$')
rm -f "$body_file"
```

Write `github: $epic_url` into epic.md right away (see Step 4), so an interrupted sync does not create the epic twice.

**Step 2 — Create task issues:**

Skip any task file that already has a `github:` URL — it was synced before. This makes a re-run after an interruption safe.

Check if the `gh-sub-issue` extension is available:
```bash
use_subissues=false
if gh extension list | grep -q "yahsan2/gh-sub-issue"; then
  use_subissues=true
fi
```

For <5 tasks: create sequentially.
For ≥5 tasks: use parallel Task agents (3-4 tasks per batch). Each agent must use its own temp file (`mktemp`), never a shared path like `/tmp/task-body.md`.

Per task:
```bash
body_file=$(mktemp)
strip_frontmatter <task_file> > "$body_file"
task_url=$(gh issue create \
  --repo "$REPO" \
  --title "<task_name>" \
  --body-file "$body_file" \
  --label "task,epic:<name>")
task_number=$(echo "$task_url" | grep -oE '[0-9]+$')
rm -f "$body_file"

# Link the task to the epic as a sub-issue
if [ "$use_subissues" = true ]; then
  gh sub-issue add "$epic_number" "$task_number" --repo "$REPO"
fi
```

Right after each issue is created, write `github: $task_url` into that task file (see Step 4). Do not wait for the whole batch.

**Step 3 — Rename task files and update references:**

After all issues are created, rename `001.md` → `<issue_number>.md` and update all `depends_on`/`conflicts_with` arrays to use real issue numbers (not sequential numbers).

```bash
# Build old→new mapping, then for each task file:
sed -i.bak "s/\b001\b/<new_num_1>/g" <file>  # repeat for each mapping
mv 001.md <new_num>.md
```

**Step 4 — Update frontmatter:**
```bash
current_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
# Update github: and updated: fields in epic.md and each task file
github_url="https://github.com/$REPO/issues/<number>"
sed -i.bak "/^github:/c\\github: $github_url" <file>
sed -i.bak "/^updated:/c\\updated: $current_date" <file>
rm <file>.bak
```

**Step 5 — Create worktree for the epic:**
```bash
git checkout main && git pull origin main
git worktree add ../epic-<name> -b epic/<name>

# Optional Neon branch. Exits 0 when the Neon CLI or NEON_PROJECT_ID is missing.
# Stop if this exits non-zero.
bash <skill_path>/references/scripts/neon-branch.sh create <name>
```

`<skill_path>` is the directory that contains this skill's `SKILL.md`.

**Step 6 — Create github-mapping.md:**

This is a generated index, so it has no frontmatter. `validate.sh` does not check it.
```markdown
# GitHub Issue Mapping
Epic: #<N> - https://github.com/<repo>/issues/<N>
Tasks:
- #<N>: <title> - https://github.com/<repo>/issues/<N>
Synced: <datetime>
```

**Output:**
```
✅ Synced epic <name> to GitHub
  Epic: #<N>
  Tasks: N sub-issues
  Worktree: ../epic-<name>
  Next: "start working on issue <N>" or "start the <name> epic"
```

---

## Issue Sync — Post Progress to GitHub

**Trigger**: User wants to sync local development progress to a GitHub issue as a comment.

### Preflight
- Verify issue exists: `gh issue view <N> --json state`
- Check `.claude/epics/*/updates/<N>/` exists with a `progress.md` file.
- Check `last_sync` in progress.md — if synced <5 minutes ago, confirm before proceeding.

### Process

Gather updates from `.claude/epics/<epic>/updates/<N>/` (progress.md, notes.md, commits.md).

Format and post a comment:
```bash
gh issue comment <N> --body-file /tmp/update-comment.md
```

Comment format:
```markdown
## 🔄 Progress Update - <date>

### ✅ Completed Work
### 🔄 In Progress
### 📝 Technical Notes
### 📊 Acceptance Criteria Status
### 🚀 Next Steps
### ⚠️ Blockers

---
*Progress: N% | Synced at <timestamp>*
```

After posting: update `last_sync` in progress.md frontmatter, update `updated` in the task file.

Add sync marker to local files to prevent duplicate comments:
```markdown
<!-- SYNCED: <datetime> -->
```

---

## Closing an Issue

**Trigger**: User marks a task complete. If the task's code goes through a PR, close the issue only after GitHub confirms that PR is merged (see "Review & Merge via Pull Request").

### Process

1. Find the local task file (`.claude/epics/*/<N>.md`).
2. Update frontmatter: `status: closed`, `updated: <now>`.
3. Post completion comment:
```bash
echo "✅ Task completed — all acceptance criteria met." | gh issue comment <N> --body-file -
gh issue close <N>
```
4. Check off the task in the epic issue body:
```bash
gh issue view <epic_N> --json body -q .body > /tmp/epic-body.md
sed -i "s/- \[ \] #<N>/- [x] #<N>/" /tmp/epic-body.md
gh issue edit <epic_N> --body-file /tmp/epic-body.md
```
5. Recalculate and update epic progress: `progress = closed_tasks / total_tasks * 100`
6. If this issue had its own Neon branch, delete just that child. No-op when Neon is not configured:
```bash
bash <skill_path>/references/scripts/neon-branch.sh cleanup <epic_name> --issue <N>
```

---

## Review & Merge via Pull Request

**Trigger**: User wants to deliver a task or epic, address review findings, or merge a completed epic ("merge the X epic").

Code reaches `main` only through a GitHub Pull Request. Never `git push origin main`, never merge locally into `main`, never use `gh pr merge --admin`, `--force`, or any other branch-protection bypass.

Roles: the **builder** (this agent) implements; an **independent reviewer** (e.g. Codex) reviews; the **repository owner** gives the final merge decision.

### 1. Read and plan
- Read the GitHub Issue (`gh issue view <N> --json title,body,labels`) and the project's `AGENTS.md` / `CLAUDE.md`.
- Use the Issue's acceptance criteria as the reference for done.
- Record the technical plan and any scope changes that need agreement (as an Issue comment or in the PR body) before building them.

### 2. Implement and open a Draft PR
Work on the task or epic branch (`epic/<name>` in `../epic-<name>`, or a task branch cut from it). Then:

```bash
cd ../epic-<name>
# run the project's validation commands (tests, lint, typecheck, build)
git push -u origin epic/<name>
gh pr create --draft --base main --head epic/<name> \
  --title "<Epic or task title>" --body-file /tmp/pr-body.md
```

The PR body must:
- link the Issue(s) with `Closes #<N>` (one line per task issue it completes; add the epic issue only when the PR completes the whole epic),
- list the validation commands and the test results that were actually run (never results that were not run),
- list scope changes that still need agreement.

### 3. Address review findings in the same PR
- Fix the reviewer's findings on the same branch and PR; do not open a replacement PR.
- After each revision, post a PR comment with: summary of changes, latest test results, and the head commit SHA (`git rev-parse HEAD`), and ask for re-review of that SHA.

### 4. Merge gate (all three required)
1. The independent reviewer has finished reviewing the **latest** head SHA. If any commit was pushed after that review, request re-review first.
2. Required GitHub checks pass: `gh pr checks <PR> --required`.
3. The repository owner has given **explicit** permission to merge this PR.

Then mark ready and merge through GitHub, pinned to the reviewed SHA:

```bash
reviewed_sha=<SHA the reviewer approved>
gh pr ready <PR>
gh pr merge <PR> --merge --match-head-commit "$reviewed_sha"
```

If the merge is refused (protection rules, failing checks, SHA mismatch), stop and report; do not work around it.

### 5. After GitHub confirms the merge

```bash
gh pr view <PR> --json state,mergedAt,mergeCommit   # state must be MERGED
git checkout main && git pull origin main
```

Only after `state` is `MERGED`:
- Close or update the task issues it completed (see "Closing an Issue"), and update local task/epic frontmatter and epic progress.
- Close the epic only when every required task is closed:
```bash
epic_issue=$(grep 'github:' .claude/epics/<name>/epic.md | grep -oE '[0-9]+$')
gh issue close $epic_issue -c "Epic completed — merged via PR #<PR>"
```
- When the epic is closed: set `status: completed` in epic.md, then clean up:
```bash
# Delete Neon branches for this epic (children first). No-op when Neon is not configured.
# Stop if this exits non-zero.
bash <skill_path>/references/scripts/neon-branch.sh cleanup <name>

git worktree remove ../epic-<name>
git branch -d epic/<name>
git push origin --delete epic/<name>

mkdir -p .claude/epics/archived/
mv .claude/epics/<name> .claude/epics/archived/
```

---

## Reporting a Bug Against a Completed Issue

**Trigger**: User finds a bug while testing a completed or in-progress issue — e.g. "found a bug in issue 42", "email validation is broken, came up while testing issue 42".

The workflow should stay automated: create a linked bug task without losing context from the original issue.

### Process

**Step 1 — Read the original issue for context:**
```bash
gh issue view <original_N> --json title,body,labels
```
Also read the local task file if it exists: `.claude/epics/*/<original_N>.md`

**Step 2 — Create a local bug task file:**

```markdown
---
name: Bug: <short description>
status: open
created: <run: date -u +"%Y-%m-%dT%H:%M:%SZ">
updated: <same>
github: (will be set on sync)
depends_on: []
parallel: false
conflicts_with: []
bug_for: <original_N>
---

# Bug: <short description>

## Context
Found while working on / testing issue #<original_N>: <original title>

## Description
<what's broken>

## Steps to Reproduce
<steps>

## Expected vs Actual
- Expected: 
- Actual: 

## Acceptance Criteria
- [ ] Bug is fixed
- [ ] Original issue #<original_N> behaviour is unaffected

## Effort Estimate
- Size: XS/S
```

Save to `.claude/epics/<same_epic_as_original>/bug-<original_N>-<slug>.md`

**Step 3 — Create a linked GitHub issue:**
```bash
bug_url=$(gh issue create \
  --repo "$REPO" \
  --title "Bug: <short description>" \
  --body-file "$body_file" \
  --label "bug,epic:<epic_name>")
bug_number=$(echo "$bug_url" | grep -oE '[0-9]+$')
```

`$body_file` is the bug task body without frontmatter (use `strip_frontmatter` and `mktemp` as in Epic Sync).

The issue body should open with `Fixes / follow-up to #<original_N>` so GitHub auto-links them.

**Step 4 — Update the local file** with the GitHub issue number and rename to `<new_N>.md`.

**Output:**
```
✅ Bug issue created: #<new_N> — "Bug: <short description>"
  Linked to: #<original_N>
  Epic: <epic_name>

Start fixing it: "start working on issue <new_N>"
```
