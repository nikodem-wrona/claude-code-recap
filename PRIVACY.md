# Privacy Policy

**repo-recap** is a Claude Code plugin that runs entirely on your local machine.

## Data Collection

This plugin does **not** independently collect, store, or transmit any personal data. It does not phone home or use analytics.

## What the Plugin Accesses

When you run `/recap`, the plugin operates within Claude Code and communicates with the following services:

- **Anthropic (Claude Code)** — The plugin runs as a Claude Code agent. Repository data (commit messages, author names, file paths, diff statistics) is sent to Anthropic's API as part of the Claude Code session. This is governed by [Anthropic's Privacy Policy](https://www.anthropic.com/privacy).
- **GitHub** — If the GitHub CLI (`gh`) is installed and authenticated, the plugin fetches pull request metadata (titles, descriptions, labels, reviewers, linked issues) from GitHub's API using your own credentials. This is governed by [GitHub's Privacy Statement](https://docs.github.com/en/site-policy/privacy-policies/github-general-privacy-statement).

The plugin itself has no server, database, or telemetry of its own.

## Changes

If this policy changes, updates will be posted in this file in the repository.

## Contact

For questions, open an issue at [github.com/nikodem-wrona/repo-recap](https://github.com/nikodem-wrona/repo-recap/issues).
