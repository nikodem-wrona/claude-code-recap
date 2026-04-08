# claude-code-recap

A Claude Code plugin that helps you understand what happened in your repository while you were away.

## Installation

```bash
claude plugin install <published-url-or-path>
```

Or for local development:
```bash
claude --plugin-dir /path/to/claude-code-recap
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
