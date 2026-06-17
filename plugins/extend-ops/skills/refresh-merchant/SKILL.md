---
name: refresh-merchant
description: >
  Run a full research sweep for an existing merchant and fully re-sync their project file.
  Use when a project file feels significantly stale, before a major milestone, or when new
  sections need to be backfilled. Distinct from add-merchant (new merchants only) and the
  incremental write-backs that agenda-builder and raid-log do after each run.
  Triggers: "refresh [merchant]", "update the project file for [merchant]",
  "re-sync [merchant]", "the [merchant] file is stale", "backfill [merchant]".
---

# Refresh Merchant

Fully re-sync an existing merchant project file by running the complete research sweep
defined in `research-guide.md` and writing all findings back to the project file.

This skill does not create a new file — it updates an existing one. The project file's
structure and template are preserved; only content is updated.

---

## Step 1 — Load the project file

Resolve the path using the skill's own location:
`{skill_base_dir}/../../../Claude/memory/projects/{merchant}.md`

Read the full file. It provides:
- Starting contacts list (seed for accumulated contacts list during research)
- Slack channels and aliases to search
- Notes doc URL
- Open tickets (starting point for Jira comparison)
- Existing Key Decisions and Open Action Items (baseline before overwriting)

If the file doesn't exist, stop and tell the user to run `add-merchant` instead.

---

## Step 2 — Full research sweep

Run the complete research sweep per `research-guide.md`. Read that file first — it is
the canonical guide for all sources, search strategies, and quality rules.

Path: `{skill_base_dir}/../../../Claude/memory/research-guide.md`

**Do not abbreviate the sweep.** Every section of the research guide is required. The
whole point of this skill is a deep, comprehensive sync — not a quick pass.

---

## ⛔ Pre-Reconcile Gate — Fill This Before Step 3

Do not reconcile or update the project file until every required source from research-guide.md Section 9 is confirmed run. Fill in each line. Any blank (not a tool error) = go back and run it now.

```
- Project file: ✅ / ❌
- RAID Log (all 3 tabs): ✅ / ❌ tool error
- Notes doc: ✅ / ❌ tool error
- Salesforce: ✅ / ❌ tool error
- GDrive — root folder scan (parentId = '[root folder ID]'): ✅ N results / ❌ tool error
- GDrive — each known subfolder scanned (list subfolder IDs run): ✅ / ❌ tool error
- GDrive — Glean backstop (app:gdrive [merchant name], all aliases): ✅ N results / ✅ 0 / ❌ tool error
- GDrive Gemini folder (Method A): ✅ N results / ✅ 0 / ❌ tool error
- Slack primary channel: ✅ / ❌ tool error
- Slack alias search: ✅ / ❌ tool error
- Slack DMs (contacts: [list]): ✅ / ❌ tool error
- Jira (open + project-scoped): ✅ / ❌ tool error
- Gmail 5-day check: ✅ / ❌ tool error
- Gmail 30-day inbound/outbound: ✅ / ❌ tool error
- Slack DM D057X0PEFJS — Chorus recaps: ✅ N recaps for [merchant] / ✅ 0 / ❌ tool error
- Gmail Chorus fallback: ✅ N results / N/A (Slack DM had results) / ❌ tool error
- Gmail Gemini Method B fallback: ✅ N results / N/A (Method A had results) / ❌ tool error
- Zoom: ✅ N results / ✅ 0 / N/A (Recording Tools doesn't list Zoom) / ❌ tool error
```

## Step 3 — Reconcile and update the project file

After completing research, update every section where new information was found.
Preserve existing content unless research explicitly contradicts it.

### Overview
- Update `Last researched` to today's date
- Update `Status` if implementation phase has changed (e.g., In Progress → Live)
- Update `Go-live target` if a new date was confirmed

### Key Contacts
- Add any new names/emails/titles found across all sources
- Do not remove existing contacts — flag any that appear inactive with a note

### Slack
- Add any newly discovered channels with IDs
- Update primary channel if it has changed

### Open Tickets
- Fully re-sync from Jira: add new open tickets, remove or mark any now Done
- Every ticket must be formatted as a hyperlink with description

### Key Decisions
- Append any net-new decisions confirmed during research
- Format: `- **[Date]:** [what was decided]`
- Do not remove existing entries — they are historical record
- If the section doesn't exist, create it before `## Already Resolved`

### Open Action Items
- Fully replace with the current live list derived from research
- Cross-check each existing item against all sources — only carry forward items
  with no completion evidence in notes doc, Gmail, or Slack
- Format: `- [Action description] — **[Owner]**`
- If the section doesn't exist, create it before `## Already Resolved`

### Recurring Topics
- Add net-new active themes surfaced during research
- Move any items confirmed resolved to `## Already Resolved`

### Already Resolved
- Move any Recurring Topics items that research confirmed are closed

### Also search as
- Validate this field contains plain readable text (not commented out or empty)
- Add any new aliases discovered (ticker symbols, shorthand, domain variants)

### Key Reference Docs
- For every doc already in `## Key Reference Docs` that was re-read during this sweep: update `Last Modified` date, `Excerpt`, and add a delta note if content changed. Format: 2-4 sentences, dense enough to use without re-fetching (specific field names, plan IDs, tab names, dates, owners, caveats).
- Do not touch entries for docs that were not re-read and have not changed.
- **Auto-catalogue any new doc surfaced during research** — do not wait for manual bootstrapping. For every doc found via the folder scan, Glean backstop, Gmail share notification, or Slack link that is not already in the project file: classify it (living doc vs. reference doc), read it, and add it to the appropriate section with file ID, URL, last modified date, and a 2-4 sentence excerpt. Living docs go to their dedicated sections; reference docs go to `## Key Reference Docs`. This is the mechanism by which the catalogue grows over time — every refresh-merchant run should leave the project file with a more complete doc index than it started with.

**Full subfolder scan (refresh-merchant only):** Because `parentId` queries do not recurse, run a separate `parentId = '[subfolder ID]'` query for every subfolder ID listed in the project file (`## GDrive Subfolder` section). Always use the **current implementation folder ID** as the root of the scan — not the master root. Some merchants (e.g., CarParts) have a master root containing older past-implementation folders that produce noise; the project file labels these distinctly. If only one folder ID is listed, use that. For any subfolder not yet in the project file, list the current implementation folder first, identify subfolders by mimeType = folder, add their IDs to the project file, then query each. Any doc found that is not already catalogued gets added per the auto-catalogue rule above.

---

## Step 4 — Report what changed

After updating the file, output a concise summary in the conversation:

```
## Project File Refreshed — [Merchant Name]
Last researched: [today's date]

Changes made:
- Contacts: [X added]
- Key Decisions: [X added]
- Open Action Items: [X updated — Y removed as complete, Z added]
- Open Tickets: [X added, Y closed]
- Recurring Topics: [X added, Y moved to Resolved]
- Aliases: [any changes]

Research Sources:
✅ / ⚠️ / ❌ [source] — [brief note]
```

Flag anything that couldn't be confirmed and may need manual follow-up.
