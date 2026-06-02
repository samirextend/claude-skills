---
name: project-pulse
description: >
  Fast project health snapshot across all active merchant implementations. Run on Monday
  mornings, before key meetings, or any time Samir wants to know what needs his attention.
  Reads project files and RAID TSVs only — no live research, no external source calls.
  Use when Samir says "project pulse", "what needs my attention", "weekly briefing",
  "what's open across my merchants", "give me a snapshot", "what should I focus on this
  week", "pulse check", "run the pulse", or "weekly pulse". Also runs automatically on
  the Monday morning schedule.
---

## ⚠️ Read This Skill First — Every Time

Read in full before starting, including on scheduled runs. No live research of any kind.
Data accuracy is bounded by the last skill run that wrote back to each project file —
that is by design.

---

# Project Pulse Skill

Synthesizes a priority brief across all active merchant implementations by reading local
files only: project `.md` files and RAID log TSVs. No Slack, Gmail, GDrive, Jira,
Calendar, or any other external source is called during a pulse run.

**Why local-only:** Project files are the aggregated output of every skill run
(raid-log, agenda-builder, ccerp, custom-requirements, refresh-merchant). They represent
the current known state of each implementation. The pulse trusts that state and synthesizes
from it — it does not attempt to independently verify or supplement it.

**Base directory:** Resolve relative paths from this file's own location.
Project files live at: `{skill_base_dir}/../../../Claude/memory/projects/`

---

## Step 1 — Load and classify all merchants

1. List all `.md` files in the projects directory.
2. Read every project file in parallel.
3. Classify each merchant:

| Class | Criteria |
|---|---|
| **Active** | Status contains "In Progress" OR `## Open Action Items` has at least one non-struck-through bullet |
| **Dormant** | Status = "Live" AND `## Open Action Items` is empty or all entries are struck through |

- "Live (Hypercare)" → Active.
- Any ambiguous status → Active.
- If Samir names a specific merchant, scope to that merchant only.
- Dormant merchants get a single line of acknowledgment — no detail.

### Stale status check — run after initial classification

For every Active merchant, apply this second pass:

> **If** the go-live target date (from the Overview section) is in the past relative to today **AND** `Last researched` is either blank/not yet researched or more than 14 days ago → promote classification to **⚠️ Status Unverified**.

Why this matters: the project file status is only as fresh as the last skill run that wrote to it. A merchant can go live, stall, or cancel between runs — and the file will keep saying "In Progress" until someone runs refresh-merchant. When a go-live target has passed and the file is stale, the stated status cannot be trusted. Flagging it explicitly prevents the pulse from reporting confident-sounding "In Progress" data that may be weeks out of date.

**Parsing the go-live target:** If the `Go-live target` field contains a specific parseable date (e.g., "May 12", "June 23, 2026", "2026-07-20"), compare it to today. If the field only contains vague language without a parseable date (e.g., "TBD", "post Memorial Day", "mid-April"), treat it as non-comparable — do not flag on that basis alone. A passed parseable date + stale research = Status Unverified; vague language + stale research = flag the staleness in "My read" only (existing behavior).

**Status Unverified merchants are processed like Active merchants** — read their RAID TSV, extract priority data, produce the full section — but sort them to the top of the brief and add a visible warning banner (see Step 4).

---

## Step 2 — Load RAID log TSV (local only, no network call)

For each active merchant, check for a local RAID log TSV:

Glob: `{skill_base_dir}/../../../Claude/*{merchant_slug}*raid*log*V*.tsv`

(Replace spaces/hyphens in the merchant name with underscores or wildcards; case-insensitive.)

- If multiple versions exist, use the highest V-number.
- Parse all 15 columns: `ID | Last Updated | RAID | Type | Category | Date Raised | Owner | Status | Name | Description | Next Steps / Notes | Target Completion Date | Decision | Decision Date | Future Phase?`
- If no TSV exists, use project file data only — this is fine.
- **Do not make any network call to fetch a RAID log.** Local file only.

---

## Step 3 — Extract priority data per merchant

Pull from whichever sources are available (RAID TSV and/or project file sections).

### From RAID TSV (when available):

| Category | Definition |
|---|---|
| **Overdue** | Target Completion Date < today AND Status ≠ Complete AND Status ≠ Blocked |
| **Blocked** | Status = "Blocked" (any RAID type) |
| **Due next 14 days** | Target Completion Date between today and today+14, Status ≠ Complete. Sort ascending by date. |
| **Open risks** | RAID = Risk, Status = In Progress or Not Started. Top 5 — sort by most recently updated (Last Updated desc). |
| **Decisions pending** | RAID = Decision, Status = In Progress or Not Started. Show all — these are often the most consequential stalled items. |

### From project file (always — supplement or sole source):

- **Open Action Items section** — all non-struck-through bullets. Tag items not already
  captured in the RAID TSV with `[project file]` so Samir knows the source.
- **Key Decisions section** — entries from the last 60 days that show a pending or
  recently confirmed decision. Surface any that have "still open", "pending", "TBD",
  "not yet confirmed", or similar language.
- **Overview section** — go-live target, current status, any flagged notes (e.g., "⚡",
  "URGENT", phase transitions, date slippage language).
- **Open Tickets section** — any tickets flagged as blocked, escalated, or overdue.
  Do not re-query Jira — read only what's in the project file.

### Deduplication:

If the same item appears in both the RAID TSV and the project file's Open Action Items,
show it once (prefer the RAID TSV version for structured fields). Do not double-count.

---

## Step 4 — Synthesize and output

Output in this format:

```
## Project Pulse — [Today's Date]

**Active:** [merchant names] | **⚠️ Status Unverified:** [names] | **Dormant (skipped):** [names]

---

⚠️ STATUS UNVERIFIED — go-live target passed + file not refreshed
Run /refresh-merchant [name] before acting on their data.
[name — go-live target was [date] — last researched [date or "never"]]

---

### [Merchant Name] ⚠️ STATUS UNVERIFIED
**Go-live target was:** [date] (now past) | **Last researched:** [date or "not yet researched"]
> ⚠️ Go-live target has passed but this file hasn't been refreshed. The data below reflects
> the last known state as of [last researched / initial setup]. Actual status is unknown —
> run /refresh-merchant [name] before acting on anything here.

[... same overdue/blocked/due-next-14/risks/decisions/other-actions sections ...]

**My read:** [Lead with what could have changed since the file was last touched — did they
go live? stall? cancel? Then note the most important known open item. End explicitly with:
"Run /refresh-merchant [name] before acting on this."]

---

### [Merchant Name] — [N flagged items]
**Go-live:** [date or status from project file Overview]
**Last researched:** [date from project file] | **RAID TSV:** [V-number and date, or "none"]

**🔴 Overdue ([N])**
- [ID/#] [Name] — [Owner] — [N days overdue]

**⛔ Blocked ([N])**
- [ID/#] [Name] — [context]

**🟡 Due next 14 days ([N])**
- [ID/#] [Name] — [Owner] — due [MM/DD]

**Open risks ([N])**
- [ID] [Name]

**Decisions pending ([N])**
- [ID or label] [Name]

**Other open actions ([N])** ← project file items without RAID IDs
- [description] — [Owner] [project file]

**My read:** [2–3 sentences. Name the single most important thing to move this week
and exactly what action is needed. If a decision is blocking multiple downstream items,
call that out explicitly. If the merchant is in a quiet phase, say what Extend should
be doing in that window. Never just restate the list above.]

---
[Repeat per merchant. Order by urgency: most overdue/blocked items first.]

---

## ⚡ Top 3 priorities this week

1. **[Item]** — [Merchant] — [Why #1: what breaks if this doesn't move]
2. **[Item]** — [Merchant] — [Why #2]
3. **[Item]** — [Merchant] — [Why #3]

---
*Sources: project files + RAID TSVs only. No live research this run.*
*Data is current as of each merchant's "Last researched" date shown above.*
*If a merchant's project file feels stale, run: /refresh-merchant [name]*
```

**"My read" rules:**
- Never restate what's already listed — synthesize, don't echo
- Lead with the most consequential item and a specific named action
- If a decision is upstream of multiple blocked items, name the dependencies
- If no urgent items exist, say so plainly: "No open items flagged. Project file was last
  researched [date] — consider a refresh if this feels stale."
- Include the last-researched date if it's more than 10 days old — that's context Samir
  needs to calibrate how much to trust the output

---

## Step 5 — Update the pulse artifact

After producing the brief:

1. Load tool schemas via ToolSearch: search for `mcp__cowork__list_artifacts` and
   `mcp__cowork__update_artifact`.
2. Call `list_artifacts` to find the artifact with id `project-pulse-dashboard`.
3. If found, call `update_artifact` with new `widget_code`. **Only replace the
   `pulseData` JavaScript object and `lastUpdated` timestamp** — preserve all HTML
   structure, CSS, and rendering logic exactly as-is.

The `pulseData` object schema:
```javascript
{
  lastUpdated: "MM/DD/YY",
  dormant: ["Merchant Name"],
  merchants: [
    {
      name: "Merchant Name",
      slug: "merchant-slug",
      goLive: "date or status string",
      lastResearched: "MM/DD/YY",
      hasRaidLog: true/false,
      raidVersion: "V3" or null,
      statusUnverified: true/false,   // true when go-live target passed + file stale (>14d or never researched)
      overdue: [ { id, name, owner, daysOverdue, label } ],
      blocked: [ { id, name, context } ],
      dueNext14: [ { id, name, owner, targetDate } ],
      risks: [ { id, name } ],
      decisionsPending: [ { id, name } ],
      otherActions: [ { name, owner } ],   // project file items without RAID IDs
      myRead: "synthesis text"
    }
  ],
  top3: [
    { rank: 1, item: "", id: null, merchant: "", why: "" },
    { rank: 2, item: "", id: null, merchant: "", why: "" },
    { rank: 3, item: "", id: null, merchant: "", why: "" }
  ]
}
```

For `id` field: use the numeric RAID ID when available; use `null` for project-file-only
items (set `name` to include the source tag, e.g. `"Submit Store ID to Shawna [project file]"`).

4. If artifact not found: note it at the end of output — "Pulse dashboard artifact not
   found — reopen it from the sidebar to restore it."

---

## Constraints

- **Local files only** — no Slack, Gmail, GDrive, Jira, Calendar, Salesforce, Chorus,
  Gemini, or Zoom calls during a pulse run. If you find yourself reaching for a live
  source, stop — that belongs in a raid-log or refresh-merchant run instead.
- **Read-only** — do not write to project files, RAID TSVs, or any other file.
- **No removes** — do not modify, delete, or reorganize any existing file content.
- **Speed** — if any file read fails, skip that merchant, note the failure, and continue.
- **Honest staleness signaling** — always show each merchant's "Last researched" date.
  If it's more than 10 days old, flag it explicitly in the "My read" section. Don't
  manufacture confidence from stale data.
- **Dormant = skip** — one-line acknowledgment only. Do not read RAID TSVs or any
  section detail for dormant merchants.
