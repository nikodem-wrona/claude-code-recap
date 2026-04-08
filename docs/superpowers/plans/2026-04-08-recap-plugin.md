# Claude Code Recap Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that generates layered git history summaries via `/recap <duration>`.

**Architecture:** A thin skill (`/recap`) parses the user's duration argument and dispatches a background agent (`recap-agent`) that runs git commands and synthesizes a structured summary. Two components, three files total.

**Tech Stack:** Claude Code plugin system (plugin.json manifest, skill markdown, agent markdown). Git CLI for data gathering.

**Spec:** `docs/superpowers/specs/2026-04-08-recap-plugin-design.md`

---

## File Structure

```
repo-recap/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest (create)
├── skills/
│   └── recap/
│       └── SKILL.md             # /recap slash command (create)
├── agents/
│   └── recap-agent.md           # Background summarization agent (create)
├── docs/                        # Already exists
├── LICENSE                      # Already exists
└── README.md                    # Already exists
```

Three files to create. All are declarative (JSON/markdown) — no compiled code, no dependencies.

---

### Task 1: Create Plugin Manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

- [ ] **Step 1: Create the plugin manifest**

```json
{
  "name": "repo-recap",
  "version": "1.0.0",
  "description": "Generate layered summaries of what happened in your repository",
  "author": {
    "name": "Nikodem Wrona"
  },
  "license": "MIT",
  "keywords": ["git", "recap", "summary", "changelog"]
}
```

- [ ] **Step 2: Verify the manifest is valid JSON**

Run: `cat .claude-plugin/plugin.json | python3 -m json.tool > /dev/null && echo "Valid JSON"`
Expected: `Valid JSON`

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add plugin manifest"
```

---

### Task 2: Create the `/recap` Skill

**Files:**
- Create: `skills/recap/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: recap
description: Generate a summary of what happened in your repository over a given time period.
argument-hint: <duration: e.g. 7d, 2w, 24h>
user-invocable: true
allowed-tools: Agent
---

You are the recap dispatcher. Your ONLY job is to parse the user's duration argument and dispatch the `recap-agent` to do the actual work.

## Parse the duration

The user's input is: $ARGUMENTS

**Supported formats:** A number followed by a unit letter:
- `h` = hours (e.g., `24h`)
- `d` = days (e.g., `7d`)
- `w` = weeks (e.g., `2w`)
- `m` = months (e.g., `3m`)

**Conversion to git --since format:** Strip the unit letter, expand to the full word, append "ago". Examples:
- `24h` → `24 hours ago`
- `7d` → `7 days ago`
- `2w` → `2 weeks ago`
- `3m` → `3 months ago`

**If the input is missing or doesn't match the pattern** `<number><h|d|w|m>`, respond with:

> Invalid or missing duration. Usage: `/recap <duration>`
>
> Examples: `/recap 24h`, `/recap 7d`, `/recap 2w`, `/recap 3m`

and do NOT dispatch the agent.

## Dispatch the agent

If the duration is valid, dispatch the `recap-agent` using the Agent tool with these parameters:
- **subagent_type:** `recap-agent`
- **run_in_background:** `true`
- **prompt:** `Generate a repository recap for the last <converted duration>. Use --since="<converted duration>" for all git commands.`

Tell the user: "Generating recap for the last `<duration>`... I'll notify you when it's ready."
```

- [ ] **Step 2: Verify skill file structure**

Run: `head -5 skills/recap/SKILL.md`
Expected: The YAML frontmatter opening with `---` and the `name: recap` field.

- [ ] **Step 3: Commit**

```bash
git add skills/recap/SKILL.md
git commit -m "feat: add /recap skill for dispatching recap agent"
```

---

### Task 3: Create the Recap Agent

**Files:**
- Create: `agents/recap-agent.md`

- [ ] **Step 1: Create the agent file**

```markdown
---
name: recap-agent
description: Analyzes git history and produces a layered repository recap.
model: sonnet
tools: Bash
maxTurns: 15
background: true
color: cyan
---

You are a git history analyst. Your job is to analyze the repository's recent activity and produce a structured recap.

You will receive a prompt containing a `--since` duration (e.g., `--since="7 days ago"`). Use this value in ALL git commands below.

## Step 1: Verify git repository

Run:
```bash
git rev-parse --is-inside-work-tree
```

If this fails, respond with: "This directory is not a git repository. Please run `/recap` from within a git repo." and stop.

## Step 2: Collect data

Run these commands to gather repository activity data. If any command returns empty results, note it and continue with the data you have.

**2a. Commits list:**
```bash
git log --since="<duration>" --pretty=format:"%H|%an|%ad|%s" --date=short
```

If this returns nothing, respond with: "No commits found in the specified time period. Try a longer duration (e.g., `/recap 30d`)." and stop.

**2b. Contributor stats:**
```bash
git shortlog --since="<duration>" -sne HEAD
```

**2c. Per-file change stats across all commits:**
```bash
git log --since="<duration>" --numstat --pretty=format:""
```

**2d. Overall diff stat:**

First, find the earliest commit in range:
```bash
git rev-list --since="<duration>" --reverse HEAD | head -1
```

Then get the aggregate diff stat from that commit to HEAD:
```bash
git diff --stat <earliest_commit>^..HEAD
```

If the `^` parent reference fails (e.g., it's the root commit), use the commit itself:
```bash
git diff --stat <earliest_commit>..HEAD
```

**2e. Per-author file stats:**

For each contributor found in step 2b, run:
```bash
git log --since="<duration>" --author="<author name>" --numstat --pretty=format:""
```

## Step 3: Produce the recap

Using the collected data, produce a summary in EXACTLY this format:

```
## Repository Recap (last <duration>)

### Summary
<N> commits by <N> contributors. <N> files changed (+<additions> / -<deletions> lines).
Main areas: <top 3-5 directories by change volume>.

### By Contributor
**<Author Name>** (<N> commits)
- <2-3 sentence summary of what they did, derived from their commit messages>
- Files: <top changed directories/files with stats, e.g. src/auth/*.ts (8 files, +400/-120)>

<repeat for each contributor, ordered by commit count descending>

### By Area
**<directory path>** — <N> files changed (+<additions>/-<deletions>)
- <1-2 sentence summary of changes> (<contributor names>)

<repeat for top areas, ordered by change volume descending>

### Most Changed Files
1. <file path> (+<additions>/-<deletions>)
2. <file path> (+<additions>/-<deletions>)
<up to 10 files, ordered by total churn (additions + deletions) descending>
```

## Rules

- **Single contributor:** If there is only one contributor, skip the "By Contributor" section entirely. The Summary and By Area sections already cover who did what.
- **Large ranges (100+ commits):** Summarize, don't list. Group commits by theme/area. Use phrases like "and N more commits in this area" when a contributor or area has many similar changes.
- **Shallow clones:** If `git rev-list` returns fewer commits than expected or `^` references fail, work with what's available and add a note: "*Note: repository appears to be a shallow clone. Some history may be missing.*"
- **File stats:** Always aggregate per-file stats by directory for the "By Area" section. Use individual file paths only in "Most Changed Files."
- **Commit message analysis:** Read commit messages to understand WHAT changed, not just WHERE. The recap should tell a story, not just list statistics.
```

- [ ] **Step 2: Verify agent file structure**

Run: `head -8 agents/recap-agent.md`
Expected: The YAML frontmatter with `name: recap-agent`, `model: sonnet`, `tools: Bash`, `maxTurns: 15`, `background: true`, `color: cyan`.

- [ ] **Step 3: Commit**

```bash
git add agents/recap-agent.md
git commit -m "feat: add recap-agent for git history analysis"
```

---

### Task 4: Test the Plugin End-to-End

- [ ] **Step 1: Verify plugin structure**

Run: `find . -not -path './.git/*' -not -path './docs/*' -type f | sort`
Expected output should include:
```
./.claude-plugin/plugin.json
./LICENSE
./README.md
./agents/recap-agent.md
./skills/recap/SKILL.md
```

- [ ] **Step 2: Validate plugin loads without errors**

Run: `claude --plugin-dir . --print-plugins 2>&1 || echo "Note: --print-plugins may not be a real flag; use the manual test below instead"`

If the flag doesn't exist, test manually:
```bash
claude --plugin-dir .
```
Then type `/recap 7d` in the session. Verify:
1. The skill is listed when typing `/`
2. It dispatches the recap-agent
3. The agent runs git commands and produces the layered summary

- [ ] **Step 3: Test with various durations**

In the same Claude Code session with `--plugin-dir .`:
- `/recap 24h` — should work (hours)
- `/recap 7d` — should work (days)
- `/recap 2w` — should work (weeks)
- `/recap 1m` — should work (months)
- `/recap` — should show usage error (missing argument)
- `/recap abc` — should show usage error (invalid format)

- [ ] **Step 4: Update README**

Update `README.md` with installation and usage instructions:

```markdown
# repo-recap

A Claude Code plugin that helps you understand what happened in your repository while you were away.

## Installation

```bash
claude plugin install <published-url-or-path>
```

Or for local development:
```bash
claude --plugin-dir /path/to/repo-recap
```

## Usage

```
/recap 24h     # What happened in the last 24 hours
/recap 7d      # What happened in the last 7 days
/recap 2w      # What happened in the last 2 weeks
/recap 3m      # What happened in the last 3 months
```

## Output

The recap includes:
- **Summary** — total commits, contributors, files changed, main areas
- **By Contributor** — per-person breakdown with file stats
- **By Area** — changes grouped by directory
- **Most Changed Files** — top 10 files by churn
```

- [ ] **Step 5: Final commit**

```bash
git add README.md
git commit -m "docs: update README with installation and usage instructions"
```
