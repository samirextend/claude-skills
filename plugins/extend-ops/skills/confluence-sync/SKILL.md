---
name: confluence-sync
description: >
  Syncs a merchant's Confluence pages — the Requirements Doc and Implementation Summary —
  using the project MD, Jira, notes doc, RAID log, GDrive technical docs, Salesforce, and
  Slack as sources. Runs a targeted (not full) research sweep, generates a diff of proposed
  changes, confirms before writing. Creates the Implementation Summary child page if it
  doesn't exist. Can be run at any point: mid-implementation to keep docs current, or at
  close-out to produce the final summary.

  Use when Samir says "sync [merchant] to Confluence", "update the requirements doc for
  [merchant]", "create an implementation summary for [merchant]", "confluence sync",
  "update Confluence for [merchant]", or any variation of keeping Confluence in sync with
  current implementation state.
---

# Confluence Sync

Keeps two Confluence docs current for a named merchant:

1. **Requirements Doc** — the existing merchant requirements page. Targeted updates to
   dynamic sections only. Static sections (co-marketing assets, legal, accounting,
   store ops) are never touched.
2. **Implementation Summary** — the completed reference artifact based on the
   `[TEMPLATE] Implementation Summary` template. Created as a child page of the
   requirements doc on first run; updated on all subsequent runs.

**This is not a full research sweep.** No Calendar, no GDrive folder traversal, no
full Gmail sweep, no Chorus/Gemini/Zoom. Those belong in `refresh-merchant`. This skill
runs targeted source reads mapped to specific Confluence fields.

**Base directory:** Resolve relative paths from this file's own location.
Project files: `{skill_base_dir}/../../../Claude/memory/projects/{merchant}.md`
Research guide: `{skill_base_dir}/../../../Claude/memory/research-guide.md`

---

## Research Scope

Read `research-guide.md` before starting — follow its source access patterns, query
syntax, and alias rules exactly. But only run the sections listed below. Do not expand
to the full sweep.

**⛔ HARD RULE: Every source marked REQUIRED below must run. Do not skip any source because it seems low-value or unlikely to have changed. The only valid reason to mark a source as not-run is a hard tool error. You will fill out a source checklist in Step 5 before writing anything — if a required source is blank, go back and run it.**

**Run — ALL REQUIRED unless noted:**

- **Section 0 — Project file** (REQUIRED, always first)
- **Section 0b — Implementation Plan and RAID Log data** (REQUIRED — primary source for decisions, risks, and custom requirements; must be fetched before any other research). Check `## RAID Log` in the project file for `**Structure:** Dedicated` before reading:
  - **Dedicated structure (WP, CarParts, Sleep Number):** Read the Implementation Plan sheet (file ID from `## GDrive Subfolder`). Validate tab names after reading — route by actual tab name, not position: project plan tab → phase/milestone data; custom requirements tab → CCERP data. Read RAID rows from the dedicated RAID sheet (file ID from `## RAID Log`).
  - **Combined structure (Peloton, others):** Read the single sheet (file ID from `## RAID Log`). Validate tab names after reading — route by actual tab name: project plan tab → phase/milestone data; RAID log tab → decisions/issues; custom requirements tab → CCERP data. Skip any tab that does not exist yet. Do not assume a fixed tab order.
- **Section 0b — Notes doc** (REQUIRED — targeted read of last 4 weeks for recent decisions and edge cases; do not filter the RAID log decisions by date)
- **Section 1 — Salesforce** (REQUIRED — go-live date, products, pilot conditions, commercial context)
- **Section 3 — GDrive targeted doc reads** (REQUIRED — read every doc explicitly linked in the project MD or requirements doc; do not browse folders):
  technical proposal, architecture doc, program setup spreadsheet, SOP PRD, claims process doc
- **Section 4 — Slack targeted reads** (REQUIRED — 3 weeks back; primary channel + DMs with SA and GSM; search merchant name + aliases)
- **Section 6 — Jira** (REQUIRED — CCERP epic + all child tickets + TAM board)

**Do NOT run:**
- Calendar (Section 2)
- Full GDrive folder traversal
- Full Gmail sweep (Section 5)
- Chorus, Gemini, Zoom
- Go-Live Checklist Assessment (Section 8) — unless explicitly running a go-live check
- Pre-Output Verification subagent (Section 8b)

---

## Field Source Map

Maps each Implementation Summary section to its primary sources. Use this to know
where to look for each field — do not search every source for every field.

| Section | Primary Sources |
|---|---|
| Quick Reference | Project MD, requirements doc, Salesforce opp |
| Merchant Tech Stack | Notes doc, technical proposal doc, requirements doc |
| Offers | Notes doc, technical proposal, requirements doc |
| Order and Contract Flow | Technical proposal, architecture doc, notes doc, requirements doc |
| API and Integrations | Jira TAM tickets, technical proposal, notes doc |
| Claims Architecture | Claims process doc, notes doc, requirements doc |
| What Is NOT Integrated | RAID log (deferred/deprioritized), requirements doc Phase table |
| Known Edge Cases | Notes doc, requirements doc, Slack |
| Custom Engineering (CCERP) | Jira CCERP epic + all child tickets |
| Known Issues at Go-Live | RAID log open issues, Jira open tickets, notes doc |
| Key Decisions Log | RAID log decisions tab, notes doc, Slack |
| Consumer Experience | Requirements doc (CRM table), notes doc |
| Co-Marketing | Requirements doc, notes doc, project MD |
| Reporting | Requirements doc, notes doc, project MD |
| Legal and Licensing | Requirements doc, Salesforce opp |
| Business KPIs / Pilot | Notes doc, Salesforce opp, project MD |
| SOP Program | SOP PRD doc (linked in project MD) |
| Billing Edge Cases | Notes doc, requirements doc, RAID log |
| Project Team Contacts | Project MD contacts section (both Extend and Merchant tables) |
| Escalation Contacts | Project MD contacts section, notes doc |
| Ongoing Operations | Project MD, notes doc, Jira TAM |
| Phase Status | Project MD, requirements doc, Implementation Plan tab 1 |
| Reference Links | Project MD (all linked docs and Jira boards) |

---

## Requirements Doc — Updatable Sections

Only these sections are ever written to in the requirements doc. Everything else
is left exactly as-is.

**Update automatically (no diff required):**
- Project Details: go-live date and store IDs if blank or outdated
- Key Onboarding Assets: RAID log link, Jira links if blank
- Phases: delivered vs. committed vs. deprioritized items

**Show diff, confirm before writing:**
- Project Team: Extend and Merchant contact tables — add/update names, emails, and
  workstream notes sourced from project MD contacts section
- Technical Design and Requirements: integration type, contract trigger, all orders,
  offers, contract creation, order structure — only if content has changed or was blank
- Claims: intake ownership, adjudication method, servicing — only if changed or blank
- Reporting Requirements: order data coverage, attach rate

**Never touch:**
- Pre-Sales Discovery Assets
- Co-Marketing (all subsections)
- Accounting and Finance
- Legal and Regulatory Compliance
- Enablement and Store Operations
- UAT and Deployment

---

## Step 1 — Identify merchant and load project file

Determine the merchant name from Samir's message.

Read the project file:
`{skill_base_dir}/../../../Claude/memory/projects/{merchant}.md`

Extract and hold:
- All contact names and Slack user IDs (Extend side: SA, GSM, TAM)
- Aliases from "Also search as"
- Notes doc URL
- Salesforce opp URL
- CCERP epic ID or link
- TAM Jira board or epic ID
- Implementation Plan sheet file ID and tab GIDs
- `confluence_requirements_doc_id` — the Confluence page ID of the requirements doc
- `confluence_implementation_summary_id` — the Confluence page ID of the implementation
  summary (may be blank on first run)
- Any GDrive doc links: technical proposal, architecture doc, program setup spreadsheet,
  SOP PRD, claims process doc

If `confluence_requirements_doc_id` is blank: stop and ask Samir for the requirements
doc Confluence page URL before proceeding. This field is required.

---

## Step 2 — Read current Confluence state

Read both docs in parallel using `getConfluencePage` with `contentFormat: markdown`.

**Requirements doc:** use `confluence_requirements_doc_id` from project MD.

**Implementation Summary:**
- If `confluence_implementation_summary_id` is in the project MD: read it directly.
- If blank: call `getConfluencePageDescendants` on the requirements doc page ID to check
  if a child page titled "[Merchant] Implementation Summary" already exists.
  - If found: note the page ID and read it.
  - If not found: note that it will be created in Step 6.

Parse and hold the current content of both docs in memory. This is the baseline —
everything found in research is compared against this before any write.

---

## Step 3 — Run targeted research

Run all sources in parallel where possible. Follow research-guide.md for access
patterns, query syntax, and alias rules.

### 3a. Project file and key artifacts (Section 0 and 0b)

Already loaded in Step 1. Now read the linked artifacts:

**Notes doc:** Read the last 4 weeks of content. Focus on: integration decisions,
technical edge cases, billing arrangements, escalation contacts, scope changes.
Discard content older than 4 weeks.

**Implementation Plan sheet:** Check the project file's `## RAID Log` section for `**Structure:** Dedicated` before reading:

**If dedicated (WP, CarParts, Sleep Number):**
- Read **RAID log data** from the **dedicated RAID sheet** (file ID from `## RAID Log` section).
- Read the **Implementation Plan sheet** (file ID from `## GDrive Subfolder`). After reading, validate tab names — route by actual tab name, not position: project plan tab → phase/milestone data; custom requirements tab → CCERP status.

**If combined (Peloton, others):**
- Read the single sheet (file ID from `## RAID Log`). After reading, validate tab names — route by actual tab name: project plan tab → phase/milestone data; RAID log tab → decisions/issues; custom requirements tab → CCERP status. Skip any tab that does not exist yet.

Do not assume a fixed tab order for any merchant.

### 3b. Salesforce (Section 1)

Use the Salesforce opp URL from the project file. Call `read_document` on the URL.
If that fails, use Glean `search` with `app: salesforce` and merchant name.

Extract: go-live date or target, products in scope, store count, pilot conditions,
commercial context, AE and GSM names.

### 3c. GDrive — targeted doc reads (Section 3, targeted only)

For each URL found in the project file or requirements doc that maps to a high-signal
technical doc, check `## Key Reference Docs` first before calling `read_document`:

- If a matching entry exists (by File ID or URL) and its `Last Modified` date is ≤ `last_sync_date` from `## Last Sync Metadata`: use the cached excerpt for the Confluence diff — do not fetch the full doc.
- Only call `read_document` if: no cached entry exists, `Last Modified` > `last_sync_date`, or the excerpt is clearly insufficient for the specific field being populated (e.g., exact field values or table data required).
- If `## Last Sync Metadata` is absent from the project file, treat all entries as stale and fetch normally.

Docs to check and their downstream targets:

- Technical proposal / architecture doc → feeds Order/Contract Flow, API sections
- Program setup spreadsheet → feeds Offers, Products/Eligibility, Pricing sections
- SOP PRD → feeds SOP Program section
- Claims process doc → feeds Claims Architecture section

Do not browse folders. If a URL is not explicitly linked somewhere, skip it.

### 3d. Slack — targeted reads (Section 4)

Look back 3 weeks. Run in parallel:
1. Read primary implementation channel from project file
2. Read DMs with SA and GSM (use `slack_search_users` if user IDs unknown)
3. Search merchant name + each alias across all channels

For every message with thread replies, read the full thread.
Goal: technical decisions, edge cases, known issues, scope agreements.

### 3e. Jira (Section 6)

Run in parallel:
1. CCERP epic: fetch the epic, then list all child tickets with status and summary
2. TAM board: `project = TAM AND text ~ "[merchant name]" AND statusCategory != Done ORDER BY updated DESC`
3. Any other Jira project prefixes in the project file's "Relevant projects" field

For each ticket, fetch: `["summary", "status", "assignee", "comment"]`

---

## Step 4 — Build the diff

For each field in both the requirements doc and implementation summary, compare:
- **Current value** (from Step 2 — what's in Confluence now)
- **Proposed value** (from Step 3 — what sources show)

Classify each field as:

- **No change** — current and proposed match, or sources have no data. Skip entirely.
- **New data** — field is currently blank and sources have data. Include in diff.
- **Update** — field has an existing value that differs from what sources show. Include
  in diff with both current and proposed values clearly shown.
- **Gap** — field is blank and no source has data for it. Flag as `[UNKNOWN — needs review]`
  but do not include in the write-back. List all gaps in the output summary.

Build two diff tables — one for the requirements doc, one for the implementation summary:

```
### Requirements Doc — Proposed Changes
| Section | Field | Current Value | Proposed Value | Source |
|---|---|---|---|---|
| Technical Design | Contract Trigger | [blank] | Fulfillment/POD | Notes doc, 2026-04-15 |
| Phases | Phase 2 | [blank] | Claims integration (in progress) | RAID log |

### Implementation Summary — Proposed Changes
| Section | Field | Current Value | Proposed Value | Source |
|---|---|---|---|---|
| Quick Reference | Go-Live Date | [blank] | June 2024 | Salesforce opp |
| Custom Engineering | Claims webhook | [blank] | LIVE — POST-3807 | Jira |
```

For the implementation summary on first run, most fields will be "New data" — that's
expected. Show the full proposed content section by section rather than a per-field
table, since it's a new document.

---

## Step 5 — Source checklist + confirm before writing

**⛔ Do not present the diff until you have completed this checklist. Every REQUIRED source must be confirmed run or marked with a hard tool error.**

```
### Source Checklist — [Merchant] Confluence Sync
- Project file: ✅ read
- Implementation Plan Tab 1 (Project Plan): ✅ read / ❌ tool error
- RAID Log (dedicated sheet or Tab 2 of combined sheet): ✅ read / ❌ tool error
- Custom Requirements (Tab 3 or dedicated tab): ✅ read / ❌ tool error
- Notes doc (last 4 weeks): ✅ read / ❌ tool error
- Salesforce opp: ✅ read / ❌ tool error
- GDrive docs: ✅ [list doc names read] / ⏭ none linked / ❌ tool error
- Slack (primary channel + DMs): ✅ read / ❌ tool error
- Jira (CCERP + TAM): ✅ [N tickets] / ❌ tool error
```

Then present the diff to Samir with:
- N changes to the requirements doc
- N fields to populate in the implementation summary (or "creating new page")
- N gaps flagged — including any requirements doc fields that contradict RAID log decisions (stale data)
- The source checklist above

Ask: "Ready to write these changes?"

Do not write anything until Samir confirms. If he wants to adjust any proposed value,
take the correction and update the diff before writing.

**Exception — create without confirming if:**
- The implementation summary page doesn't exist yet AND all fields being populated
  are marked "New data" (not overwriting existing content). In this case, note that
  you're creating the page and proceed.

---

## Step 6 — Write to Confluence

### Requirements doc updates

Use `updateConfluencePage` with the full page body. Reconstruct the HTML by:
1. Taking the current page content from Step 2
2. Applying only the confirmed changes from the diff
3. Leaving all other sections byte-for-byte identical

Only update sections in the "Updatable Sections" list above.

### Implementation summary

**If creating new:** Use `createConfluencePage` with:
- `parentId`: `confluence_requirements_doc_id`
- `title`: "[Merchant Name] Implementation Summary"
- `spaceId`: `164102149` (Solutions space)
- Full content built from the diff using the template structure

**If updating existing:** Use `updateConfluencePage` with the page ID.
The implementation summary is fully machine-managed — it can be more aggressively
updated than the requirements doc. Replace entire sections where new data exists.
Leave sections blank (with placeholder text) where no data was found.

In both cases, use the `[TEMPLATE] Implementation Summary` structure
(Confluence page ID: 3654090759) as the reference for section order and formatting.

---

## Step 7 — Update project MD with Confluence page IDs

After writing, make two surgical updates to the project file:

1. Add or update `confluence_requirements_doc_id` if it was missing or changed
2. Add or update `confluence_implementation_summary_id` with the page ID of the
   implementation summary (especially important after a first-run creation)

These IDs are how the skill finds its target pages on every subsequent run.
If either is missing from the project file, the skill cannot run without prompting
Samir — so always write them back.

---

## Step 8 — Output summary

```
## Confluence Sync — [Merchant] — [Today's Date]

### What was written

**Requirements Doc** ([N] changes):
- [Section]: [what changed]
- [Section]: [what changed]

**Implementation Summary** ([created / updated]):
- [Section]: [what was populated]
- [Section]: [what was populated]

### Gaps flagged ([N] fields need manual input)
- [Section > Field]: no data found in any source
- [Section > Field]: conflicting data — [explain conflict]

### Sources used
✅ Project MD
✅ Notes doc — [date range read]
✅ Implementation Plan sheet — [tabs read]
✅ Salesforce opp
✅ GDrive: [list of docs read by name]
✅ Slack: [channel name] + DMs with [names]
✅ Jira: CCERP ([N] tickets) + TAM ([N] tickets)
❌ [Source] — [hard tool error if applicable]

---
Project MD updated with Confluence page IDs.
Run again anytime to pull in new changes.
```

---

## Constraints

- **Targeted research only** — do not run Calendar, full Gmail sweep, full GDrive traversal,
  Chorus, Gemini, or Zoom. If you find yourself reaching for those, stop. They belong in
  `refresh-merchant`.
- **Delta updates only** — never rewrite a section that hasn't changed. The goal is precision,
  not thoroughness. A section with accurate existing content should come out of this skill
  identical to how it went in.
- **Never touch static requirements doc sections** — Co-Marketing, Accounting and Finance,
  Legal and Regulatory, Enablement and Store Operations, Pre-Sales Discovery Assets.
  These are manually maintained and are not sync targets.
- **Confirm before writing** — always show the diff and wait for confirmation before calling
  any write tool, except when creating a brand new implementation summary page.
- **Store Confluence IDs in project MD** — always write back `confluence_requirements_doc_id`
  and `confluence_implementation_summary_id` after every run. These are how the skill finds
  its targets.
- **Flag, don't guess** — if a field has no data in any source, mark it
  `[UNKNOWN — needs review]` and surface it in gaps. Do not infer values.
- **4-week filter** — discard content older than 4 weeks from notes doc and Slack reads,
  per research-guide.md Section 0b.
- **Implementation Summary template reference** — always follow the structure of
  Confluence page ID 3654090759 (`[TEMPLATE] Implementation Summary`) for section order
  and formatting. Do not invent new sections.
