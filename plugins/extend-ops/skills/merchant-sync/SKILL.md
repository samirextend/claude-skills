---
name: merchant-sync
description: >
  Lightweight end-of-week sync that updates merchant project files with the past 5 days
  of activity (Gmail + Slack + Jira) — fast, targeted, no full research sweep. Runs
  automatically every Friday at 5 PM so project files are fresh for the Monday pulse.
  Also supports ad hoc use mid-week: when Samir shares context ("I emailed the URL
  parameter docs to Eric today", "CarParts replied", "I closed that ticket"), run this
  to log it immediately without waiting for Friday.

  Use when Samir says "sync [merchant]", "log this for [merchant]", "quick sync",
  "I sent/emailed/replied [context] for [merchant]", "update [merchant] with [context]",
  "end of week sync", "merchant sync", "eow sync", "close this action item", or
  "mark [X] as done for [merchant]". Also runs on the Friday 5 PM schedule.
---

# Merchant Sync

Keeps project files current between full `refresh-merchant` runs. Covers the past 5 days
of Gmail, Slack, Jira, and call recordings (Chorus, Gemini, Zoom) — then writes resolved
items, updated context, and net-new action items back to each project file.

**This is not a full research sweep.** No Salesforce, Calendar, or GDrive deep-dive.
Those belong in `refresh-merchant`. This skill runs in minutes, not 10+.

**Base directory:** Resolve relative paths from this file's own location.
Project files live at: `{skill_base_dir}/../../../Claude/memory/projects/`

---

## Modes

### Scheduled mode (Friday 5 PM — all active merchants)
Process every active merchant automatically. Active = Status contains "In Progress" OR
has at least one non-struck-through Open Action Item. Skip dormant merchants (one-line
acknowledgment only).

### Ad hoc mode (Samir provides context mid-week)
Process one named merchant. Samir's stated context is the primary signal — "I emailed X
to Y on Friday" is treated as authoritative. The source sweep supplements it (confirms
the send, picks up any reply) but is not required to close an item. If Samir says it's
done, it's done.

---

## Step 1 — Determine mode and scope

**Is there a named merchant + explicit context in the message?** → Ad hoc mode.  
**Is this a scheduled run or a general "sync all" request?** → Scheduled mode, all active merchants.

For scheduled mode, list all `.md` files in the projects directory and read them in
parallel to identify active merchants.

---

## Step 2 — Read the project file(s)

For each merchant being processed, read:
`{skill_base_dir}/../../../Claude/memory/projects/{merchant}.md`

Extract and hold in memory:
- Every open action item (non-struck-through `[ ]` bullets in `## Open Action Items`)
- Key contacts: merchant domain(s), individual email addresses, Slack channel ID
- `Also search as` aliases
- Last researched date
- Extend-side contacts (TAM, MSM, SA) for DM reads

---

## Step 3 — Run the 5-day source sweep (all in parallel)

### Gmail

1. **Inbound:** `from:@[merchant-domain] newer_than:5d`
2. **Outbound:** `to:@[merchant-domain] newer_than:5d`
3. **Keyword:** `[merchant name] newer_than:5d` — repeat for each alias in "Also search as"
4. **Known contacts:** `from:[contact@email] newer_than:5d` for any merchant-side contacts with individual email addresses (not just domain)

For every thread returned: call `get_thread` with `messageFormat: FULL_CONTENT`. Don't
rely on snippets — the latest reply in a long-running thread is the real signal and
snippets won't show it.

### Slack

1. Read the primary channel from the project file (`slack_read_channel`, 5-day window)
2. Search aliases: `slack_search_public_and_private` for merchant name + each alias from "Also search as", filtered `after:[5 days ago]`
3. Read DMs with the key Extend contacts in the project file (TAM, MSM, SA):
   - Use `slack_search_users` to get user IDs if not already known
   - Call `slack_read_channel` with each person's user ID as the channel ID
4. For any message with thread replies, call `slack_read_thread` to read the full thread

### Jira

Run: `text ~ "[merchant name]" AND updated >= -5d ORDER BY updated DESC`  
Repeat for each alias. For any tickets returned, fetch key fields:
`["summary", "status", "assignee", "comment"]`

### Notes Doc — 1-week window

Read the implementation notes doc URL from the project file. Fetch the last 1 week of
content only — request the final portion of the doc since recent entries are at the bottom.
If the tool supports a limit or startIndex parameter, use it. If not, fetch the full doc
and discard anything with a date older than 7 days.

This captures manual notes written by anyone on the team (Vince, Jordan, GSM) that
wouldn't surface in Gmail or Slack. Skip if no notes doc URL is in the project file.

### Call Recordings (Chorus, Gemini, Zoom) — 5-day window

Run all three in parallel. These capture call outcomes that won't appear in Gmail or Slack.

**Chorus:**
Search Gmail: `subject:[merchant name] "meeting insights ready for review" newer_than:5d`
Also run for each alias. For each result, use Method A (read_document on Gmail URL) or
Method B (get_thread + extract embedded URL) — whichever returns substantive content.

**Gemini:**
- Method A: `parentId = '1qU9GdzkiMDYiXvqHXrVPTUfimulg2zmH' and title contains '[merchant name]' and modifiedTime > '[5 days ago RFC3339]'` — fetch each doc found
- Method B: Gmail search `from:gemini-notes@google.com [merchant name] newer_than:5d` — get_thread with FULL_CONTENT for each result
Run both methods. Run each for merchant name and all aliases.

**Zoom:**
`search_zoom` with `entity_type: zoom_doc`, merchant name, filtered to last 5 days.
Repeat for each alias. For each result, call `get_file_content` with the file_id.

If any of the three return zero results, mark ✅ "no results found" — do not skip without running.

---

## Step 4 — Cross-check open action items for completion

For each open `[ ]` action item in the project file, scan the 5-day sweep results:

**Mark complete** if any source shows:
- An email confirming the item was done ("sent", "here's the doc", "submitted", "done", "pushed")
- A Slack message confirming resolution
- A Jira ticket moved to Done that corresponds to the item

**Update with new context** (leave open) if:
- Partial progress found — e.g., email sent but no reply yet, ticket still in progress
- Reply received that changes the scope or timeline

**Add new item** if:
- Something new surfaced in the 5-day window that isn't captured in any existing item
- A merchant reply or Slack message raised a new question or ask

**Ad hoc context takes priority:** If Samir stated in the message that something is done,
mark it complete regardless of whether the sweep independently confirms it. His direct
knowledge is more reliable than searching for evidence of it.

---

## Step 5 — Write back to the project file

Make surgical updates only — do not rewrite sections or reorganize content.

**`## Open Action Items` section:**
- Change `[ ]` to `[x]` for completed items; append a brief completion note:
  `[x] ~~[original item text]~~ — ✅ [date] via [email/Slack/Jira/Samir]`
- Add new `[ ]` bullets for net-new items at the top of the open list
- Update the section header date: `## Open Action Items (as of [today])`
- If an item is definitively done (not just partial), also move it to `## Already Resolved`

**`## Key Decisions` section:**
- Append any confirmed new decisions from the 5-day window
- Format: `- **[Date]:** [what was decided]`
- Only add if something was genuinely decided — don't add speculative or still-open items

**`## Last researched` field (in Overview):**
- Update to today's date

**Do not touch:** Background/Context, Recurring Topics, Contacts, Notes Doc, Jira Search
Config, Gemini/Chorus sections. Those belong to `refresh-merchant`.

---

## Step 6 — Output summary

After writing back, output a brief summary in the conversation:

```
## Merchant Sync — [Today's Date]

[Scheduled / Ad hoc] — [N merchants processed]

### [Merchant Name]
✅ Resolved ([N]): [item names, comma-separated]
➕ New items ([N]): [item names, or "none"]
📝 Updated context ([N]): [items with new info but still open, or "none"]
Sources: Gmail [N threads] | Slack [N messages/threads] | Jira [N tickets updated] | Notes doc [read/skipped] | Calls [N recaps found: Chorus/Gemini/Zoom]

---
[Repeat per merchant]

---
Project files updated. Monday pulse will reflect these changes.
[If any source returned zero results, note it: "Gmail — no threads found for [merchant]"]
```

---

## Constraints

- **5-day lookback only** — do not pull data older than 5 days. This keeps runs fast and
  avoids duplicating what refresh-merchant already captured.
- **No full research sweep sources** — do not call Salesforce, Google Calendar, or GDrive
  (folder browsing or doc reading). Those belong in `refresh-merchant`. If you find
  yourself reaching for one, stop.
- **Surgical writes only** — modify `## Open Action Items`, `## Key Decisions` (if
  net-new decisions), and `## Last researched`. Nothing else.
- **Append-only for `## Already Resolved`** — only add; never remove.
- **Trust Samir's direct input** — if he says something is done, mark it done. Don't
  require the sweep to independently verify what he's telling you directly.
- **Speed** — if a source sweep returns no results for a merchant, note it and move on.
  A clean week is valid; don't treat zero results as an error.
