---
name: raid-log
description: >
  Capture and update RAID log items (Risks, Actions, Issues, Decisions) for merchant
  implementation projects. Use whenever Samir says "update the RAID log for [merchant]",
  "log a RAID item for [merchant]", "add to [merchant]'s RAID log", "capture a
  risk/action/issue/decision for [merchant]", or any variation of wanting to document
  implementation tracking items. Also triggers when reviewing notes or meeting outputs
  and Samir says "what should go in the RAID log", "log anything important from
  [meeting/call]", or "prep the RAID log before my [merchant] call".
---

## ⚠️ Read This Skill First — Every Time

Read this skill in full before starting any work — including on session resume or mid-task continuation. Do not rely on memory from prior runs.

**No-skip rule for research:** Every section of the research guide is required on every run. "I already had enough context" is never a valid reason to skip a source — the only valid ❌ in Research Sources is a hard tool error with zero results.

---

# RAID Log Skill

This skill surfaces and logs RAID items (Risks, Actions, Issues, Decisions) for a merchant's implementation project by doing deep research across all available sources, then presenting proposed entries in a structured, ready-to-paste format for the merchant's Implementation Plan & RAID Log spreadsheet.

## Workflow

### Step 1: Locate the Merchant's RAID Log

1. Read the merchant's project file. Resolve the path using the skill's own location: `{skill_base_dir}/../../../outputs/memory/projects/{merchant}.md` (where skill_base_dir is the directory containing this SKILL.md, e.g. `/sessions/abc123/mnt/.claude/skills/raid-log`). This resolves to `/sessions/abc123/mnt/outputs/memory/projects/{merchant}.md`.
2. Check for a **dedicated RAID log sheet** first: look for a `## RAID Log` section in the project file with a `**Google Sheet:**` field and `**Structure:** Dedicated` note. If present, use this sheet — it is a standalone RAID-only sheet (single tab). Skip to step 4.
3. If no dedicated sheet: look for the **Implementation Plan & RAID Log** combined sheet in the project file's GDrive section. If not in the project file, locate it in Google Drive:
   - **Root merchants folder**: `https://drive.google.com/drive/folders/16w2ilKVgPtXgwKEmYeHE6-viNsGrBZTG`
   - Navigate into the appropriate **status subfolder** (In Progress, Live, On Hold, or similar), then into the merchant-specific subfolder, then into the onboarding subfolder
   - The file title contains the merchant name + "RAID" + "Plan" — e.g., "[Merchant] Extend Implementation Plan & RAID Log"
   - Once found, save the file ID into the merchant's project file for future reference
4. Read the spreadsheet. For dedicated sheets (Structure A — WP, CarParts, Sleep Number): all rows are the RAID log — read with one `read_file_content` call. For combined sheets (Structure B — Peloton): the RAID log is Tab 2. Column B is `Last Updated`. Full column order: `ID | Last Updated | RAID | Type | Category | Date Raised | Owner | Status | Name | Description | Next Steps / Notes | Target Completion Date | Decision | Decision Date | Future Phase?`
5. Capture **all existing rows in full**, including each row's current `Last Updated` value and the exact current Name value (preserving any [NEW]/[UPDATED] prefix already in the sheet from a prior run — you will strip these for untouched rows when producing output)
6. Note the **next available ID number** to continue the sequence for new items

### Delta Mode Gate (check before Step 2)

**Before running the full research sweep, check whether delta mode applies.**

Delta mode is a faster alternative to the 8-week full sweep. It uses cached project MD sections as the baseline for context up to last sync, then sweeps only the gap between last sync and today. It is appropriate for routine mid-week or weekly updates where the project MD was freshly synced.

**Gate — all three must pass. If any fails, skip to the full sweep in Step 2.**

1. `## Last Sync Metadata` in the project MD has `date` within **8 days** of today AND `skill: weekly-merchant-sync`
2. `## Recent Call Recaps` section exists in the project MD (even if it reads "no calls found in 8-day window" — a quiet week is valid; the section just needs to have been written by a prior sync)
3. RAID TSV was successfully read in Step 1 (not a hard tool error)

**If gate passes:** Log `DELTA MODE activated — last sync [date], gap: [N] days` and follow the Delta Research Scope below. Skip the full sweep in Step 2.

**If gate fails:** Log which condition failed, then proceed to the full sweep in Step 2 as normal.

---

#### Delta Research Scope (only when gate passes — replaces the full Step 2 sweep)

**Compute timestamps first (bash — no mental math):**
```bash
# Replace [last_sync_date] with the date value from ## Last Sync Metadata
date -d '[last_sync_date]' +%s                    # gap_ts — oldest= param for Slack reads since last sync
date -d '7 days ago 00:00:00' +%s                 # seven_day_ts — for Gmail/Jira queries
date -d '7 days ago 00:00:00' --utc +%Y-%m-%dT%H:%M:%SZ  # RFC3339 for GDrive/Gemini queries
```

Note: `gap_ts` from a date string (e.g., `date -d '2026-06-10'`) defaults to midnight automatically — it is already midnight-anchored. The `7 days ago` timestamps require the explicit `00:00:00` because without it they compute relative to the current time of day, which can exclude messages posted early on the boundary day.

**Read (required):**

- **Project MD — all sections:** `## Recent Call Recaps`, `## Open Action Items`, `## Open Ticket Statuses`, `## Phase Breakdown`, `## Custom Requirements Summary`, `## Key Decisions`, `## Sync History`. These are the primary context source for everything up to last_sync_date. Use them verbatim — do not re-query for items already captured here.
  - **`## Recent Call Recaps` — date-scope when processing:** This is a running log (newest first). Only process entries newer than `last_sync_date` for new RAID items — older entries were already processed in prior runs and should not generate duplicate proposals.
- **Project MD — `## GDrive Doc Index`:** Read before any GDrive query. Skip any doc whose `Last Modified` date in the index predates `last_sync_date`. Only fetch docs with a newer modified date. If absent, proceed with normal GDrive query.
- **Project MD — `## Key Reference Docs`:** Check before any GDrive doc read. For each entry with a File ID, compare `Last Modified` against `last_sync_date`. Skip docs not modified since last sync — use cached excerpts as context. Only fetch docs with `Last Modified` > `last_sync_date`. Entries with `File ID: N/A` (Lucidchart, folders, external platforms) cannot be skipped this way.
- **Project MD — `## Slack Highlights`:** Read as ambient Slack context from the last sync window. Use as supplement for non-critical context; don't re-query Slack for items already captured here.
- **Project MD — `## Gmail Highlights`:** Same as Slack Highlights, for email context.
- **Project MD — `## Last Agendas`:** Read if present. Apply a two-level read: (1) find entries matching the most recently run meeting type (Internal or External) as primary context — surfaces what was known and planned going into the last call, including action items and decision context not yet logged in RAID; (2) scan all other entries of the same type as supplemental background. Do not skip entries because their title differs slightly — the same-type scan catches context from similarly-named meetings.
- **RAID TSV — all 3 tabs:** Same as full mode. Required for current state of all rows.
- **Notes doc — last 7 days only:** Fetch the last portion of the doc and discard entries older than 7 days.
- **Slack primary channel:** `oldest: gap_ts` — only messages since last sync
- **Slack DMs with key contacts:** `oldest: gap_ts` — each contact in the project file's Extend-side list
- **Slack DM `D057X0PEFJS` (Chorus):** `oldest: gap_ts` — captures any calls since last sync that aren't yet in `## Recent Call Recaps`. Extract every action item and summary in full — same fidelity rules as full mode.
- **Gemini GDrive folder (Method A):** docs modified since `last_sync_date` (use RFC3339 timestamp). For each doc found, check against the Chorus covered meetings set and read uncovered docs in full.
- **Jira:** Two sources — both required:
  - Live query: `text ~ "[merchant name]" AND updated >= -7d ORDER BY updated DESC` — open tickets updated in the gap window
  - Project file `## Jira Epic Hierarchy` — read directly, no live query needed. Catches child tasks and cross-project linked issues not recently updated and therefore invisible to the date-gated query above
- **Gmail inbound:** `from:@[merchant-domain] newer_than:7d` — inbound only. For each thread, call `get_thread MINIMAL`, then `FULL_CONTENT` if last message is within 7 days.

**Gmail Chorus fallback:** Run only if Slack DM `D057X0PEFJS` returned 0 recaps for this merchant since `last_sync_date`.
**Gemini Gmail Method B fallback:** Run only if GDrive Method A returned 0 results since `last_sync_date`. If it runs, call `get_thread FULL_CONTENT` on every result.

**Skip (not needed in delta mode):**
- 8-week Slack lookback — replaced by gap-window read above
- Gmail 30-day outbound / keyword sweep
- Salesforce
- Calendar
- GDrive folder traversal

**Pre-output checklist for delta mode (fill before Step 5):**

```
DELTA MODE — [Merchant] — last sync [date] / gap: [N] days

- Project MD (all sections): ✅ / ❌
- Project MD — GDrive Doc Index: ✅ read, [N] docs in index / ✅ absent / ❌ error
- Project MD — Slack Highlights: ✅ read / ✅ absent / ❌ error
- Project MD — Gmail Highlights: ✅ read / ✅ absent / ❌ error
- Project MD — Last Agendas: ✅ read ([N] entries, same-type scan done) / ✅ absent / ❌ error
- RAID TSV (all 3 tabs): ✅ / ❌ tool error
- Notes doc (last 7d): ✅ read / ❌ tool error / ⏭ no URL in project file
- Slack primary channel (since [last_sync_date]): ✅ N messages / ✅ 0 / ❌ tool error
- Slack DMs (contacts: [list], since [last_sync_date]): ✅ / ❌ tool error
- Jira (updated -7d): ✅ N tickets / ❌ tool error
- Gmail inbound (7d): ✅ N threads / N opened FULL_CONTENT / ❌ tool error
- Slack DM D057X0PEFJS — Chorus since [last_sync_date]: ✅ N recaps / ✅ 0 / ❌ tool error
- Gemini GDrive Method A (since [last_sync_date]): ✅ N docs / ✅ 0 / ❌ tool error
- Gmail Chorus fallback: N/A (Slack DM had results) / ✅ N results / ❌ tool error
- Gmail Gemini Method B fallback: N/A (Method A had results) / ✅ N threads, N FULL_CONTENT / ❌ tool error
```

Label the Part 1 output header as `DELTA SWEEP — [N]-day gap since last sync ([date])`.

---

### Step 2: Deep Research (full sweep — only when delta gate fails)

Read `research-guide.md` at `{skill_base_dir}/../../../Claude/memory/research-guide.md` and follow **Sections 0b through 7** for all research mechanics: key merchant artifacts (notes doc, Implementation Plan, RAID log, custom requirements sheet), Salesforce, Calendar, GDrive, Slack, Gmail (including Chorus and Gemini dual-method), Zoom, Jira, alias discipline, dynamic contacts list, completed-action checks, and cross-source reconciliation.

**Before producing any output, execute Section 8b (Pre-Output Verification) from the research guide — spawn the verification subagent as described there and address any gaps before proceeding to output. This is mandatory and not optional.**

**RAID-specific overrides:**

- **Extended lookback:** Use **8 weeks** for Slack and GDrive instead of the guide's 3-week and 4-week defaults. RAID items frequently surface from decisions and conversations made weeks ago that haven't been formally documented yet. Gmail 30-day default applies as-is.
- **Research focus:** Actively hunt for anything that deserves a RAID entry — risks, blockers, decisions, and committed actions. Don't wait for something to be explicitly labeled; read between the lines in every source.
- **External merchant channel is highest priority:** Informal decisions, security questions, and integration asks from the merchant side surface here first and often nowhere else.
- **Chorus/Gemini — always read in full, never use the cache:** The agenda-builder delta shortcut does NOT apply here. RAID log entries require the full technical context — specific field names, API behaviors, integration details, verbatim decisions — that a cached summary may compress or drop. For Chorus: read Slack DM `D057X0PEFJS` and extract every relevant recap in full (all action items + full meeting summary — do not truncate). For Gemini: read every uncovered GDrive doc in full via `read_document`; if Gmail Method B runs as fallback, call `get_thread FULL_CONTENT` on every result. Do not rely on project file caches regardless of `last_call_date`.

**What constitutes each RAID type:**

| Type | Definition |
|------|------------|
| **Risk** | Something that *could* derail the timeline, scope, or quality — but hasn't happened yet. Forward-looking. |
| **Action** | A committed next step that a specific person or team needs to complete. |
| **Issue** | A problem that has already occurred and needs resolution. Past or present tense. |
| **Decision** | A choice that was made (or explicitly needs to be made) that affects the project direction. |

When in doubt between Risk and Issue, ask: has it happened yet? If not, it's a Risk.

**Judgment filter — what belongs in the RAID log:**

The RAID log is a curated view of what materially matters to the project, not a comprehensive task list. Before logging any item, ask: *if this were missed or left unresolved, would it meaningfully impact the project timeline, go-live, integration quality, or merchant relationship?* If yes, log it. If not, leave it out.

Items that belong:
- Risks that could delay go-live or derail a workstream
- Decisions that shape project direction, scope, or architecture
- Issues actively blocking progress or requiring cross-team resolution
- Actions with real dependencies, deadlines, or stakeholder accountability

Items that don't belong:
- Simple follow-ups that will close within a day or two without consequence
- Administrative minutia (scheduling a meeting, sending a document)
- Items already handled informally with no ongoing risk
- Anything that would resolve itself without meaningful impact if no one tracked it

Err on the side of including rather than excluding — but apply this filter first. A RAID log with 15 meaningful items is far more useful than one with 50 that bury the critical ones.

### Step 3: Identify Updates to Existing Items

Before proposing new items, review every existing RAID entry against your research findings to see if any fields should be updated. Use evidence from Slack, email, Jira, or meeting notes — never assume a change without a source.

**Fields that may be updated on existing rows:**

| Field | When to update |
|-------|---------------|
| **Status** | Evidence shows the item has moved — e.g., Jira ticket closed → Complete, Slack message flagging a blocker → Blocked, work has begun → In Progress |
| **Owner** | A specific person's name can now be identified where it was previously just "Extend" or "Merchant" |
| **Next Steps / Notes** | New information exists. **Always append — never overwrite.** Prepend the new note with a date prefix above existing notes: `MM/DD/YY: [new note] \| [existing notes preserved below]` |
| **Decision** | A decision was reached for a Decision-type item that previously had this field blank |
| **Decision Date** | Accompanies a Decision field update |
| **Target Completion Date** | A specific date was committed to in a meeting, email, or Jira |
| **Future Phase?** | It was explicitly confirmed that the item is deferred |

**Fields that must never be changed on existing rows:**

- **ID** — immutable
- **RAID** — the item type doesn't change retroactively
- **Type** (Internal/External) — only change if Samir explicitly instructs it
- **Category** — only change if Samir explicitly instructs it
- **Date Raised** — always preserves the original capture date
- **Name** — only change if Samir explicitly instructs it
- **Description** — treat as immutable unless Samir asks; surface new context in Next Steps instead

For each existing row you're updating, note what changed and why (what source supported the change) — this goes in the summary.

### Step 4: Populate All Fields for New Items

For each proposed item, fill in every applicable field:

| Field | Guidance |
|-------|----------|
| **ID** | Next sequential number continuing from the last existing entry |
| **Last Updated** | Today's date in MM/DD/YY format |
| **RAID** | Risk / Action / Issue / Decision |
| **Type** | `Internal` = Extend-only context, not for merchant eyes (e.g., internal resourcing, margin concerns, process gaps). `External` = Relevant to surface with the merchant on a weekly sync. |
| **Category** | Onboarding / Technical / Marketing / Programs / Legal/Compliance / Claims / Analytics / Training/Ops / Accounting |
| **Date Raised** | Today's date in MM/DD/YY format |
| **Owner** | Always include both the Extend owner and the merchant-side owner where identifiable. Format: `[Extend person] / [Merchant person]`. If the specific merchant contact is not known from research, default to the merchant name (e.g., `Warby Parker`) for any External-type item — do not leave the merchant side blank. For Internal-type items, Extend owner only is fine. |
| **Status** | Not Started / In Progress / Complete / Blocked |
| **Name** | 3–6 words max. A scannable label, not a sentence. Think Jira ticket title compressed. Bad: "There is a risk that the merchant's API credentials may expire before go-live." Good: "API credential expiry risk" |
| **Description** | Full context — what it is, why it matters, relevant background. If a Jira ticket exists, embed the link: `[TICKET-123](https://helloextend.atlassian.net/browse/TICKET-123)` |
| **Next Steps / Notes** | Prefix with date: `MM/DD/YY: [note]`. If multiple notes, stack them newest-first with separate date prefixes. |
| **Target Completion Date** | MM/DD/YY if determinable from context or commitments made. If no date is evident from research, assign a reasonable best-guess based on the item's urgency and project timeline — and flag it as estimated. Do not leave blank for Action or Risk items; undated items never get followed up on. |
| **Decision** | **Decision items only.** Format: `MM/DD/YY: [what was decided]`. This should be the actual decision, not a description of the discussion. |
| **Decision Date** | **Decision items only.** Date the decision was made or confirmed. |
| **Future Phase?** | Yes / No / blank if unclear |

**Name field discipline** is important — it's the first thing visible when filtering the log on a merchant call. Keep it brief and scannable.

**Type guidance in practice:**
- Internal: "Extend legal review pending on contract terms" — the merchant doesn't need to know this is in flight
- External: "Merchant hasn't completed seller's license application" — this is a shared accountability item

**Category guidance:**
- Pick the most specific fit. When an item spans multiple categories, use judgment to pick the dominant one.
- Exception: anything integration, API, data, or system-related defaults to **Technical** even if it also touches another area (e.g., a data feed issue affecting analytics → Technical, not Analytics).
- When in doubt between two non-Technical categories, pick the one that reflects *who owns the work*, not just what it's about.

### ⛔ Pre-Output Gate — Fill This Before Step 5

**If running in delta mode:** Use the delta checklist in the Delta Research Scope section above instead of this one. Do not fill in this full-sweep checklist for a delta run.

**If running the full sweep:** Do not present results until every required source from research-guide.md Section 9 is confirmed run. Fill in each line. Any blank (not a tool error) = go back and run it now.

```
- Project file: ✅ / ❌
- RAID Log (all 3 tabs): ✅ / ❌ tool error
- Notes doc: ✅ / ❌ tool error
- Salesforce: ✅ / ❌ tool error
- GDrive recent docs: ✅ / ❌ tool error
- GDrive Gemini folder (Method A): ✅ N results / ✅ 0 / ❌ tool error
- Slack primary channel: ✅ / ❌ tool error
- Slack alias search: ✅ / ❌ tool error
- Slack DMs (contacts: [list]): ✅ / ❌ tool error
- Jira (open + project-scoped): ✅ / ❌ tool error
- Gmail 5-day check: ✅ / ❌ tool error
- Gmail 30-day inbound/outbound: ✅ / ❌ tool error
- Slack DM D057X0PEFJS — Chorus recaps: ✅ N recaps for [merchant], all extracted in full / ✅ 0 / ❌ tool error
- Gmail Chorus fallback: ✅ N results / N/A (Slack DM had results) / ❌ tool error
- Gmail Gemini Method B fallback: ✅ N threads found, N opened FULL_CONTENT / N/A (Method A had results) / ❌ tool error
- Zoom: ✅ N results / ✅ 0 / N/A (Recording Tools doesn't list Zoom) / ❌ tool error
```

**⚠️ Slack DM Chorus — extract in full, no truncation:**
The Slack DM messages are pre-parsed — no `get_thread` needed. But extract every action item and every meeting summary bullet for relevant recaps. Do not summarize or drop items. The count of action items extracted must be reported: "✅ 2 recaps, 14 action items total extracted." A recap with 0 action items logged is a hard error — re-read the Slack message before accepting zero.

**⚠️ Gmail Gemini Method B — FULL_CONTENT is mandatory when it runs as fallback:**
If Method B runs (GDrive folder returned 0 results), call `get_thread` with `messageFormat: FULL_CONTENT` on every Gemini email thread — snippets are not sufficient. This line must report both: "✅ 2 threads found, 2 opened with FULL_CONTENT." If threads found ≠ threads opened, go back before proceeding.

### Step 5: Present Results

Always present in three parts:

**Part 1 — Summary header:**
```
## RAID Log Update — [Merchant Name]
Date: MM/DD/YY

Research Sources:
- Project file: ✅ / ❌
- RAID Log: ✅ / ❌ tool error
- Notes doc: ✅ / ❌ tool error
- Salesforce: ✅ / ❌ tool error
- GDrive recent docs: ✅ / ❌ tool error
- GDrive Gemini folder (Method A): ✅ N results / ✅ 0 / ❌ tool error
- Slack primary channel: ✅ / ❌ tool error
- Slack alias search: ✅ / ❌ tool error
- Slack DMs ([list contacts covered]): ✅ / ❌ tool error
- Jira: ✅ / ❌ tool error
- Gmail 5-day: ✅ / ❌ tool error
- Gmail 30-day: ✅ / ❌ tool error
- Slack DM D057X0PEFJS — Chorus recaps: ✅ N recaps for [merchant], N action items extracted / ✅ 0 / ❌ tool error
- Gmail Chorus fallback: ✅ N results / N/A (Slack DM had results) / ❌ tool error
- Gmail Gemini Method B fallback: ✅ N threads found, N opened FULL_CONTENT / N/A (Method A had results) / ❌ tool error
- Zoom: ✅ N results / ✅ 0 / N/A (Recording Tools doesn't list Zoom) / ❌ tool error
[FULL SWEEP / PARTIAL SWEEP — if partial, list exactly what was skipped and why]

Existing items updated: X (list each by ID and Name, with a one-line note on what changed and why)
New items added: Y total (N Risks, N Actions, N Issues, N Decisions)

Needs target date (estimated): [list any new Action or Risk items where no firm date was available — show the best-guess date assigned and the reasoning, so Samir can confirm or adjust before pasting]

Existing items to flag: [surface anything open/blocked that looks stale but wasn't updated — for Samir's awareness only]
```

**Part 2 — Full replacement table:**

Output the **complete RAID log table** — all rows in ID order, with existing rows updated where applicable and new rows appended at the end. This table is designed to be copy-pasted directly into the sheet to replace the current content.

Format the table with all 15 columns in order:
`ID | Last Updated | RAID | Type | Category | Date Raised | Owner | Status | Name | Description | Next Steps / Notes | Target Completion Date | Decision | Decision Date | Future Phase?`

**[NEW]/[UPDATED] prefix rules — applied to the Name field in both the table and the TSV:**
- **New rows this run** → prefix Name with `[NEW]`
- **Existing rows updated this run** → prefix Name with `[UPDATED]`
- **Existing rows not touched this run** → strip any existing `[NEW]` or `[UPDATED]` prefix from the Name field (write the clean name with no prefix)

This means the sheet always reflects exactly what changed in the most recent run — nothing more, nothing less.

**Last Updated rules:**
- **New rows this run** → set `Last Updated` to today's date (MM/DD/YY)
- **Existing rows updated this run** → set `Last Updated` to today's date (MM/DD/YY)
- **Existing rows not touched this run** → preserve the existing `Last Updated` value from the sheet verbatim

**Part 3 — Change detail view:**
After the full table, list each changed row (both new and updated) in a labeled format for easy review:

```
### [ID] — [Name] — [NEW or UPDATED]
- Changed fields: [list only the fields that changed, with old → new value where applicable]
- Source: [what evidence supported this change — e.g., "Slack message from @jane on 04/22", "Jira TICKET-123 closed"]
- Description: [full text — only for new items]
- Next Steps / Notes: [full text including any newly appended note]
- Decision: [only if Decision type and populated]
```

After presenting, confirm: *"The full table is ready to paste into the RAID log — updated rows are marked [UPDATED], new rows are marked [NEW], and prior-run tags have been stripped from untouched rows. Review the change detail below the table, then copy it over when you're ready. Let me know if anything needs adjusting."*

**Part 4 — TSV file output:**

After presenting the table and change detail, always generate a downloadable TSV file saved to `/sessions/.../mnt/outputs/` using the naming convention:

```
{merchant_name}_raid_log_V{N}_{MM-DD-YY}.tsv
```

- Version number (`V1`, `V2`, etc.) increments with each run for the same merchant, enabling audit trail and rollback
- Columns must match the sheet exactly, in order: `ID | Last Updated | RAID | Type | Category | Date Raised | Owner | Status | Name | Description | Next Steps / Notes | Target Completion Date | Decision | Decision Date | Future Phase?`
- Apply the same [NEW]/[UPDATED] prefix rules and Last Updated stamping as the table above — the TSV is the source of truth for what goes into the sheet
- Use tab separators (not commas) so Google Sheets pastes cleanly into columns
- Provide a `computer://` link so Samir can open and copy directly

## Important Constraints

- **⛔ Never substitute manual instructions for the replacement table.** If the full sheet cannot be parsed from the first read attempt, re-fetch it immediately and try again before producing any output. Presenting step-by-step update instructions as a substitute for the replacement table is never acceptable — it defeats the entire purpose of this skill. The only valid output is the complete replacement table.
- **No direct writes to the sheet.** Output a full replacement table for Samir to copy-paste. This preserves traceability — existing rows reflect his last manual edits, with only intentional updates applied on top.
- **Next Steps / Notes is always append-only.** Never overwrite existing note content. Always prepend new notes with a date prefix above the existing text, separated by ` | ` or a newline. Historical notes must stay intact.
- **Updates require evidence.** Only update an existing field if research clearly supports it — a Slack message, email, Jira status change, or meeting note. Do not infer or guess at status changes.
- **Immutable fields stay locked.** ID, RAID type, Category, Date Raised, Name, and Description must not be changed on existing rows unless Samir explicitly instructs it.
- **[NEW]/[UPDATED] prefix is run-scoped.** Prefixes appear only on rows touched in the current run. All other rows must have their Name written with no prefix — actively strip any prefix carried over from a prior run.
- **Last Updated is always stamped today for touched rows.** Never leave Last Updated blank on a new or updated row. Untouched rows carry their existing Last Updated date forward verbatim.
- **Deduplicate new items.** Always compare proposed new items against existing entries. If something is already captured, reference it in the summary but do not re-add it.
- **Date format**: MM/DD/YY everywhere — in date fields, Next Steps prefixes, and Decision prefixes.
- **Decision column**: Only populate for Decision-type RAID items. The value should state the actual decision clearly, not describe that a decision exists.
- **Target Completion Date is required for every Action and Risk.** Never leave it blank. If no firm date is evident from research, assign a best-guess based on urgency and project timeline, and flag it in the Part 1 "Needs target date" section so Samir can confirm or adjust before pasting. Decisions and Issues may have blank target dates if no deadline applies.
- **Err toward logging more**: It's easier to delete a proposed item than to realize later it was never captured. When in doubt, surface it and let Samir decide.
- **Jira links**: If a Jira ticket is associated with an item, always embed it as a markdown link in the Description field.
- **Slack — never skip the native MCP**: Always use `slack_search_channels` to find the external and internal merchant channels by name, then search them directly. Glean Slack results without snippets are a known failure mode — catch it early and switch to the native Slack tools before concluding research is complete.

### Step 6: Write back to the merchant project file

After presenting results, silently update the merchant's project file to keep it in sync.

- **Last researched**: update to today's date in the Overview section
- **Key Decisions**: for every Decision-type RAID item that was added or confirmed during this run, append a corresponding entry to the project file's `## Key Decisions` section. Format: `- **[MM/DD/YY]:** [what was decided]`. Only add net-new decisions not already captured. Do not duplicate entries already in the section. **If this section does not exist, create it** before `## Already Resolved`.
- **Open Action Items**: for every Action-type RAID item that is open or in progress, ensure it is reflected in the project file's `## Open Action Items` section. Format: `- [Action description] — **[Owner]**`. Remove any actions confirmed complete during research. This section should reflect the live open list. **If this section does not exist, create it** before `## Already Resolved`.
- **Open Tickets**: sync any new Jira tickets found during research — add with hyperlink and description. Remove or note any now showing as Done.
- **Recurring Topics**: add any net-new active themes surfaced during research.

After updating, note at the end of your output:
`Project file updated: [brief summary of what changed]`
