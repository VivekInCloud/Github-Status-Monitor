# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GitHub Status Monitor is a GitHub Actions workflow that polls the [GitHub Status API](https://www.githubstatus.com/api/v2/incidents/unresolved.json) every 5 minutes and sends Slack alerts when incidents persist across consecutive runs. The key design goal is eliminating alert fatigue — alerts fire only when an incident is confirmed across two polling cycles, not immediately on first detection.

## Repository Structure

```
.github/workflows/github-status-monitor.yaml   # The executable workflow (source of truth)
github-status-monitor.yaml                     # Root-level copy for reference/documentation
README.md                                       # Full setup guide and customization examples
```

There is no build system, package manager, or test suite. The project is pure YAML + shell.

## Architecture

### State Management via Git

The workflow stores incident state by committing `previous_incidents.txt` back to the repository after each run. This file contains newline-separated incident IDs from the last successful poll. On the next run, `comm -12` compares the sorted previous and current incident ID lists — only IDs present in **both** files trigger a Slack alert.

### Two-Phase Alert Logic (core pattern)

```
Run N:   fetch API → write current_incidents.txt → compare with previous_incidents.txt
                      (no match yet = no alert)    → commit current as new previous

Run N+1: fetch API → write current_incidents.txt → match found = send Slack alert
```

This means a new incident will not alert until the **second consecutive run** that sees it.

### Data Flow

```
GitHub Status API (/api/v2/incidents/unresolved.json)
  └─ jq: extract .incidents[].id → current_incidents.txt
         comm -12 sorted(previous, current) → persistent_incidents.txt
         for each persistent ID: extract name/status/impact → format Slack payload
         curl POST → SLACK_WEBHOOK_URL secret
         git commit + push → previous_incidents.txt updated
```

## Required Setup

- `SLACK_WEBHOOK_URL` secret must be set in **Settings → Secrets and variables → Actions**
- The workflow needs `contents: write` permission (already set) to commit state back to the repo

## Making Changes

All logic lives in the `run:` shell block of `.github/workflows/github-status-monitor.yaml`. The root-level `github-status-monitor.yaml` is a documentation copy — keep it in sync when modifying the workflow.

### Changing the poll interval

Edit the cron expression in `on.schedule`:
```yaml
- cron: '*/5 * * * *'   # every 5 minutes (minimum GitHub allows is 5)
```

### Filtering by specific GitHub services

Add a `jq` filter on the incident `components` array before writing `current_incidents.txt`. See README for examples targeting Actions, Webhooks, etc.

### Switching alert destination (e.g., Teams, PagerDuty)

Replace the `curl` Slack POST at the end of the `run:` block with the target webhook format. The `$incident_details` variable already contains the formatted text.
