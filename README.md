# claude-code-recap

A Claude Code plugin that generates structured summaries of what happened in your git repository. Get a quick overview of who did what, which areas changed, and what the most active files were — all from a single slash command.

## Installation

### From a git repository

```bash
claude plugin install https://github.com/nikodem-wrona/claude-code-recap
```

### Local development

```bash
claude --plugin-dir /path/to/claude-code-recap
```

This loads the plugin for a single session without installing it globally. Useful for testing changes.

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

The recap is organized in four layers, from high-level overview to specific details:

### 1. Summary

A one-line overview of total activity.

```
12 commits by 3 contributors. 45 files changed (+1,200 / -340 lines).
Main areas: auth, API, infrastructure.
```

### 2. By Contributor

Per-person breakdown showing what each contributor worked on, with file change statistics.

```
**Alice** (5 commits)
- Refactored auth module, added unit tests
- Files: src/auth/*.ts (8 files, +400/-120)

**Bob** (7 commits)
- New API endpoints for user management, fixed rate limiter
- Files: src/api/*.ts (6 files, +500/-80), src/middleware/rate-limiter.ts (+30/-15)
```

This section is automatically skipped when there's only one contributor.

### 3. By Area

Changes grouped by directory, showing aggregate stats and who worked in each area.

```
**src/auth/** — 8 files changed (+400/-120)
- Auth module refactor, new test coverage (Alice)

**src/api/** — 6 files changed (+500/-80)
- New user management endpoints (Bob)
```

### 4. Most Changed Files

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

No external services or APIs are required — everything is derived from local git history.

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| No commits in range | Suggests trying a longer duration |
| Single contributor | Skips the "By Contributor" section |
| 100+ commits | Summarizes by theme instead of listing individually |
| Shallow clone | Works with available history, notes the limitation |
| Not a git repo | Shows a clear error message |
| Invalid duration format | Shows usage examples |

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

## License

MIT
