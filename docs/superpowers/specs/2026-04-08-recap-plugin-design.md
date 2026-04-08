# Claude Code Recap Plugin — Design Spec

## Overview

A Claude Code plugin that generates layered summaries of what happened in a git repository over a specified time period. The user invokes `/recap <duration>` and receives a structured summary covering commits, contributors, changed areas, and file statistics.

## Goals

- Provide a quick, structured overview of repository activity for any time period
- Work with any git repository (no GitHub/external service dependency)
- Run in the background without cluttering the user's main conversation
- Produce a layered output: high-level summary, by contributor, by area, most changed files

## Plugin Structure

```
claude-code-recap/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── skills/
│   └── recap/
│       └── SKILL.md         # /recap slash command — parses args, dispatches agent
├── agents/
│   └── recap-agent.md       # Background agent that runs git commands + builds summary
├── LICENSE
└── README.md
```

## Manifest

```json
{
  "name": "claude-code-recap",
  "version": "1.0.0",
  "description": "Generate layered summaries of what happened in your repository",
  "author": {
    "name": "Nikodem Wrona"
  },
  "license": "MIT",
  "keywords": ["git", "recap", "summary", "changelog"]
}
```

Minimal manifest. No MCP servers, hooks, or bin scripts.

## Component 1: `/recap` Skill

**Location:** `skills/recap/SKILL.md`

**Frontmatter:**
```yaml
name: recap
description: Generate a summary of what happened in your repository over a given time period.
argument-hint: <duration: e.g. 7d, 2w, 24h>
user-invocable: true
allowed-tools: Agent
```

**Behavior:**
- Parses `$ARGUMENTS[0]` for a duration string
- Supported units: `h` (hours), `d` (days), `w` (weeks), `m` (months)
- Converts duration to git-compatible `--since` format:
  - `24h` → `24 hours ago`
  - `7d` → `7 days ago`
  - `2w` → `2 weeks ago`
  - `3m` → `3 months ago`
- Git natively understands all these units, so the conversion is a direct mapping: strip the unit letter, expand to the full word, append "ago".
- Validates the format — if invalid, tells the user the expected format with examples
- Dispatches `recap-agent` in the background with the parsed duration

**Example usage:**
```
/recap 7d      → recap of the last 7 days
/recap 2w      → recap of the last 2 weeks
/recap 24h     → recap of the last 24 hours
/recap 3m      → recap of the last 3 months
```

## Component 2: Recap Agent

**Location:** `agents/recap-agent.md`

**Frontmatter:**
```yaml
name: recap-agent
description: Analyzes git history and produces a layered repository recap.
model: sonnet
tools: Bash
maxTurns: 15
background: true
color: cyan
```

> **Decision note (2026-04-08):** Using `sonnet` for speed since this is a summarization task, not complex reasoning. `maxTurns: 15` gives enough room for all git commands plus edge case handling. These values may need tuning based on real-world usage — if summaries are too shallow, consider `opus`; if the agent finishes in 5 turns consistently, `maxTurns` can be reduced.

### Git Commands

The agent runs these commands in sequence:

1. **Collect commits:**
   ```bash
   git log --since="<duration>" --pretty=format:"%H|%an|%ae|%ad|%s" --date=short
   ```
   Gets all commits with hash, author, email, date, and message.

2. **Get contributor stats:**
   ```bash
   git shortlog --since="<duration>" -sne
   ```
   Commit count per contributor.

3. **Get overall file change stats:**
   ```bash
   git diff --stat $(git rev-list --since="<duration>" --reverse HEAD | head -1)^..HEAD
   ```
   Total files changed, insertions, deletions.

4. **Get per-file diff stats:**
   ```bash
   git log --since="<duration>" --numstat --pretty=format:""
   ```
   Per-file additions/deletions across all commits.

5. **Get per-author file stats:**
   ```bash
   git log --since="<duration>" --author="<name>" --numstat --pretty=format:""
   ```
   Repeated per contributor for the "by contributor" section.

### Output Format

```markdown
## Repository Recap (last <duration>)

### Summary
12 commits by 3 contributors. 45 files changed (+1,200 / -340 lines).
Main areas: auth, API, infrastructure.

### By Contributor
**Alice** (5 commits)
- Refactored auth module, added unit tests
- Files: src/auth/*.ts (8 files, +400/-120)

**Bob** (7 commits)
- New API endpoints for user management, fixed rate limiter
- Files: src/api/*.ts (6 files, +500/-80), src/middleware/rate-limiter.ts (+30/-15)

### By Area
**src/auth/** — 8 files changed (+400/-120)
- Auth module refactor, new test coverage (Alice)

**src/api/** — 6 files changed (+500/-80)
- New user management endpoints (Bob)

### Most Changed Files
1. src/auth/session.ts (+150/-60)
2. src/api/users/controller.ts (+120/-10)
3. ...
```

## Edge Cases & Error Handling

- **No commits in range:** Report "No commits found in the last `<duration>`." and suggest trying a longer period.
- **Single contributor:** Skip the "By Contributor" heading — show summary + areas + files directly (no per-person breakdown needed for one author).
- **Very large ranges (hundreds of commits):** Cap detailed commit message analysis at a reasonable depth. Group remaining commits as "and N more commits in this area." Summarize, don't list.
- **Shallow clones:** If `git rev-list` fails due to shallow history, fall back to available commits and note the limitation in the output.
- **No git repo:** Tell the user clearly that they need to be in a git repository.
- **Merge commits:** No special handling — they're part of the history and contain useful context.

## Out of Scope

- Pull request / issue integration (GitHub API) — **moved in-scope**, see [GitHub PR Enrichment spec](2026-04-08-gh-pr-enrichment-design.md)
- Natural language time parsing ("last night", "since Monday")
- MCP servers, hooks, or bin scripts
- Interactive / conversational flow for specifying time ranges
