# GitHub PR Enrichment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enhance the recap agent to optionally enrich its output with GitHub PR data (titles, summaries, labels, reviews, linked issues) using the `gh` CLI, with graceful degradation when unavailable.

**Architecture:** Single-file modification to `agents/recap-agent.md`. The agent gains a new optional Step 2.5 for GitHub data collection and its output template is extended. No changes to the skill, manifest, or plugin structure.

**Tech Stack:** Markdown (agent prompt), `gh` CLI, `git` CLI, Bash

---

## File Map

- Modify: `agents/recap-agent.md` (the recap agent prompt — all changes happen here)
- Modify: `docs/superpowers/specs/2026-04-08-recap-plugin-design.md` (remove PR integration from out-of-scope)

---

### Task 1: Update Agent Frontmatter

**Files:**
- Modify: `agents/recap-agent.md:1-9`

- [ ] **Step 1: Increase maxTurns**

Change the frontmatter `maxTurns` from `15` to `20` to accommodate the additional `gh` commands:

```yaml
---
name: recap-agent
description: Analyzes git history and produces a layered repository recap.
model: sonnet
tools: Bash
maxTurns: 20
background: true
color: cyan
---
```

- [ ] **Step 2: Commit**

```bash
git add agents/recap-agent.md
git commit -m "chore: increase recap agent maxTurns to 20 for PR enrichment"
```

---

### Task 2: Add Step 2.5a — Detect `gh` Availability

**Files:**
- Modify: `agents/recap-agent.md` (insert after Step 2f, before Step 3)

- [ ] **Step 1: Add the detection step**

Insert the following after the "2f. Active branches" section and before "## Step 3: Produce the recap":

```markdown
## Step 2.5: Collect GitHub PR data (optional)

This step enriches the recap with pull request context. It requires the `gh` CLI to be installed and authenticated. If `gh` is unavailable, skip this entire step — the recap proceeds as git-only.

**2.5a. Detect `gh` availability:**

Run:
```bash
command -v gh && gh auth status 2>&1
```

If either command fails (non-zero exit), skip all remaining 2.5 steps and go directly to Step 3. Set a mental flag that GitHub data is unavailable — you will need this in Step 3 to add the degradation note.
```

- [ ] **Step 2: Commit**

```bash
git add agents/recap-agent.md
git commit -m "feat: add gh CLI detection step to recap agent"
```

---

### Task 3: Add Step 2.5b — Batch-Fetch Merged PRs

**Files:**
- Modify: `agents/recap-agent.md` (append after Step 2.5a)

- [ ] **Step 1: Add the batch-fetch step**

Append after Step 2.5a:

```markdown
**2.5b. Calculate date and batch-fetch merged PRs:**

First, calculate the cutoff date for the `gh` search query. The `--since` duration you received (e.g., "7 days ago") must be converted to a `YYYY-MM-DD` date.

Detect the platform and compute the date:
```bash
if date -v-1d +%Y-%m-%d 2>/dev/null; then
  # macOS (BSD date)
  CUTOFF_DATE=$(date -v-<N><unit> +%Y-%m-%d)
else
  # Linux (GNU date)
  CUTOFF_DATE=$(date -d "<N> <unit> ago" +%Y-%m-%d)
fi
```

Replace `<N>` and `<unit>` with the values from the duration (e.g., for "7 days ago": `-v-7d` on macOS, `-d "7 days ago"` on Linux).

Then fetch all merged PRs in the window:
```bash
gh pr list --state merged --search "merged:>$CUTOFF_DATE" \
  --json number,title,body,labels,reviews,mergeCommit,author,closedAt,url \
  --limit 200
```

This returns a JSON array. Parse it and hold it in memory for the remaining steps. If this command fails (e.g., the repo is not on GitHub), treat it as `gh` unavailable and skip to Step 3.
```

- [ ] **Step 2: Commit**

```bash
git add agents/recap-agent.md
git commit -m "feat: add batch PR fetch step to recap agent"
```

---

### Task 4: Add Step 2.5c — Fetch Linked Issues

**Files:**
- Modify: `agents/recap-agent.md` (append after Step 2.5b)

- [ ] **Step 1: Add the linked issues step**

Append after Step 2.5b:

```markdown
**2.5c. Fetch linked issues:**

For each PR from step 2.5b (capped at 50 PRs — if more were returned, use the 50 most recent by `closedAt`), fetch linked/closing issues:

```bash
gh pr view <number> --json closingIssuesReferences
```

Loop through all PR numbers. For each PR, record which issue numbers it closes. If a `gh pr view` call fails, skip that PR and continue with the rest.

If there were more than 50 PRs, note the count of remaining PRs — you will include "*and N more pull requests*" in the output.
```

- [ ] **Step 2: Commit**

```bash
git add agents/recap-agent.md
git commit -m "feat: add linked issues fetch step to recap agent"
```

---

### Task 5: Add Step 2.5d — Match PRs to Commits

**Files:**
- Modify: `agents/recap-agent.md` (append after Step 2.5c)

- [ ] **Step 1: Add the matching step**

Append after Step 2.5c:

```markdown
**2.5d. Match PRs to commits:**

Compare `mergeCommit.oid` from each PR (from step 2.5b) against the commit SHAs collected in Step 2a. Build a mapping of `commit SHA → PR data`.

- Commits with a matching PR: their narrative in "By Contributor" and "By Area" sections will use the PR title and summarized description instead of the raw commit message.
- Commits without a matching PR: use their commit message as-is (current behavior).
- PRs without a matching commit (e.g., squash merges where the SHA changed): these still appear in the "Pull Requests" table but don't enrich the per-commit sections.
```

- [ ] **Step 2: Commit**

```bash
git add agents/recap-agent.md
git commit -m "feat: add PR-to-commit matching step to recap agent"
```

---

### Task 6: Update Output Format — Summary and By Contributor

**Files:**
- Modify: `agents/recap-agent.md` (modify the output template in Step 3)

- [ ] **Step 1: Update the Summary section in the output template**

In Step 3's output format, replace the Summary block:

```markdown
### Summary
<N> commits by <N> contributors. <N> files changed (+<additions> / -<deletions> lines).
Main areas: <top 3-5 directories by change volume>.
```

With:

```markdown
### Summary
<N> commits by <N> contributors. <N> files changed (+<additions> / -<deletions> lines).
Main areas: <top 3-5 directories by change volume>.
<if GitHub data available and PRs found:>
<N> pull requests merged, covering <N> linked issues. Top labels: `label1`, `label2`, `label3`.
```

- [ ] **Step 2: Update the By Contributor section in the output template**

Replace the By Contributor block:

```markdown
### By Contributor
**<Author Name>** (<N> commits)
- <2-3 sentence summary of what they did, derived from their commit messages>
- Files: <top changed directories/files with stats, e.g. src/auth/*.ts (8 files, +400/-120)>

<repeat for each contributor, ordered by commit count descending>
```

With:

```markdown
### By Contributor
**<Author Name>** (<N> commits)
- <2-3 sentence summary of what they did. For commits linked to PRs, use the PR title and a 1-2 sentence AI-generated summary of the PR body instead of raw commit messages. For commits not linked to PRs, use commit messages as before.>
- Files: <top changed directories/files with stats, e.g. src/auth/*.ts (8 files, +400/-120)>
<if GitHub data available:>
- Also reviewed: #<PR number>, #<PR number> <list PRs where this person left reviews, if any>

<repeat for each contributor, ordered by commit count descending>
```

- [ ] **Step 3: Commit**

```bash
git add agents/recap-agent.md
git commit -m "feat: enrich Summary and By Contributor sections with PR data"
```

---

### Task 7: Update Output Format — By Area, Pull Requests Section

**Files:**
- Modify: `agents/recap-agent.md` (modify the output template in Step 3)

- [ ] **Step 1: Update the By Area section in the output template**

Replace the By Area block:

```markdown
### By Area
**<directory path>** — <N> files changed (+<additions>/-<deletions>)
- <1-2 sentence summary of changes> (<contributor names>)

<repeat for top areas, ordered by change volume descending>
```

With:

```markdown
### By Area
**<directory path>** — <N> files changed (+<additions>/-<deletions>)
- <1-2 sentence summary of changes> (<contributor names>)
<if GitHub data available and PRs with labels touched this area:>
- Labels: `label1`, `label2` <deduplicated, sorted alphabetically>

<repeat for top areas, ordered by change volume descending>
```

- [ ] **Step 2: Add the new Pull Requests section**

Insert a new section in the output template between "By Area" and "Most Changed Files":

```markdown
<if GitHub data available and PRs were found:>
### Pull Requests

| PR | Author | Summary | Labels | Reviewers | Issues |
|----|--------|---------|--------|-----------|--------|
| #<number> | <author login> | <1-2 sentence AI summary of PR body; use PR title if body is empty or only template boilerplate> | `label1`, `label2` or — | <reviewer> ✅ or 🔄, or — | #<issue>, #<issue> or — |

<list all merged PRs ordered by merge date descending>
<if more than 50 PRs: add "*and N more pull requests*" at the end>

Review status icons: ✅ = approved, 🔄 = changes requested.
```

- [ ] **Step 3: Commit**

```bash
git add agents/recap-agent.md
git commit -m "feat: add PR labels to By Area and new Pull Requests section"
```

---

### Task 8: Add Graceful Degradation Note and New Rules

**Files:**
- Modify: `agents/recap-agent.md` (modify Step 3 intro and Rules section)

- [ ] **Step 1: Add degradation note instruction**

At the beginning of Step 3 (before the output format), add:

```markdown
**If GitHub data is unavailable** (Step 2.5 was skipped), add this line at the very top of the recap output, before the Summary:

```
*Note: GitHub CLI not available — showing git-only recap. Install `gh` and run `gh auth login` for PR-enriched recaps.*
```

When GitHub data is unavailable, produce all sections exactly as before (without PR enrichment) and omit the "Pull Requests" section entirely.
```

- [ ] **Step 2: Add new rules to the Rules section**

Append the following rules to the existing `## Rules` section at the end of the file:

```markdown
- **GitHub data is optional:** If `gh` is not installed or not authenticated, produce the recap without PR data. Add the graceful degradation note at the top. All existing sections render exactly as before. The "Pull Requests" section is omitted entirely.
- **PR description summarization:** Condense each PR's body into 1-2 sentences that capture what the PR accomplished and why. Strip template boilerplate (checkboxes, `## Description` headers, horizontal rules, empty sections) before summarizing. If the body is empty or only contains template text, use the PR title as the summary.
- **API efficiency:** Use the batch `gh pr list` command to fetch all merged PRs at once. Only use per-PR `gh pr view` calls for linked issues. Cap linked-issue fetches at 50 PRs to limit API calls.
- **Label deduplication:** When showing labels in the "By Area" section, collect labels from all PRs that touch files in that directory, deduplicate, and sort alphabetically.
- **Rate limit resilience:** If any `gh` command fails mid-collection (after the initial availability check passed), use whatever PR data has been gathered so far and continue to Step 3. Do not abort the entire recap because of a single failed API call.
```

- [ ] **Step 3: Commit**

```bash
git add agents/recap-agent.md
git commit -m "feat: add graceful degradation and GitHub-specific rules to recap agent"
```

---

### Task 9: Update Original Design Spec

**Files:**
- Modify: `docs/superpowers/specs/2026-04-08-recap-plugin-design.md:170-176`

- [ ] **Step 1: Update out-of-scope section**

In the original design spec, change the "Out of Scope" section. Remove the line:

```markdown
- Pull request / issue integration (GitHub API)
```

And replace it with:

```markdown
- Pull request / issue integration (GitHub API) — **moved in-scope**, see [GitHub PR Enrichment spec](2026-04-08-gh-pr-enrichment-design.md)
```

- [ ] **Step 2: Commit**

```bash
git add docs/superpowers/specs/2026-04-08-recap-plugin-design.md
git commit -m "docs: update original spec to reference PR enrichment feature"
```

---

### Task 10: End-to-End Verification

**Files:**
- Read: `agents/recap-agent.md` (verify final state)

- [ ] **Step 1: Read and review the complete agent file**

Read `agents/recap-agent.md` in full. Verify:
- Frontmatter has `maxTurns: 20`
- Step 2.5 sections (a through d) appear after Step 2f and before Step 3
- Step 3 output template includes: enriched Summary, enriched By Contributor, enriched By Area with labels, new Pull Requests table, graceful degradation note
- Rules section includes all 5 new rules
- No placeholders, TODOs, or incomplete sections remain

- [ ] **Step 2: Verify `gh` is available in this environment**

```bash
command -v gh && gh auth status 2>&1
```

If `gh` is available, proceed to Step 3. If not, skip to Step 4 (git-only verification).

- [ ] **Step 3: Test with `gh` available**

Run `/recap 30d` against this repository. Verify:
- The agent runs without errors
- If PRs exist, the Pull Requests table appears with correct columns
- Summary includes PR count and labels
- By Contributor narratives reference PR titles where applicable
- By Area shows labels where applicable

- [ ] **Step 4: Test graceful degradation**

To simulate `gh` being unavailable, temporarily test against a local-only repo (no GitHub remote) or verify the agent's logic by reading the output carefully. The degradation note should appear and all sections should render as the original git-only format.

- [ ] **Step 5: Final commit if any fixes needed**

If any issues were found and fixed during verification:
```bash
git add agents/recap-agent.md
git commit -m "fix: address issues found during PR enrichment verification"
```
