# 👁️ Toggle (ToggleX) — The Context Layer

> **Context layer for your agent.** ToggleX captures your work sessions, projects, focus scores, and context switches across the web — giving your agent awareness of what you've actually been doing.

[![Version](https://img.shields.io/badge/version-1.0.5-blue)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()

---

## What It Does

Unlike skills that only know what you *tell* your agent, Toggle knows what you *did*. It fetches structured activity data from the ToggleX platform and enables your agent to:

- **Auto-sync** — Check your activity every hour so the agent always has context
- **Daily digests** — End-of-day summary, no prompt needed
- **Smart nudges** — Flag projects you haven't touched in a while
- **Focus alerts** — Heads up when you're context-switching too much
- **Pattern detection** — Spot repeated workflows and suggest automations
- **Instant recall** — Answer questions like *"What was I working on Tuesday?"*

---

## Requirements

- `python3`
- A `TOGGLE_API_KEY` (see [Getting Your API Key](#getting-your-api-key))

---

## Getting Your API Key

1. Visit [https://x.toggle.pro/new/clawbot-integration](https://x.toggle.pro/new/clawbot-integration)
2. Generate your key
3. Add it to your OpenClaw config:

```json
{
  "skills": {
    "entries": {
      "toggle": {
        "apiKey": "your_key_here"
      }
    }
  }
}
```

Or export it in your shell:

```bash
export TOGGLE_API_KEY=your_key_here
```

> **Never paste your API key into chat.**

---

## Quick Start

| Action | Command |
|--------|---------|
| Fetch today's data | `python3 {baseDir}/scripts/toggle.py` |
| Fetch a date range | `python3 {baseDir}/scripts/toggle.py --from-date 2026-02-17 --to-date 2026-02-19` |
| Fetch + save to memory | `python3 {baseDir}/scripts/toggle.py --persist {baseDir}/../../memory` |
| Cron run (skip cron check) | `python3 {baseDir}/scripts/toggle.py --persist {baseDir}/../../memory --skip-cron-check` |

> `{baseDir}` refers to the root directory of this skill's installation (the folder containing this file). This is standard OpenClaw skill convention.

---

## API Endpoint

```
https://ai-x.toggle.pro/public-openclaw/workflows
```

Operated by [ToggleX](https://x.toggle.pro). Your `TOGGLE_API_KEY` is sent as an `x-openclaw-api-key` header. No other data is transmitted.

---

## Data Schema

Each workflow entry returned by the API includes:

| Field | Description |
|-------|-------------|
| `type` | `"WORK"`, `"BREAK"`, or `"LEISURE"` |
| `workflowType` | Category of work (e.g. `"coding"`, `"research"`, `"communication"`) |
| `workflowDescription` | Human-readable summary of the session |
| `projectTask.name` | Task context (if present) |
| `project.name` | Project context (if present) |
| `productivityScore` | 0–100 focus rating |
| `startTime` | ISO 8601 timestamp (session start) |
| `endTime` | ISO 8601 timestamp (session end; `null` if ongoing) |
| `duration` | Session length in seconds |

### Productivity Score Guide

| Range | Meaning |
|-------|---------|
| **90–100** | Sharp / deep focus |
| **70–89** | Solid |
| **Below 70** | Fragmented |

### Interpretation Rules

- Focus on `type: "WORK"` entries; skip `BREAK` unless asked; mention `LEISURE` briefly if present.
- Always use `startTime` to sort entries chronologically when analyzing sequences or patterns.
- If `totalWorkflows` is `0`, Toggle wasn't running or captured nothing for that period.

---

## Proactive Behaviors

These features run automatically when data is available — no user prompt required.

### 1. Daily Digest

**Trigger:** Cron job at the user's `digest_time` (default 6:00 PM), or the first interaction after end of workday.

Produces a short 3–4 line summary covering total work time, top focus area, biggest open item, and one notable stat (focus trend, longest session, or context-switch count).

Example:

> Your day: 5.2h across 4 sessions. Deepest focus on auth migration (2.1h, 94 focus). Stripe webhooks still open — you were 80% through. Context switches down to 3 from 6 yesterday.

### 2. Stale Project Nudges

A project is flagged as stale when:
- It has **>2 hours of total accumulated time** (a real project, not a one-off)
- It has **no activity in the last `nudge_stale_hours`** (default: 48h)

The agent surfaces up to 2 stale nudges per interaction and respects dismissals.

### 3. Context-Switch Alerts

Triggered when **all three conditions** are true within the last 2 hours:
1. 3+ different projects within a 30-minute window
2. Average productivity score below `focus_alert_threshold` (default: 30)
3. The scattered pattern spans at least `focus_alert_window_min` minutes (default: 20)

Alerts are limited to once per 3-hour window and can be paused by the user.

### 4. Pattern Detection & Automation Proposals

After 7+ days of persisted data, the agent scans for repeated workflow sequences (e.g. *GitHub → Notion → Slack* every morning) and proposes automations when a pattern appears 3+ times.

### 5. Instant Recall

Responds to questions like:
- *"What was I working on yesterday?"*
- *"Where did I leave off on [project]?"*
- *"Pick up where I left off"*

---

## Cron Setup

### Hourly Sync

```bash
openclaw cron create \
  --name "Toggle hourly sync" \
  --schedule "0 * * * *" \
  --message "Run: python3 {baseDir}/scripts/toggle.py --persist {baseDir}/../../memory --skip-cron-check"
```

### Daily Digest (default 6 PM)

```bash
openclaw cron create \
  --name "Toggle daily digest" \
  --schedule "0 18 * * *" \
  --message "Fetch today's Toggle data and generate my end-of-day digest. Run: python3 {baseDir}/scripts/toggle.py --persist {baseDir}/../../memory --skip-cron-check"
```

---

## Configuration

All preferences are stored in `{baseDir}/state.yaml`.

### User Preferences

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `cron_disabled` | bool | `false` | Skip cron status check |
| `digest_enabled` | bool | `true` | Generate end-of-day digests |
| `digest_time` | string | `"18:00"` | Digest time (24h format) |
| `nudge_stale_hours` | int | `48` | Hours of inactivity before stale nudge |
| `focus_alert_threshold` | int | `30` | Score below this triggers alert |
| `focus_alert_window_min` | int | `20` | Minutes of low-focus before alerting |
| `pattern_detection_days` | int | `7` | Days of history to scan for patterns |
| `pattern_min_occurrences` | int | `3` | Repeats required to propose automation |

### Tracking State (managed by the agent)

| Key | Type | Description |
|-----|------|-------------|
| `last_nudged` | map | Project → last nudge timestamp (24h cooldown) |
| `dismissed_projects` | list | Projects the user dropped (never nudge again) |
| `last_focus_alert` | string | Last alert timestamp (3h backoff) |
| `focus_alert_paused_until` | string | Skip focus alerts until this date |
| `proposed_patterns` | map | Pattern → timestamp (14-day cooldown) |
| `last_error` | string | Last cron error (cleared after reporting) |

---

## Persisting to Memory

```bash
python3 {baseDir}/scripts/toggle.py --persist {baseDir}/../../memory
```

Writes a `<!-- toggle-data-start -->` / `<!-- toggle-data-end -->` section inside `<date>.md` files (one per day). Existing content outside that section is preserved.

Always use `--persist` when running via cron or when saving/refreshing data.

---

## Error Handling

| Error | Likely Cause | Resolution |
|-------|-------------|------------|
| `HTTP 401` | Invalid or expired API key | Regenerate at [x.toggle.pro](https://x.toggle.pro/new/clawbot-integration) |
| `HTTP 403` | Insufficient permissions | Check ToggleX integration settings |
| `HTTP 429` | Rate limited | Retry in a few minutes |
| `HTTP 5xx` | ToggleX server issue | Retry shortly |
| `Request failed` / timeout | Network issue | Check internet connection |
| `TOGGLE_API_KEY is not set` | Missing env var | Add key to OpenClaw config |
| JSON parse error | Malformed response | Usually temporary — retry |

---

## Links

- **ToggleX Platform:** [https://x.toggle.pro](https://x.toggle.pro)
- **Get API Key:** [https://x.toggle.pro/new/clawbot-integration](https://x.toggle.pro/new/clawbot-integration)
