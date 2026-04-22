---
name: daily-plan-slack
description: Run the full daily-plan workflow, then send a digest to Matias's Slack DM. Safe to run scheduled (headless). Outputs plan file + Slack DM.
model_routing:
  default: balanced
  steps:
    data-gathering: fast
    synthesis: balanced
context: fork
---

## Purpose

Run `/daily-plan` in full, then send a concise digest of the plan to Matias's Slack DM. Designed to run both manually and as a scheduled 8 AM automation.

## Usage

- `/daily-plan-slack` — Generate today's plan and send Slack digest
- `/daily-plan-slack tomorrow` — Plan for tomorrow and send Slack digest

---

## Phase 1: Generate the Daily Plan

Execute the **complete** `/daily-plan` skill as defined in `.claude/skills/daily-plan/SKILL.md` — all steps (0 through 8), with full context gathering and file generation.

This will create the plan file at `07-Archives/Plans/YYYY-MM-DD.md`.

> **Scheduled context:** When running headlessly (no active user session), skip interactive steps:
> - Step 3 (Monday gate): proceed without prompting
> - Step 4 (yesterday's review): load file silently, skip if missing
> - Step 5.10a (Dex Inbox triage): surface in the Slack digest instead of prompting
> - Any confirmation prompts: resolve using defaults

---

## Phase 2: Build the Slack Digest

After the plan file is written, read `07-Archives/Plans/YYYY-MM-DD.md` and extract:

1. **Date header** — from the plan frontmatter
2. **Today's shape** — day type (stacked / moderate / open) + free block summary (one line)
3. **Focus items** — all P0/P1/P2 items from the "Today's Focus" section (max 5)
4. **Meetings** — each meeting's time, title, and prep note if present (from "Meetings" section)
5. **Commitments due** — items from the "Commitments Due Today" section
6. **Heads up** — the first 2 items from the "Heads Up" section (if any)
7. **Inbox items** — count of Dex Inbox captures pending triage (if any, from Step 5.10a)

### Slack message format

Use Slack's mrkdwn formatting. Keep the total message under 3000 characters.

```
📅 *Daily Plan — {Day}, {Month} {DD}*

*Today's shape:* {stacked/moderate/open} — {X meetings, ~Yh of focus time}

*Focus:*
• 🔴 {P0 item title}
• 🟡 {P1 item title}
• {P2 item title}

*Meetings ({count}):*
• {HH:MM AM/PM} — {Meeting title} {[Prep: one-line note] if present}

*Commitments due:*
• {commitment 1}
• {commitment 2}

{If heads up items exist:}
⚠️ *Heads up:*
• {item 1}
• {item 2}

{If Dex Inbox items pending:}
📱 *{N} items in Dex Inbox pending triage*

📁 Full plan: `07-Archives/Plans/{YYYY-MM-DD}.md`
```

**Rules:**
- Omit any section that has no content (don't show empty headers)
- If no P0 items exist, start with P1; label them neutrally (bullet, no emoji)
- Truncate any single line at 120 chars with `…`
- If the total message would exceed 3000 chars, drop Heads Up and Inbox sections first

---

## Phase 3: Send to Slack

1. **Look up user ID:**
   Call `slack_search_users` with query `matias.greco@fanduel.com`.
   Extract the `id` field from the first result (format: `U…`).
   If lookup fails, try query `Matias Greco`.

2. **Send DM:**
   Call `slack_send_message` with:
   - `channel_id`: the user ID from step 1
   - `message`: the formatted digest from Phase 2

3. **Confirm silently** (no output needed when running scheduled):
   If running interactively, print: `✅ Daily plan sent to Slack.`

---

## Error Handling

- **Plan file not created**: Report error and stop — do not send a blank Slack message
- **Slack user lookup fails**: Log warning to `System/errors.log`, skip Slack send, plan file is still valid
- **Slack send fails**: Log warning to `System/errors.log` with timestamp and error, do not retry
- **MCPs unavailable**: Follow the same graceful degradation as `/daily-plan` — a partial plan is fine; still send whatever was generated
