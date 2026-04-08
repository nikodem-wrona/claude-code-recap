# Privacy Policy

**repo-recap** is a Claude Code plugin that runs entirely on your local machine.

## Data Collection

This plugin does **not** collect, store, or transmit any personal data. It does not phone home, use analytics, or communicate with any external service.

## What the Plugin Accesses

When you run `/recap`, the plugin executes local commands on your machine:

- **Git commands** (`git log`, `git shortlog`, `git diff`, etc.) to read your repository's commit history.
- **GitHub CLI commands** (`gh pr list`, `gh pr view`) to fetch pull request metadata from GitHub — only if `gh` is installed, authenticated, and the repository is hosted on GitHub. These requests go directly from your machine to GitHub using your own credentials.

All output stays within your Claude Code session.

## Third Parties

The plugin itself makes no third-party requests. GitHub CLI usage is governed by [GitHub's Privacy Statement](https://docs.github.com/en/site-policy/privacy-policies/github-general-privacy-statement).

## Changes

If this policy changes, updates will be posted in this file in the repository.

## Contact

For questions, open an issue at [github.com/nikodem-wrona/repo-recap](https://github.com/nikodem-wrona/repo-recap/issues).
