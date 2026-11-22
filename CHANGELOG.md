# CCPM Changelog

## [Unreleased]

### ✨ Added
- **Role modes: who builds, who reviews**
  - New `.claude/ccpm/roles.json` chooses between `agent-builds` (default, unchanged behaviour: the agent builds, an independent reviewer such as Codex reviews) and `reviewer-agent` (an external builder such as Codex builds, the agent reviews)
  - New `references/review.md`: config fields, `/ccpm roles` show/switch, build request template, local verification of every head SHA, review checklist, severity-badged inline findings (P1/P2 blocking by default), re-review loop, merge gate and stop rules for `reviewer-agent`
  - PRs record their mode with a `CCPM-Role-Mode:` line and keep it when the project switches; task `progress.md` records `role_mode`
- **Builder failover** (`review.md` §6, `failover` block in `roles.json`, on by default when both roles are filled)
  - When the builder hits a usage/rate limit (agent: any sub-agent or request fails with e.g. HTTP 429, fail over before the main session is blocked) or the external builder reports it cannot run or does not respond within `no_response` (default 45m), the other agent takes over the active task on the same branch and PR
  - Handoff comment (done / not done, last head SHA, reason, reset time); the incoming builder re-runs every `verify` command before continuing; `failover: <from>→<to>, reason, at, reset` recorded in progress.md (authoritative) and the PR body
  - A builder never reviews its own commits: the original builder reviews the substitute's work; if it is limited the merge waits (`review_if_reviewer_limited: "wait"`), never unreviewed; the owner may review or waive explicitly
  - `return: "next-task"`: no mid-task switching back; the configured roles apply from the next task
  - Owner notified on every failover (PR comment @-mentioning the repository owner, plus a harness notification where available); if both agents are limited, work stops and resumes at the earliest reset

### 🔄 Changed
- `SKILL.md` routes Execute and Sync to `review.md` in `reviewer-agent` mode
- `sync.md` "Roles" now points at the role modes; the agent-builds gate uses `reviewer.request` to ask for re-review and treats `blocking` severities as merge blockers
- `sync.md` merge gate: during a builder failover the review must come from the agent that did not build the latest commits; no unreviewed merge
- `review.md` stop rules: with failover on, an unresponsive builder triggers failover instead of only asking the owner

## [2025-01-24] - Major Cleanup & Issue Resolution Release

### 🎯 Overview
Resolved 10 of 12 open GitHub issues, modernized command syntax, improved documentation, and enhanced system accuracy. This release focuses on stability, usability, and addressing community feedback.

### ✨ Added
- **Local Mode Support** ([#201](https://github.com/automazeio/ccpm/issues/201))
  - Created `LOCAL_MODE.md` with comprehensive offline workflow guide
  - All core commands (prd-new, prd-parse, epic-decompose) work without GitHub
  - Clear distinction between local-only vs GitHub-dependent commands

- **Automatic GitHub Label Creation** ([#544](https://github.com/automazeio/ccpm/issues/544))
  - Enhanced `init.sh` to automatically create `epic` and `task` labels
  - Proper colors: `epic` (green #0E8A16), `task` (blue #1D76DB)  
  - Eliminates manual label setup during project initialization

- **Context Creation Accuracy Safeguards** ([#48](https://github.com/automazeio/ccpm/issues/48))
  - Added mandatory self-verification checkpoints in context commands
  - Implemented evidence-based analysis requirements
  - Added uncertainty flagging with `⚠️ Assumption - requires verification`
  - Enhanced both `/context:create` and `/context:update` with accuracy validation

### 🔄 Changed
- **Modernized Command Syntax** ([#531](https://github.com/automazeio/ccpm/issues/531))
  - Updated 14 PM command files to use concise `!bash` execution pattern
  - Simplified `allowed-tools` frontmatter declarations
  - Reduced token usage and improved Claude Code compatibility

- **Comprehensive README Overhaul** ([#323](https://github.com/automazeio/ccpm/issues/323))
  - Clarified PRD vs Epic terminology and definitions
  - Streamlined workflow explanations and removed redundant sections
  - Fixed installation instructions and troubleshooting guidance
  - Improved overall structure and navigation

### 📋 Research & Community Engagement
- **Multi-Tracker Support Analysis** ([#200](https://github.com/automazeio/ccpm/issues/200))
  - Researched CLI availability for Linear, Trello, Azure DevOps, Jira
  - Identified Linear as best first alternative to GitHub Issues
  - Provided detailed implementation roadmap for future development

- **GitLab Support Research** ([#588](https://github.com/automazeio/ccpm/issues/588))  
  - Confirmed strong `glab` CLI support for GitLab integration
  - Invited community contributor to submit existing GitLab implementation as PR
  - Updated project roadmap to include GitLab as priority platform

### 🐛 Clarified Platform Limitations
- **Windows Shell Compatibility** ([#609](https://github.com/automazeio/ccpm/issues/609))
  - Documented as Claude Code platform limitation (requires POSIX shell)
  - Provided workarounds and alternative solutions

- **Codex CLI Integration** ([#585](https://github.com/automazeio/ccpm/issues/585))
  - Explained future multi-AI provider support in new CLI architecture

- **Parallel Worker Agent Behavior** ([#530](https://github.com/automazeio/ccpm/issues/530))
  - Clarified agent role as coordinator, not direct coder
  - Provided implementation guidance and workarounds

### 🔒 Security
- **Privacy Documentation Fix** ([#630](https://github.com/automazeio/ccpm/issues/630))
  - Verified resolution via PR #631 (remove real repository references)

### 💡 Proposed Features
- **Bug Handling Workflow** ([#654](https://github.com/automazeio/ccpm/issues/654))
  - Designed `/pm:attach-bug` command for automated bug tracking
  - Proposed lightweight sub-issue integration with existing infrastructure
  - Community feedback requested on implementation approach

### 📊 Issues Resolved
**Closed**: 10 issues  
**Active Proposals**: 1 issue (#654)  
**Remaining Open**: 1 issue (#653)

#### Closed Issues:
- [#630](https://github.com/automazeio/ccpm/issues/630) - Privacy: Remove real repo references ✅  
- [#609](https://github.com/automazeio/ccpm/issues/609) - Windows shell error (platform limitation) ✅
- [#585](https://github.com/automazeio/ccpm/issues/585) - Codex CLI compatibility (architecture update) ✅  
- [#571](https://github.com/automazeio/ccpm/issues/571) - Figma MCP support (platform feature) ✅
- [#531](https://github.com/automazeio/ccpm/issues/531) - Use !bash in custom slash commands ✅
- [#323](https://github.com/automazeio/ccpm/issues/323) - Improve README.md ✅
- [#201](https://github.com/automazeio/ccpm/issues/201) - Local-only mode support ✅
- [#200](https://github.com/automazeio/ccpm/issues/200) - Multi-tracker support research ✅  
- [#588](https://github.com/automazeio/ccpm/issues/588) - GitLab support research ✅
- [#48](https://github.com/automazeio/ccpm/issues/48) - Context creation inaccuracies ✅
- [#530](https://github.com/automazeio/ccpm/issues/530) - Parallel worker coding operations ✅
- [#544](https://github.com/automazeio/ccpm/issues/544) - Auto-create labels during init ✅
- [#947](https://github.com/automazeio/ccpm/issues/947) - Project roadmap update ✅

### 🛠️ Technical Details
- **Files Modified**: 16 core files + documentation
- **New Files**: `LOCAL_MODE.md`, `CONTEXT_ACCURACY.md`  
- **Commands Updated**: All 14 PM slash commands modernized
- **Backward Compatibility**: Fully maintained
- **Dependencies**: No new external dependencies added

### 🏗️ Project Health
- **Issue Resolution Rate**: 83% (10/12 issues closed)
- **Documentation Coverage**: Significantly improved
- **Community Engagement**: Active contributor invitation and feedback solicitation
- **Code Quality**: Enhanced accuracy safeguards and validation

### 🚀 Next Steps
1. Community feedback on bug handling proposal (#654)
2. GitLab integration PR review and merge
3. Linear platform integration (pending demand)
4. Enhanced testing and validation workflows

---

*This release represents a major stability and usability milestone for CCPM, addressing the majority of outstanding community issues while establishing a foundation for future multi-platform support.*