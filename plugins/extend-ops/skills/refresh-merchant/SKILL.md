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
