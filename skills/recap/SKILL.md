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
