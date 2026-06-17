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

---

## Accuracy Mandate

The output of this skill directly populates Samir's project files, which he uses in live
client conversations, executive meetings, and go-live decisions. Missing a key email
thread, an internal Slack decision, or a new action item has real downstream consequences.

**Thoroughness and accuracy are not optional.** Speed is secondary to correctness.

Specific rules that are non-negotiable:
- Every Gmail thread returned by `search_threads` MUST be opened with `get_thread`.
  Snippets are not sufficient. A thread can contain 40+ messages and show a year-old
  snippet — the real signal is always in the most recent reply.
- Every `slack_read_channel` call MUST use the timestamp computed in Step 0 via bash.
  A mentally-calculated timestamp is not acceptable and has caused data loss before.
- If time or context pressure would cause you to shortcut either of the above, state it
  explicitly in the output rather than silently skipping. A "PARTIAL SWEEP" label is
  better than a silent gap.

**Base directory:** Resolve relative paths from this file's own location.
Project files live at: `{skill_base_dir}/../../../Claude/memory/projects/`

---

## Modes

### Scheduled mode (Friday 5 PM — all merchants)
Process every merchant according to their tier. Read project file status to determine tier:

**Full active sync** — Status = "In Progress" or "Live (Hypercare)":
Warby Parker, CarParts, Peloton, Sleep Number.
Run the complete Step 3 sweep (Gmail, Slack, Jira, Notes doc, Chorus, Gemini, Zoom).

**Processing order is fixed — always process in this sequence:**
1. Warby Parker
2. CarParts
3. Peloton
4. Sleep Number

Do not reorder. Higher-priority merchants must complete their full pipeline (Phase 1 + Phase 2 + all source reads + verification gate + write-back) before the next merchant begins. This ensures that if context pressure forces an early stop, the highest-risk merchants are already fully processed.

**Minimal pulse** — Status = "Live" or "On Hold":
505 Mattress, City Beauty, Caleres, Rooms To Go, DNA Mattress, RugsUSA.
Run Step 3b only (see below) — three sources, fast check for unexpected activity.

**Skip entirely** — Status = "Not Assigned":
Glamnetic. One-line acknowledgment, no research.

### Ad hoc mode (Samir provides context mid-week)
Process one named merchant. Samir's stated context is the primary signal — "I emailed X
to Y on Friday" is treated as authoritative. The source sweep supplements it (confirms
the send, picks up any reply) but is not required to close an item. If Samir says it's
done, it's done.

---

## Step 0 — Compute timestamps (PREREQUISITE GATE — do this before anything else)

**Do not proceed to Step 1 until this step is complete and the values are in hand.**

Run all three bash commands and record the output:

```bash
date -d '6 days ago 00:00:00' +%s    # 6-day midnight — oldest= param for all regular Slack channel reads
date -d '8 days ago 00:00:00' +%s    # 8-day midnight — oldest= param for Chorus DM D057X0PEFJS and Gemini queries ONLY
date +%Y-%m-%d                        # Today's date string for Jira queries and write-back dates
```

**Why midnight anchoring:** Without `00:00:00`, `date -d 'N days ago' +%s` computes relative to the exact moment the command runs. If the sync runs at 5 PM Friday, "5 days ago" = Sunday 5 PM — a message posted Monday at 9 AM is only 4 days 16 hours earlier and falls inside the window, but one posted at 4:30 PM Sunday would be excluded. By anchoring to midnight, the window always covers the full boundary day regardless of when the sync runs.

**Why 6 days for regular channels:** The scheduled sync runs Fridays at 5 PM CST. A 6-day midnight window covers Saturday midnight 6 days ago — capturing all of Monday through Friday plus a full day of buffer. This is the correct window for "capture everything from the current work week."

Store all three values with their labels before proceeding. Labels matter — using the 6-day timestamp for Chorus or the 8-day timestamp for regular Slack channels is a silent data error.

- Every `slack_read_channel` call for regular channels (primary, secondary, group DMs, 1:1 DMs) uses the **6-day midnight timestamp**.
- Every call to `D057X0PEFJS` (Chorus) and every Gemini GDrive/Gmail query uses the **8-day midnight timestamp**.
- No overrides, no mental math, no exceptions.

**Timestamp verification (mandatory after the first Slack read):** After completing the first `slack_read_channel` call, check the date of the newest message returned. If the newest message is older than 6 days, the timestamp is wrong — stop, recompute via bash, and retry before continuing. `slack_read_channel` returns no warning when the timestamp is stale; it silently returns historical content that looks valid.

**This is a gate, not a suggestion.** If you have not run this command and recorded
the output, stop. Run it now. Only then continue to Step 1.

**Why this matters:** A wrong timestamp causes `slack_read_channel` to silently return
historical content with no warning — results look like valid data but are weeks or months old.
This has caused missed decisions, missed action items, and incorrect project file updates in
production runs. Gmail (`newer_than:Nd`) and Jira (JQL date filters) are server-computed and
not subject to this problem. Only Slack Unix timestamps are fragile.

---

## Step 1 — Determine mode and scope

**Is there a named merchant + explicit context in the message?** → Ad hoc mode.
**Is this a scheduled run or a general "sync all" request?** → Scheduled mode, all merchants.

For scheduled mode, list all `.md` files in the projects directory and read them in
parallel. Check each merchant's Status field to assign the correct tier.

---

## Step 2 — Read the project file(s) and RAID log

For each merchant being processed, read:
`{skill_base_dir}/../../memory/projects/{merchant}.md`

Extract and hold in memory:
- Every open action item (non-struck-through `[ ]` bullets in `## Open Action Items`)
- Key contacts: merchant domain(s), individual email addresses, Slack channel ID
- `Also search as` aliases
- Last researched date
- Extend-side contacts (TAM, MSM, SA) for DM reads
- RAID log sheet file ID (from `## GDrive Subfolder` section)

**Also read the merchant's Implementation Plan and RAID Log data.** Before reading, check the project file's `## RAID Log` section for `**Structure:** Dedicated`:

**If dedicated (WP, CarParts, Sleep Number):** The RAID log is a separate file from the project plan.
- Read **RAID log rows** from the **dedicated RAID sheet** — use the file ID from `## RAID Log` section.
- Read the **Implementation Plan sheet** from `## GDrive Subfolder`. After reading, **validate tab names** — identify tabs by their actual name, not position. Route the tab named for the project plan/phases → `## Phase Breakdown`; the tab named for custom requirements → `## Custom Requirements Summary`. Do not assume a fixed tab order.

**If combined (Peloton and others):** Read the single sheet (file ID from `## RAID Log`). After reading, **validate tab names** — identify tabs by their actual name. Route: project plan tab → `## Phase Breakdown`; RAID log tab → RAID context; custom requirements tab → `## Custom Requirements Summary`. Skip any tab that does not exist yet. Do not assume a fixed tab order.

In both structures: RAID log rows → used to prevent drift when marking action items complete. Phase/milestone data → populates `## Phase Breakdown` in Step 5. Custom requirements data → populates `## Custom Requirements Summary` in Step 5.

Why this matters: merchant-sync marks action items complete in the project file based on
Gmail/Slack/Jira signals. Without reading the RAID log, it may mark items complete or add
new items that are already tracked or resolved there — creating drift between the project
file and the RAID log. Reading all tabs also seeds the stable context sections that
downstream skills (confluence-sync, RAID log delta mode) rely on.

**MANDATORY: Reading the RAID log is non-negotiable.** Do not skip because the project file "looks current," "was recently refreshed," or "baseline appears stable." Those are not valid reasons. The only valid reason to skip or partially read the RAID log is a hard tool error (tool call returns an error response). If a hard tool error occurs, label the output PARTIAL SWEEP and state which file/tab could not be read and the exact error.

If the file ID is not in the project file: note it as a gap, attempt to locate the sheet via GDrive search using the merchant name, and document the result. Do not silently skip.

---

## Step 3 — Run the 5-day source sweep (all in parallel)

### ⛔ MANDATORY SOURCE EXECUTION CHECKLIST — read before running any source

Every source below must run for every merchant in scope. Before writing back any merchant's project file in Step 5, check every box:

```
[ ] Gmail Phase 1 — all four queries run, thread IDs collected and counted
[ ] Gmail Phase 2 — get_thread MINIMAL called for EVERY Phase 1 thread ID (count must match)
[ ] Gmail Phase 2 — get_thread FULL_CONTENT called for every thread with a message in the 5d window
[ ] Slack — primary channel read with Step 0 timestamp
[ ] Slack — all secondary channels and group DMs from project file read
[ ] Slack — 1:1 DMs with every listed Extend-side contact (TAM, MSM, SA) read
[ ] Slack — public/private alias search run
[ ] Slack — new channel IDs from alias search compared against project file; any undocumented IDs written to ## Slack section and read immediately
[ ] Jira — query run for merchant name and all aliases
[ ] Notes doc — HEAD fetched (or absence of URL explicitly confirmed)
[ ] GDrive — Step 4c delta check run on all catalogued file IDs in ## Key Reference Docs
[ ] GDrive — Glean backstop query run (app:gdrive [merchant name] after:[last_sync_date]) for merchant name and all aliases
[ ] GDrive — all docs surfaced this run (Gmail share notifications, Slack links, Glean results) catalogued with file ID + excerpt before Step 5
[ ] Chorus — Slack DM D057X0PEFJS read for 8d window
[ ] Gemini — GDrive title search run for 8d window (Gmail fallback if zero results)
[ ] Zoom — checked Recording Tools field; either run or explicitly marked N/A
[ ] Step 4c — reference doc metadata check run; changed docs re-fetched with delta note
```

**None of these boxes can be left unchecked without a hard tool error as justification.** These are not optional steps. The following are NOT valid reasons to skip any source:

- "The project file was recently refreshed"
- "Baseline appears stable"
- "Low signal expected for this merchant"
- "Context is getting long"
- "Already had enough information"
- Zero results from a prior step

If a source is skipped for any reason other than a hard tool error (tool call returned an explicit error response), the output must be labeled **PARTIAL SWEEP** and every skipped source must be named. A labeled partial sweep is always preferable to unacknowledged missing data — but it is still a failure mode, not an acceptable outcome.

**This checklist is a pre-write gate.** If any box is unchecked when Step 5 begins: stop, run the missing source, then continue. Do not write back incomplete data.

---

### Gmail — two-phase structured loop (no shortcuts)

**Phase 1 — Collect all thread IDs:**
Run all four queries in parallel. Deduplicate thread IDs across results.

1. **Inbound:** `from:@[merchant-domain] newer_than:5d`
2. **Outbound:** `to:@[merchant-domain] newer_than:5d`
3. **Keyword:** `[merchant name] newer_than:5d` — repeat for each alias in "Also search as"
4. **Known contacts:** `from:[contact@email] newer_than:5d` for any merchant-side contacts with individual email addresses

After Phase 1: record the total number of unique thread IDs collected.
Example: "Warby Parker — Phase 1 complete: 8 unique threads."

**Phase 2 — Open every thread (no exceptions):**

`search_threads` only surfaces the FIRST FEW messages in each thread as a preview
snippet. The most recent reply — which is always the actual signal — is NOT shown
unless you open the thread. A thread with 40 messages where someone replied today will
show a message from months ago. This has caused missed key decisions and missed action
items in production runs.

For every thread ID collected in Phase 1:

1. Call `get_thread` with `messageFormat: MINIMAL`. This returns all message dates
   without full body content. Check the date of the LAST message in the thread.
2. If last message is within 5 days: call `get_thread` with `messageFormat: FULL_CONTENT`.
3. If last message is older than 5 days: log "thread [ID] — last message [date], outside
   window, skipping FULL_CONTENT" and move on.

After Phase 2: record the counts.
Example: "Warby Parker — 8 threads: 5 opened FULL_CONTENT, 3 outside 5d window."

**These counts are required input to the Step 4b verification gate. Do not skip them.**

**Phase 1 and Phase 2 for each merchant must be fully completed before moving to the next merchant.** Do not run Phase 1 across multiple merchants and then batch Phase 2 afterward. Complete the full Gmail pipeline (Phase 1 + Phase 2) for Merchant A, then begin Merchant B. This ensures that if a context limit is hit mid-run, high-priority merchants are fully processed and lower-priority merchants are the ones left incomplete — not the reverse.

### Slack

**Before making any `slack_read_channel` call: confirm the Step 0 timestamp is in hand.
Every single `slack_read_channel` call uses that timestamp as the `oldest` parameter.
If you do not have the Step 0 timestamp, stop and run the bash command now.**

1. Read the primary channel from the project file — `slack_read_channel` with
   `oldest: [Step 0 timestamp]`, `limit: 100`
2. Read any secondary channels or group DMs listed in the project file — same params
3. Read DMs with key Extend-side contacts (TAM, MSM, SA) listed in the project file —
   use their user IDs as the `channel_id`; same `oldest` param
4. Search aliases: `slack_search_public_and_private` for merchant name + each alias,
   filtered `after:[5 days ago date string]`
5. **New channel capture (runs after alias search):** From the alias search results,
   extract every unique channel ID that appeared. Compare against all channels already
   documented in the project file's `## Slack` section. For any channel ID NOT already
   listed:
   - **Write it to the project file immediately** — add it to the `## Slack` section
     with the channel ID, any available channel name, and the tag `[discovered — verify and label]`
     so it's available on every future run without relying on keyword search to surface it again.
   - **Read it immediately** — call `slack_read_channel` with the Step 0 timestamp.
     If it contains merchant discussion, include the content in this sweep.
   Note: this catches group DMs and working channels that aren't formally named after the
   merchant but contain relevant conversations. Once the ID is written to the project file,
   it gets explicitly read on every future run.
6. For any message with thread replies, call `slack_read_thread` to read the full thread

After Slack reads: record how many channels were read and confirm each used the Step 0 timestamp. If any channel returned zero messages, note it explicitly — do not silently skip it as "no activity."

**All four Slack sub-steps (primary channel, secondary channels/DMs, 1:1 Extend contact DMs, alias search) are mandatory.** Skipping the 1:1 DMs is the most common failure mode — these are the highest-signal source for decisions and escalations. If a contact's Slack ID is not in the project file, note it and attempt `slack_search_users` to find it before skipping. Only a hard tool error justifies skipping any sub-step.

### Jira

Run two parallel workstreams — both are required every sync. Results feed separate project file sections.

**Workstream 1 — Text/alias search (feeds `## Open Ticket Statuses`):**

Run for the merchant name AND each alias:
`text ~ "[merchant name]" AND updated >= -5d ORDER BY updated DESC`

For any tickets returned, fetch key fields: `["summary", "status", "assignee", "comment"]`

This is the wide net — catches tickets across any project that mention the merchant, regardless of whether they're under the TAM epic. Results overwrite `## Open Ticket Statuses` each run (see write-back rules below).

**Workstream 2 — TAM epic traversal (feeds `## Jira Epic Hierarchy`):**

This runs unconditionally — not gated on the 5-day update window. The goal is a current snapshot of the full implementation hierarchy regardless of whether anything changed this week.

1. **Find the TAM epic.** Check the project file's `## Jira Epic Hierarchy` section for a stored epic key (e.g., `TAM-618`). If present, use it directly — skip the search.
   If not present, run: `project = TAM AND issuetype = Epic AND text ~ "[merchant name]" ORDER BY updated DESC` (repeat for each alias). Take the most relevant result. Write the epic key to the project file immediately so future runs skip this step.

2. **Fetch all children.** Run: `parent = [epic-key] ORDER BY created ASC`
   For each child returned, call `getJiraIssue` with:
   `fields: ["summary", "status", "assignee", "issuelinks", "updated"]`

3. **Follow cross-project links.** For each child where `issuelinks` is non-empty:
   - For each linked issue that is not Done/Closed: call `getJiraIssue` with `fields: ["summary", "status", "assignee"]`
   - Include all non-Done linked issues regardless of link type — In Progress stories are as relevant as blocked ones

4. **Write `## Jira Epic Hierarchy`.** Overwrite this section each sync (see format below).

**Format for `## Jira Epic Hierarchy`:**

```markdown
## Jira Epic Hierarchy
_Last updated: [YYYY-MM-DD]_

### [TAM-NNN](url) — [Epic summary] (Epic, [Status])

- [TAM-NNN](url) — [Child summary] | [Status] | [Assignee]
  - → [PROJ-NNN](url) — [Linked issue summary] | [Status] | [Assignee]
- [TAM-NNN](url) — [Child summary] | [Status] | [Assignee]
  - → [PROJ-NNN](url) — [Linked issue summary] | [Status] | [Assignee]
```

If no TAM epic exists for this merchant, write:
```markdown
## Jira Epic Hierarchy
_Last updated: [YYYY-MM-DD] — no TAM epic found_
```

If the epic exists but has no children yet, write:
```markdown
## Jira Epic Hierarchy
_Last updated: [YYYY-MM-DD]_

### [TAM-NNN](url) — [Epic summary] (Epic, [Status])
_No child tasks yet._
```

### Notes Doc — 15k character HEAD fetch

Read the implementation notes doc URL from the project file. Fetch the first 15,000
characters using `read_file_content` or `read_document` with `startIndex=0`.
Notes docs are newest-at-top — the HEAD is where recent content lives. Do not fetch
from the tail or end of the document.

This captures manual notes written by anyone on the team (Vince, Jordan, GSM) that
wouldn't surface in Gmail or Slack. Skip only if no notes doc URL is in the project file.

**MANDATORY: Do not skip because the project file "looks current," "was recently refreshed," or "baseline appears stable." Those are not valid reasons.** The only valid reason to skip is a hard tool error or the literal absence of a notes doc URL in the project file. If a tool error occurs, label the output PARTIAL SWEEP and state the error. "Recently refreshed" is never acceptable as a skip justification.

### GDrive — Glean backstop (net-new doc discovery)

No folder scan runs during merchant-sync. Known docs are covered by Step 4c (direct file ID delta check). New docs shared explicitly are caught by Gmail share notifications in the Gmail step.

The one gap: docs created or uploaded directly into a merchant folder without triggering a share notification. Catch these with a single Glean query per merchant:

`app:gdrive [merchant name] after:[last_sync_date]`

Run for the merchant name and each alias separately. For any doc returned that is not already catalogued in `## Key Reference Docs` or a dedicated project file section: read it with `read_file_content` and add it to the project file per the "Catalogue All Surfaced Docs" rule in Step 4c.

Zero results: mark ✅ — do not skip without running.

If `last_sync_date` is not set in the project file, use `after:[14 days ago date string]`.

**Folder ID scoping:** Use the **current implementation folder ID** from `## GDrive Subfolder` — not the master root. Some merchants (e.g., CarParts) have a master root containing older past-implementation folders that would produce noise. The project file distinguishes these — always use the field labeled "Current implementation folder." If only one folder ID is listed, use that.

**Why 14 days and not 5:** GDrive docs are updated less frequently than email/Slack.
A 5-day window misses docs updated earlier in the week before the sync runs. 14 days
ensures agenda-builder and custom-requirements can skip GDrive traversal entirely when
this sync ran within the past 6 days.

---

### Call Recordings (Chorus, Gemini, Zoom) — 8-day window

Run all three in parallel. These capture call outcomes that won't appear in Gmail or Slack.

**Why 8 days:** The call recording window is wider than the 5-day Gmail/Slack window to ensure calls from the prior week are always captured. This also aligns with the `last_call_date` delta threshold used by downstream skills (agenda-builder) — a sync that ran up to 8 days ago is still considered fresh for call recap purposes.

**Chorus — Primary source: Slack DM `D057X0PEFJS`**
Read this channel once per sync run (covers ALL merchants in one call):
```
slack_read_channel(channel_id: "D057X0PEFJS", oldest: [8-day timestamp], limit: 100)
```
Filter the results by merchant name and aliases. Content is pre-parsed — attendees, action items, and meeting summary are already structured. No `get_thread` calls needed.

After filtering: record the count and build a covered meetings set (date + title) for each merchant. Example: "CarParts — Chorus: 2 recaps found in Slack DM (Jun 5, Jun 10)."

**Gmail Chorus fallback:** Run `subject:[merchant name] "meeting insights ready for review" newer_than:8d` only if the Slack DM returned 0 recaps for this merchant. If Slack DM had results, skip — mark N/A.

**Gemini:**
External calls (merchant-facing) are almost always in Chorus. Internal calls are almost always Gemini. Use this to focus, but apply the deduplication rule regardless.

- **Method A — GDrive (primary):** `parentId = '1qU9GdzkiMDYiXvqHXrVPTUfimulg2zmH' and title contains '[merchant name]' and modifiedTime > '[8 days ago RFC3339]'` — repeat for each alias. If this query returns a tool error, fall back to: `title contains '[merchant name]' and modifiedTime > '[8 days ago RFC3339]'` without parentId, and note the fallback in the output. For each doc found, check its meeting date/title against the Chorus covered meetings set. If already covered by a Chorus recap, skip it. Read the rest via `read_document` or equivalent.
- **Method B — Gmail (fallback):** Run `from:gemini-notes@google.com [merchant name] newer_than:8d` only if Method A returned 0 results for this merchant. For each thread returned, call `get_thread` with `messageFormat: FULL_CONTENT` — the snippet is not sufficient. Apply the same Chorus dedup check before reading.

After running: record "Merchant — Gemini: [N] GDrive docs found, [N] read after dedup / Gmail fallback: N/A or [N] threads opened."
Zero results for an active merchant in a week with known calls is suspicious — verify the query ran correctly before accepting it.

**Zoom (conditional — check Recording Tools field):**
Only run if the project file's `Recording Tools` field lists Zoom. If Zoom is not listed, mark N/A and skip.

When running: `search_zoom` with `entity_type: zoom_doc`, merchant name, filtered to last 8 days. Repeat for each alias. For each result, call `get_file_content` with the file_id. Zero results: mark ✅ "no results found."

**Chorus, Gemini, and Zoom (where applicable) are mandatory for every full active sync merchant.** Running Chorus and finding zero results is fine — not running Chorus at all is not. The Chorus Slack DM (`D057X0PEFJS`) covers all merchants in a single read — there is no per-merchant cost to running it, so it must always run. Gemini Method A must always run; Method B is a fallback only when Method A returns zero, not a replacement. Zoom must always be checked against the Recording Tools field before being marked N/A.

If any of the three sources (Chorus, Gemini, Zoom) return zero results, mark ✅ "no results found" — do not skip without running (except Zoom, which is skipped entirely when not in Recording Tools).

---

## Step 3b — Minimal pulse (Live / On Hold merchants only)

Three sources only. Run in parallel. Fast — should take under 2 minutes per merchant.

**Gmail 120-hour check:**
`from:@[merchant-domain] newer_than:5d`
For any thread returned, call `get_thread` with `messageFormat: FULL_CONTENT`.
Flag any inbound as ⚠️ in the output — unexpected merchant contact on a live account warrants attention.

**Slack — all documented channels (no DMs):**
Read every channel listed in the project file's `## Slack` section that has a documented channel ID — internal channel, external/shared channel, and any group channels. Use `slack_read_channel` with `oldest: [Step 0 timestamp]`, `limit: 50` for each.
For any message with `reply_count > 0`, call `slack_read_thread` immediately.
Do not read individual 1:1 DMs — those belong in the full sweep only.
Flag any notable activity as ⚠️.

**Jira:**
`text ~ "[merchant name]" AND updated >= -5d ORDER BY updated DESC`
Flag any updated tickets.

Do not run Gmail 30-day, Notes doc, GDrive, or call recordings for minimal pulse merchants.

Output format for minimal pulse:
```
### [Merchant] (Live — minimal pulse)
✅ No inbound activity, no Slack activity, no Jira updates
— OR —
⚠️ [N] inbound email(s) from merchant — review needed
⚠️ Notable Slack activity: [brief summary]
⚠️ [N] Jira ticket(s) updated: [ticket IDs]
```

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

## Step 4b — Verification gate (hard counts required before writing ANYTHING)

State the following counts explicitly before touching any project file. If the numbers
do not add up, do not write — go back and fix the gaps first.

**For each merchant processed, state:**

```
[Merchant]:
  Gmail — Phase 1: [N] unique threads collected
  Gmail — Phase 2: [N] get_thread MINIMAL calls made (should equal Phase 1 count)
  Gmail — Phase 2: [N] FULL_CONTENT reads (threads with recent messages)
  Gmail — Phase 2: [N] skipped (threads outside 5d window)
  Slack — [N] channels/DMs read, all using timestamp [Step 0 value]
  Slack — oldest message returned per channel: [date] (confirms correct window)
  Jira — [N] tickets returned
  GDrive — [N] docs modified in last 14d (or "no folder ID — skipped")
  Notes doc — [read: first 15k chars] / [skipped: no URL in project file]
  Chorus — [N] recaps found in Slack DM, [N] action items extracted (or "none found")
  Gemini — [N] GDrive docs found, [N] Gmail threads opened (or "none found")
```

**Hard rules — these are blocking:**
- **Pre-write gate (fires before Step 5 for every merchant):** If Gmail Phase 2 MINIMAL count is less than Phase 1 count, DO NOT proceed to Step 5. Return to Phase 2, open all missing threads with `get_thread MINIMAL`, and only then re-run Step 4b verification. Write-back is blocked until Phase 2 is complete. This is not a warning — it is a hard stop. No amount of context pressure, time pressure, or "low signal" threads exempts a merchant from this gate.
- If any Slack channel returned messages older than 5 days as the NEWEST message: re-run
  that channel with the correct Step 0 timestamp.
- If Chorus returned 1+ emails but total action items logged is 0: re-read the thread body.
  Zero action items on a Chorus email is a parsing failure, not an empty call.
- If an active merchant (In Progress / Hypercare) shows zero results across ALL sources:
  treat as a query error, not a quiet week. Verify and re-run before writing.

**One acceptable exception:** If `get_thread FULL_CONTENT` fails due to the thread being
too large for the context window (tool returns an error or overflows), note it explicitly
as "thread [ID] too large — content saved to file, read via bash" and proceed to extract
content via bash from the saved file path.

---

## Step 4c — GDrive Doc Check (primary GDrive mechanism for merchant-sync)

This step IS the GDrive research for merchant-sync. No folder scan runs — all GDrive coverage flows through the catalogued file IDs in the project file. Every skill run (merchant-sync, agenda-builder, raid-log, refresh-merchant) should be adding to this catalogue so future runs get progressively faster and more precise.

### Delta check on catalogued docs

**If `## Key Reference Docs` is ABSENT from the project file:**
- Log a warning: "`## Key Reference Docs` not bootstrapped for [Merchant] — will be seeded from docs surfaced this run."
- Proceed to the "Catalogue All Surfaced Docs" rule below. Any docs found this run seed the section.

**If `## Key Reference Docs` IS present:**
1. For each entry with a **File ID** (GDrive docs only — skip Lucidchart, folders, external platforms):
   - Call `get_file_metadata` with the File ID.
   - Compare `modifiedTime` against `last_sync_date` from `## Last Sync Metadata`.
2. `modifiedTime` > `last_sync_date` → re-fetch with `read_file_content`. Write a fresh excerpt (2-4 sentences) and a delta note: `Last modified: [date] — Delta: [what specifically changed, e.g., "SOP threshold updated $500→$750; guest claim flow added"]`. Both the project file entry and the sync output summary must show the delta — a bare "doc was updated" is not sufficient.
3. `modifiedTime` ≤ `last_sync_date` → skip. Retain cached excerpt as-is.
4. `File ID: N/A` entries (Lucidchart, folders) → skip unless Samir explicitly reports a change.

If the doc is too large to fully parse: read the HEAD (first 15k chars) plus section headings, note partial read in the delta entry.

### Catalogue All Surfaced Docs (mandatory — runs at end of every sync, before Step 5)

Any doc that surfaces during this sync run — from any source — must have its file ID written to the project file before Step 5 completes. Sources include: Gmail share notifications, Slack links, Gemini notes docs, Chorus recap attachments, Jira ticket attachments, or any URL encountered during research.

**For each surfaced doc:**
1. Extract or look up the file ID (from the URL or `get_file_metadata`)
2. Read the doc with `read_file_content` if not already read
3. Classify it:
   - **Living doc** (notes doc, RAID log, implementation plan, custom requirements sheet) → write to the appropriate dedicated section in the project file (`## Notes Doc`, `## RAID Log`, etc.) with URL and file ID
   - **Reference doc** (SOP, BRD, integration spec, UAT sheet, requirements doc, config doc) → add a new entry to `## Key Reference Docs` with: Name, File ID, URL, Last Modified date, and a 2-4 sentence excerpt describing what it is and what it contains
4. If the doc is already in the project file: confirm the file ID and URL are current; update if stale

Do not defer this to "the next refresh." A doc that surfaces and isn't catalogued will be missed on every future run until it's explicitly added. This is the compounding mechanism — each run builds the catalogue, each future run benefits from it.

**Guiding principle:** Cached excerpts should be dense enough that a downstream skill can understand the document's purpose without fetching it. A future agenda-builder or custom-requirements run should be able to read the excerpt and know whether it needs to re-fetch the full doc.

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

**Overview fields — `Go-live target` and `Status` (surgical update when research confirms a change):**

Update these fields when any research source (Gmail, Slack, Jira, call recaps, notes doc) surfaces
an explicit confirmation. Do not infer, estimate, or pull from Tab 1 of the RAID log — only update
when a source clearly states it.

**`Go-live target`:** Update when a specific date is confirmed by a merchant contact, Extend stakeholder,
or Jira milestone in the 5-day window. Example signals: "we're targeting July 15", "go-live confirmed
for August 1", Jira milestone date updated to a named date. If signals are ambiguous or date is
only described vaguely ("end of Q3", "next month"), leave the field as-is.

**`Status`:** Update when phase transitions are explicitly confirmed. Examples:
- "In Progress" → "Live (Hypercare)" when go-live is confirmed complete in any source
- "In Progress" → "On Hold" when a hold is explicitly stated by either party
- "Live (Hypercare)" → "Live" when hypercare end is confirmed

Only update what is explicitly confirmed in the 5-day sweep. Leave all other Overview fields
(merchant name, products, channel, tech stack) to `refresh-merchant`.

**Future hook — project plan skill:** When a dedicated project plan skill exists and Tab 1 is
reliably maintained by it, add Tab 1 as a trusted source here alongside Gmail/Slack/Jira for
go-live date and phase updates. Until then, live research signals only.

**`## Last researched` field (in Overview):**
- Update to today's date AND include the verification gate counts from Step 4b.

Format:
```
- **Last researched:** [YYYY-MM-DD] (weekly merchant sync — Gmail: [N] threads/[N] full, Slack: [N] channels @ ts:[Step 0 value], Jira: [N] tickets, GDrive: [N] docs, Calls: [N] Chorus/[N] Gemini)
```

Example:
```
- **Last researched:** 2026-06-06 (weekly merchant sync — Gmail: 8 threads/5 full, Slack: 4 channels @ ts:1780262346, Jira: 6 tickets, GDrive: 3 docs, Calls: 1 Chorus/1 Gemini)
```

**`## Last Sync Metadata` section (create if absent, replace if present):**

Write this block as a section in the project file. Downstream skills parse it to decide
whether to run a full sweep or delta. If the block is absent or any field is blank,
downstream skills default to full sweep.

**⛔ CRITICAL — no blanks, no "N/A" — ever:** Every field must have an explicit numeric count. A blank or "N/A" in any field disables delta mode for all downstream skills (agenda-builder, raid-log, custom-requirements) and forces a full research sweep on the next run. Rules:
- Source not run due to tool error → write `0 (tool error)`, not blank
- Source not applicable (e.g., zero Chorus results) → write `0`, not "N/A"
- Source skipped intentionally → do not skip sources; if you did, write `0 (skipped: [reason])`
- `last_call_date` unknown → write `none`, not blank

A correctly populated `## Last Sync Metadata` section is the single most impactful action for reducing downstream skill run time. Missing or malformed fields silently disable delta mode for the entire following week.

```markdown
## Last Sync Metadata
date: [YYYY-MM-DD]
skill: weekly-merchant-sync
gmail: [N] threads / [N] opened / [N] full-content
slack: [N] channels @ ts:[Step 0 value]
jira: [N] tickets
gdrive: [N] docs (14d)
chorus: [N] action items across [N] calls
gemini: [N] docs / [N] threads
notes_doc: read | skipped
raid_log: read | skipped
last_call_date: [YYYY-MM-DD] (date of most recent Chorus/Gemini/Zoom recording found, or "none")
last_raid_date: [YYYY-MM-DD] (read from project file if RAID log has written it; "never" if absent)
```

**Delta activation rule for downstream skills:**
A downstream skill (agenda-builder, raid-log, custom-requirements) may skip a Jira re-query
and use `## Open Ticket Statuses` from the project file when ALL of the following are true:
1. `skill` = `weekly-merchant-sync`
2. `date` is within 8 days of today
3. All fields have explicit counts (no blanks, no "N/A")

`last_call_date` is used separately by agenda-builder to decide whether cached call recaps
are fresh enough to cross-reference against (see agenda-builder Step 3 for details).

If any condition fails: full sweep. The 8-day window covers Friday sync through the
following Saturday, ensuring Monday agendas and mid-week updates can use delta mode
throughout the week.

**`## Recent Call Recaps` section (append-only — prepend new calls at top):**

⛔ **Hard gate:** If Chorus, Gemini, or Zoom returned 1+ calls for this merchant and this section is not written, the sync is incomplete. Do not mark Step 5 done, do not output the sync summary, and do not update `## Last Sync Metadata` until this section is written. This gate is equivalent to the Gmail Phase 2 gate — a call recap that surfaced but wasn't written to the project file will be silently missed by every downstream skill (agenda-builder, raid-log) that relies on the cache.

Do not overwrite this section. Prepend new call blocks at the top — most recent call first.
Before writing, check existing entries by date and title to avoid duplicates. Only add calls
from the current 8-day window that are not already logged.

```markdown
## Recent Call Recaps
_Last updated: [YYYY-MM-DD] (8-day window)_

### [YYYY-MM-DD] — [Call Name] ([Chorus / Gemini / Zoom])
**Attendees:** [list from recap]
**Key decisions:** [verbatim list — do not compress or summarize. Format: "1. [decision]"]
**Action items:** [verbatim numbered list with owner — every item, no truncation. Format: "1. [Owner] — [action]". If Chorus listed 19 items, all 19 appear here.]
**Risks/blockers flagged:** [any risks, blockers, or concerns raised — verbatim. "none" if nothing flagged.]
**Issues surfaced:** [any problems that already occurred and need resolution — verbatim. "none" if nothing surfaced.]
**Open questions:** [verbatim list — every unresolved question. "none" if none.]
**Technical details:** [specific field names, API behaviors, configurations, payload structures, integration specifics — required for RAID log accuracy downstream. "none" if none.]
**Summary:** [2–3 sentence plain-language summary of what was discussed]
```

**⚠️ Fidelity requirement — no compression:** The `## Recent Call Recaps` section is the cache that downstream skills (agenda-builder, RAID log) rely on. If action items or technical details are compressed or omitted here, those downstream skills will miss them when using the cache. Log everything verbatim from the FULL_CONTENT read. A recap with 19 action items should have 19 lines in this section. Summarizing is not acceptable.

If no new calls were found in the 8-day window, prepend a single datestamped line:
```markdown
_[YYYY-MM-DD] — no calls found in 8-day window_
```
Do not overwrite or delete existing entries.

Downstream skills (agenda-builder) read this section to cross-reference against live
Chorus/Gemini queries — catching any content already parsed so the same data isn't
re-processed, and flagging if the live query returns something not yet in the cache.

---

**`## Open Ticket Statuses` section (overwrite each sync):**

Replace this section entirely each run. Lists every active (non-Done) Jira ticket found
in the 8-day sweep. Downstream skills use this to skip a Jira re-query when the sync is
fresh (within 8 days per delta rule above).

```markdown
## Open Ticket Statuses
_Last updated: [YYYY-MM-DD]_

| Ticket | Summary | Status | Assignee |
|--------|---------|--------|----------|
| [ACE-243](link) | [summary] | Ready For Work | Unassigned |
| [FUL-203](link) | [summary] | In Progress | [name] |
```

If no tickets were found, write:
```markdown
## Open Ticket Statuses
_Last updated: [YYYY-MM-DD] — no active tickets found_
```

---

**`## Phase Breakdown` section (overwrite each sync):**

Replace this section entirely each run. Sourced from the project plan tab of the Implementation Plan sheet (identified by tab name, not position — same file for both dedicated and combined structures). Downstream
skills (confluence-sync) read this to understand current phase status without re-querying GDrive.

```markdown
## Phase Breakdown
_Last updated: [YYYY-MM-DD]_

**Phase 1 (Delivered / In Scope):** [items confirmed shipped or in production]
**Phase 2 (Committed / Upcoming):** [items confirmed for a future phase]
**Deferred / Deprioritized:** [items explicitly moved out of scope]
**Go-live target:** [date from Tab 1, or "TBD"]
**Current milestone:** [active milestone from Tab 1]
```

If Tab 1 was not readable, write:
```markdown
## Phase Breakdown
_Last updated: [YYYY-MM-DD] — Tab 1 not readable (tool error)_
```

---

**`## Custom Requirements Summary` section (overwrite each sync):**

Replace this section entirely each run. Sourced from the custom requirements tab of the Implementation Plan sheet (identified by tab name, not position). If no custom requirements tab exists yet, write the "no custom requirements found" placeholder. Downstream
skills (confluence-sync, RAID log delta mode) read this to understand CCERP status without
re-querying Jira.

```markdown
## Custom Requirements Summary
_Last updated: [YYYY-MM-DD]_

| Requirement | Status | Jira Ticket |
|-------------|--------|-------------|
| [name from Tab 3] | [status from Tab 3] | [ticket link] |
```

If Tab 3 was not readable or has no rows, write:
```markdown
## Custom Requirements Summary
_Last updated: [YYYY-MM-DD] — no custom requirements found (or Tab 3 not readable)_
```

---

**`## GDrive Doc Index` section (overwrite each sync):**

Replace this section entirely each run. Lists every doc returned by the 14-day GDrive query.
Downstream skills use this to check whether a doc has changed since last sync without re-querying GDrive.

```markdown
## GDrive Doc Index
_Last updated: [YYYY-MM-DD]_

| Doc Title | File ID | Last Modified |
|-----------|---------|---------------|
| [title] | [file ID] | [YYYY-MM-DD] |
```

If no folder ID in project file or query returned 0 results:
```markdown
## GDrive Doc Index
_Last updated: [YYYY-MM-DD] — no folder ID / no docs found_
```

---

**`## Key Reference Docs` section (update changed entries only — do not overwrite unchanged entries):**

This section is bootstrapped manually. On each sync, only re-fetch and update entries confirmed modified in Step 4c. Do not touch entries that were not modified. Update `Last Modified` and `Excerpt` for any entry you re-fetch. Do not change File IDs, URLs, or Types.

Format per entry (for any entry you update):
```markdown
### [Doc Title]
- **Type:** [type]
- **File ID:** [file_id or "N/A — [platform]"]
- **URL:** [url]
- **Last Modified:** [YYYY-MM-DD]
- **Excerpt:** [2-4 sentence summary — dense enough to understand the doc's content and relevance without fetching it]
```

If no entries changed this run: do not touch this section.

---

**`## Slack Highlights` section (append-only — prepend new entries at top, archive old entries):**

Do not overwrite. Prepend a new dated block only when there is net-new Slack context that is NOT
already captured in `## Open Action Items`, `## Key Decisions`, or `## Recent Call Recaps`.

Gate before logging any highlight: *would a downstream skill (agenda-builder, raid-log, confluence-sync)
benefit from knowing this without re-querying Slack?* If yes, log it. If the context is already
captured elsewhere, skip it — do not duplicate.

**Archive rule (run on every sync):** Before prepending new entries, move any dated block in
`## Slack Highlights` that is older than 60 days to `## Slack Highlights Archive`. The archive
is preserved in the project file for historical reference but is never read by downstream skills.
Create `## Slack Highlights Archive` at the bottom of the project file if it doesn't exist.

```markdown
## Slack Highlights
_Last appended: [YYYY-MM-DD]_

### [YYYY-MM-DD]
- **[#channel or DM with Person]:** [1–2 sentence summary of notable context]
- **[#channel or DM with Person]:** [1–2 sentence summary of notable context]
```

If no net-new highlights this run: do not append a blank entry — leave the section as-is.

---

**`## Gmail Highlights` section (append-only — prepend new entries at top, archive old entries):**

Same structure, gating logic, and archive rule as Slack Highlights. Move entries older than 60 days
to `## Gmail Highlights Archive` at the bottom of the project file. Only log inbound/outbound email
context that is net-new and not already captured in action items, decisions, or call recaps.

```markdown
## Gmail Highlights
_Last appended: [YYYY-MM-DD]_

### [YYYY-MM-DD]
- **[From/To — subject line]:** [1–2 sentence summary of notable context]
```

If no net-new highlights this run: do not append a blank entry — leave the section as-is.

---

**`## Recurring Topics` section (close resolved items only — do not add or restructure):**

Merchant-sync may move a topic from `## Recurring Topics` to `## Already Resolved` when the 5-day
sweep provides explicit resolution evidence: a confirmed decision logged in Key Decisions, an action
item marked complete with a direct source, or a Samir-stated closure. Do not move topics based on
"no new activity" alone — silence is not resolution.

Format for the resolved entry in `## Already Resolved`:
`- [Topic name] — resolved [YYYY-MM-DD] via [source: email/Slack/Jira/Samir]`

Do not add new topics to `## Recurring Topics`, rename existing ones, or reorganize the section.
Net-new recurring themes belong in the next agenda-builder or refresh-merchant run.

---

**`## Sync History` section (append — do not replace):**

Prepend a new dated entry to this section each run. Newest entry goes at the top.
Create the section if it doesn't exist.

```markdown
## Sync History

### [YYYY-MM-DD] (weekly-merchant-sync)
**Resolved:** [item names, or "none"]
**New action items:** [item names, or "none"]
**Key decisions:** [decisions logged, or "none"]
**Context updates:** [brief note on anything material added — call recaps, doc findings, etc., or "none"]
```

This section is the delta record. Downstream skills read the most recent entry to
understand what changed since their last run, without re-querying sources.

**Owned by this skill:**
- **Overwrite each run:** `## Last researched`, `## Last Sync Metadata`, `## Open Ticket Statuses`, `## Jira Epic Hierarchy`, `## Phase Breakdown`, `## Custom Requirements Summary`, `## GDrive Doc Index`
- **Surgical edits (mark complete, add net-new items, update header date):** `## Open Action Items`
- **Surgical update when research confirms a change (not from Tab 1):** Overview `Go-live target`, Overview `Status`
- **Append-only (prepend at top):** `## Key Decisions`, `## Recent Call Recaps`, `## Slack Highlights`, `## Gmail Highlights`, `## Sync History`
- **Archive old entries (>60 days) to archive section, then prepend new:** `## Slack Highlights`, `## Gmail Highlights`
- **Close resolved items to `## Already Resolved` only — no adds or restructuring:** `## Recurring Topics`
- **Update changed entries only:** `## Key Reference Docs`

**Do not touch:** Background/Context, Contacts, Notes Doc URL, Jira Search Config, Gemini/Chorus config sections. Those belong to `refresh-merchant`.

**Owned by agenda-builder (never touch from merchant-sync):** `## Last Agendas` — upsert by meeting title. merchant-sync must not overwrite or clear this section.

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
🔒 Recurring Topics closed ([N]): [topic names, or "none"]
Sources: Gmail [N threads] | Slack [N channels] | Jira [N tickets] | GDrive [N docs] | Notes doc [read/skipped] | Calls [N recaps: Chorus/Gemini/Zoom]

📅 Overview updated: [field] → [new value] (source: [email/Slack/Jira/call recap, date])
  (Omit this line if no Overview fields were updated this run.)

📎 Reference doc bootstrap candidates (GDrive 14d sweep — not yet in ## Key Reference Docs):
  - [doc title] ([file ID]) — looks like [BRD / spec / integration mapping / SOP requirements]
  (Omit this block if no unbootstrapped candidates found.)

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
- **No full research sweep sources** — do not call Salesforce or Google Calendar. Do not
  browse GDrive folder trees. The one exception is `## GDrive Recent Docs (14d)` in Step 3,
  which uses a targeted `modifiedTime` query on the merchant's specific folder — not a
  traversal. If you find yourself browsing folders or reading Salesforce, stop.
- **Surgical writes only** — modify `## Open Action Items`, `## Key Decisions`, `## Last researched`, `## Last Sync Metadata`, `## Recent Call Recaps`, `## Open Ticket Statuses`, `## Jira Epic Hierarchy`, `## Phase Breakdown`, `## Custom Requirements Summary`, `## GDrive Doc Index`, `## Slack Highlights`, `## Gmail Highlights`, `## Slack Highlights Archive`, `## Gmail Highlights Archive`, `## Recurring Topics` (close resolved items only), `## Already Resolved`, `## Sync History`, `## Key Reference Docs`, and Overview `Go-live target` / `Status` (when research confirms a change). Nothing else.
- **Append-only for `## Already Resolved`** — only add; never remove.
- **Trust Samir's direct input** — if he says something is done, mark it done. Don't
  require the sweep to independently verify what he's telling you directly.
- **Accuracy over speed** — thoroughness is the priority. A missed key decision or
  action item has direct downstream impact on Samir's client work. If a run takes
  longer due to proper `get_thread` calls, that is the correct trade-off.
- **Never present partial research as complete** — if any source was skipped or
  produced suspicious results, label the output clearly and explain what was missed.
  A labeled gap is always preferable to a silent one.
