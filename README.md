# claude-code-recap

A Claude Code plugin that generates structured summaries of what happened in your git repository. Get a quick overview of who did what, which areas changed, and what the most active files were — all from a single slash command.

When the [GitHub CLI (`gh`)](https://cli.github.com/) is installed and authenticated, recaps are automatically enriched with pull request data: titles, AI-summarized descriptions, labels, review status, and linked issues. Without `gh`, everything works the same — just git-only.

## Installation

### Clone and load

```bash
git clone https://github.com/nikodem-wrona/claude-code-recap ~/.claude/plugins/claude-code-recap
claude --plugin-dir ~/.claude/plugins/claude-code-recap
```

### Local development

If you've cloned the repo elsewhere, point to it directly:

```bash
claude --plugin-dir /path/to/claude-code-recap
```

The `--plugin-dir` flag loads the plugin for a single session without installing it globally.

## Usage

```
/recap <duration>
```

Where `<duration>` is a number followed by a time unit:

| Unit | Meaning | Example |
|------|---------|---------|
| `h`  | Hours   | `/recap 24h` — last 24 hours |
| `d`  | Days    | `/recap 7d` — last 7 days |
| `w`  | Weeks   | `/recap 2w` — last 2 weeks |
| `m`  | Months  | `/recap 3m` — last 3 months |

The recap agent runs in the background — you can continue working while it analyzes the repository. You'll be notified when the summary is ready.

## Output Format

The recap is organized in layers, from high-level overview to specific details. When GitHub PR data is available, sections are enriched with PR context and an additional Pull Requests section is included.

### 1. Summary

A one-line overview of total activity.

```
12 commits by 3 contributors. 45 files changed (+1,200 / -340 lines).
Main areas: auth, API, infrastructure.
8 pull requests merged, covering 5 linked issues. Top labels: bug, feature, security.
```

The PR line appears only when `gh` is available and PRs were merged in the time window.

### 2. By Contributor

Per-person breakdown showing what each contributor worked on, with file change statistics. When PR data is available, narratives use PR titles and AI-summarized descriptions instead of raw commit messages, and a "Also reviewed" line lists PRs the contributor reviewed.

```
**Alice** (5 commits)
- Refactored auth module, added unit tests
- Files: src/auth/*.ts (8 files, +400/-120)
- Also reviewed: #38, #41

**Bob** (7 commits)
- New API endpoints for user management, fixed rate limiter
- Files: src/api/*.ts (6 files, +500/-80), src/middleware/rate-limiter.ts (+30/-15)
```

This section is automatically skipped when there's only one contributor.

### 3. By Area

Changes grouped by directory, showing aggregate stats and who worked in each area. When PR data is available, relevant labels from PRs touching each area are shown.

```
**src/auth/** — 8 files changed (+400/-120)
- Auth module refactor, new test coverage (Alice)
- Labels: refactor, security

**src/api/** — 6 files changed (+500/-80)
- New user management endpoints (Bob)
- Labels: feature
```

### 4. Pull Requests

*Only shown when `gh` is available.* Lists all merged PRs in the time window with AI-summarized descriptions, labels, review status, and linked issues.

```
| PR  | Author | Summary                          | Labels   | Reviewers   | Issues   |
|-----|--------|----------------------------------|----------|-------------|----------|
| #42 | alice  | Refactored session token storage | security | bob ✅      | #10, #12 |
| #41 | bob    | Added user CRUD endpoints        | feature  | alice ✅    | —        |
```

Review status: ✅ approved, 🔄 changes requested.

### 5. Most Changed Files

Top 10 files ranked by total churn (additions + deletions).

```
1. src/auth/session.ts (+150/-60)
2. src/api/users/controller.ts (+120/-10)
3. src/middleware/rate-limiter.ts (+30/-15)
```

## How It Works

The plugin has two components:

1. **`/recap` skill** — A thin dispatcher that parses your duration argument (e.g., `7d` becomes `7 days ago`) and launches the recap agent in the background.

2. **`recap-agent`** — A background agent that runs a series of git commands to collect commit history, contributor statistics, and file change data, then synthesizes everything into the structured summary.

The agent uses these git commands under the hood:
- `git log` — commit messages, authors, dates
- `git shortlog` — contributor commit counts
- `git log --numstat` — per-file addition/deletion counts
- `git diff --stat` — aggregate change statistics
- `git rev-list` — finding commit ranges

When the GitHub CLI is available, the agent also runs:
- `gh pr list` — batch-fetch merged PRs in the time window
- `gh pr view` — fetch linked/closing issues per PR

If `gh` is not installed or not authenticated, the recap falls back to git-only mode automatically.

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| No commits in range | Suggests trying a longer duration |
| Single contributor | Skips the "By Contributor" section |
| 100+ commits | Summarizes by theme instead of listing individually |
| Shallow clone | Works with available history, notes the limitation |
| Not a git repo | Shows a clear error message |
| Invalid duration format | Shows usage examples |
| `gh` not installed | Git-only recap with a note |
| `gh` not authenticated | Git-only recap with a note |
| Repo not on GitHub | Git-only recap (no error) |
| No PRs in time window | Sections render without PR enrichment |
| 50+ merged PRs | Linked-issue fetches capped at 50 |

## Plugin Structure

```
claude-code-recap/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── skills/
│   └── recap/
│       └── SKILL.md         # /recap slash command
├── agents/
│   └── recap-agent.md       # Background analysis agent
├── LICENSE
└── README.md
```

## Requirements

- [Claude Code](https://claude.ai/code) CLI with plugin support
- A git repository with commit history
- (Optional) [GitHub CLI (`gh`)](https://cli.github.com/) — for PR-enriched recaps

## License

MIT
