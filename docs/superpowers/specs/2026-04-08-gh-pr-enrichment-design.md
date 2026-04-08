# GitHub PR Enrichment — Design Spec

## Overview

Enhance the existing recap agent to optionally enrich its output with GitHub pull request data using the locally installed `gh` CLI. When available, the recap gains PR titles, AI-summarized descriptions, labels, review status, and linked issues — woven into existing sections and presented in a dedicated new PR section.

## Goals

- Enrich recap output with PR context when `gh` is available and the repo is GitHub-hosted
- Gracefully degrade to the current git-only recap when `gh` is unavailable or unauthenticated
- Keep the existing architecture (skill dispatches background agent) unchanged
- Minimize API calls via batch-fetching merged PRs for the time window

## Non-Goals

- Standalone PR lookup command (e.g. `/recap-pr 123`)
- Support for non-GitHub hosts (GitLab, Bitbucket)
- Caching PR data between runs

## Architecture

No structural changes. The recap agent gains a new optional data collection step (Step 2.5) and its output format is extended. The skill and plugin manifest are unchanged.

## Data Collection: Step 2.5 — GitHub PR Data (Optional)

This step runs after the existing git data collection (Step 2) and before output generation (Step 3).

### 2.5a. Detect `gh` availability

Run:
```bash
command -v gh && gh auth status 2>&1
```

If either fails, set `gh_available=false`, skip the rest of Step 2.5, and continue to Step 3. No error message — the recap proceeds as a git-only recap with a single note at the top of the output.

### 2.5b. Batch-fetch merged PRs

Run:
```bash
gh pr list --state merged --search "merged:>YYYY-MM-DD" \
  --json number,title,body,labels,reviews,mergeCommit,author,closedAt,url \
  --limit 200
```

The `YYYY-MM-DD` date is derived from the same `--since` duration used for git log. The agent calculates this by running:
```bash
date -v-<N><unit> +%Y-%m-%d
```
For example, for "7 days ago": `date -v-7d +%Y-%m-%d`. On Linux, use `date -d "7 days ago" +%Y-%m-%d` instead. The agent should detect the platform and use the appropriate syntax.

This returns a JSON array of all PRs merged within the recap window.

### 2.5c. Fetch linked issues

For each PR from step 2.5b, run:
```bash
gh pr view <number> --json closingIssuesReferences
```

This returns the issues that the PR closes/resolves. Batch this by looping through all PR numbers. To limit API calls, cap at 50 PRs — if more than 50 were merged, process the 50 most recent and note "and N more PRs" in the output.

### 2.5d. Match PRs to commits

Compare `mergeCommit.oid` from each PR against the commit SHAs collected in Step 2a. This creates a `commit SHA → PR` mapping. Commits without a matching PR remain git-only (their commit messages are used as-is in the output).

## Output Changes

### Graceful Degradation Note

When `gh` is unavailable, add a single line at the very top of the recap output:

```
*Note: GitHub CLI not available — showing git-only recap. Install `gh` and run `gh auth login` for PR-enriched recaps.*
```

All sections render identically to the current behavior. The Pull Requests section is omitted entirely.

### Summary (modified)

After the existing summary line, append:

```
<N> pull requests merged, covering <N> linked issues. Top labels: `label1`, `label2`, `label3`.
```

If no PRs were found in the window (even though `gh` is available), omit this line.

### By Contributor (modified)

The 2-3 sentence narrative summary is enhanced:
- For commits linked to PRs, use the PR title and AI-summarized description instead of raw commit messages. This produces a higher-level narrative.
- For commits not linked to PRs, fall back to commit messages as today.
- Add a reviewer line: "Also reviewed: #42, #45" listing PRs where this contributor left reviews (if any).

### By Area (modified)

After the existing stats and summary line, append relevant labels from PRs that touched files in that directory:

```
**src/auth/** — 8 files changed (+400/-120)
- Auth module refactor, new test coverage (Alice)
- Labels: `security`, `refactor`
```

Labels are deduplicated and sorted alphabetically. If no PRs with labels touched the area, omit the labels line.

### Pull Requests (NEW section)

Inserted between "By Area" and "Most Changed Files". Lists all merged PRs in the recap window, ordered by merge date descending:

```
### Pull Requests

| PR | Author | Summary | Labels | Reviewers | Issues |
|----|--------|---------|--------|-----------|--------|
| #42 | alice | 1-2 sentence AI summary of PR description | `bug`, `auth` | bob ✅ | #10, #12 |
| #41 | bob | 1-2 sentence AI summary of PR description | `feature` | alice ✅, carol 🔄 | — |
```

Column details:
- **PR**: PR number, linked to the PR URL
- **Author**: PR author login
- **Summary**: Agent-generated 1-2 sentence summary of the PR body. If the body is empty or just a template, use the PR title instead.
- **Labels**: Comma-separated label names. "—" if none.
- **Reviewers**: Reviewer names with status icons: ✅ approved, 🔄 changes requested. "—" if no reviews.
- **Issues**: Linked/closed issue numbers. "—" if none.

If more than 50 PRs were merged, show the top 50 and add a note: "*and N more pull requests*".

### Most Changed Files (unchanged)

No modifications.

### Active Branches (unchanged)

No modifications.

## Agent Changes

### Frontmatter

Increase `maxTurns` from 15 to 20 to accommodate the additional `gh` commands and the richer summarization work.

```yaml
maxTurns: 20
```

### New Rules

Add to the agent's Rules section:

- **GitHub data is optional:** If `gh` is not installed or not authenticated, produce the recap without PR data. Add the graceful degradation note at the top.
- **PR description summarization:** Condense each PR's body into 1-2 sentences that capture what the PR accomplished and why. Strip template boilerplate (checkboxes, "## Description" headers, etc.) before summarizing. If the body is empty or only contains template text, use the PR title as the summary.
- **API efficiency:** Use batch `gh pr list` to fetch all merged PRs at once. Only use per-PR `gh pr view` calls for linked issues. Cap at 50 PRs to limit API calls.
- **Label deduplication:** When showing labels in the "By Area" section, collect labels from all PRs touching that area and deduplicate.

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| `gh` not installed | Git-only recap with degradation note |
| `gh` installed but not authenticated | Git-only recap with degradation note |
| Repo not on GitHub (e.g. local-only) | `gh pr list` will fail — treat as `gh` unavailable |
| No PRs merged in the time window | Skip PR-related additions to all sections; omit Pull Requests section |
| PR with empty body | Use PR title as summary |
| PR body is only a template | Strip template boilerplate, use PR title if nothing meaningful remains |
| 50+ merged PRs | Process top 50 by merge date, note remainder |
| Squash-merged PRs where merge commit SHA doesn't match | These PRs won't match any commit — they still appear in the Pull Requests table but won't enrich the By Contributor/By Area sections |
| Rate limiting from `gh` API | If a `gh` command fails mid-collection, use whatever PR data was gathered so far and continue |
