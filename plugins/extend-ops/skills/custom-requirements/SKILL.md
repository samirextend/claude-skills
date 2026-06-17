---
name: custom-requirements
description: Track and update custom product requirements for merchant implementations. Use whenever Samir says "update the custom requirements for [merchant]", "add a requirement for [merchant]", "what custom tickets do we need for [merchant]", "log a custom requirement for [merchant]", "pull up the requirements tracker for [merchant]", "run the custom requirements skill for [merchant]", or any variation of identifying, documenting, or updating Extend product enhancements/features/configs needed for a merchant's integration. Also triggers when reviewing onsite notes, meeting recaps, or Slack threads and Samir says "what custom requirements came out of this" or "what do we need to build for [merchant]". This skill reads the live Google Sheet tab as baseline before making any changes — preserving items Samir manually added, removed, or edited.
---

# Custom Requirements Tracker Skill

This skill identifies, documents, and updates the custom product requirements needed for a merchant's Extend integration. It reads the live Google Sheet tab as baseline, researches sources for new requirements or updates to existing ones, and outputs a full replacement table ready to paste back into the sheet, along with a versioned TSV file.

## Column Schema

The custom requirements tab uses these columns in order:

| Column | Description |
|--------|-------------|
| **Requirement** | Short name for the requirement (3–8 words, Jira-title style) |
| **Description** | 1–2 sentences capturing the use case and key context for engineering review |
| **Team** | Extend team responsible: MINT, PEX, POST, EMIT, DE, Analytics, or Invoicing |
| **Type** | Enhancement / Feature / Config / Integration |
| **Impacted Objects** | Comma-separated list of APIs, portals, webhooks, or systems affected |
| **Ticket** | Jira ticket link(s) if filed — use the ticket number as display text with the URL embedded as a Google Sheets HYPERLINK formula: `=HYPERLINK("https://helloextend.atlassian.net/browse/TICKET-123","TICKET-123")` |
| **Priority** | High / Medium / Low |
| **Notes** | Date-prefixed context, source references, and prior art. Always append — never overwrite. |
| **Next Steps** | Concrete next actions for this requirement |
| **Status** | Not Started / In Progress / Complete / Blocked |

**Team definitions:**
- **MINT** — Merchant integrations (order ingestion, API setup, in-store/eComm integration plumbing)
- **PEX** — Platform/contracts (contract lifecycle, proration, activation states, cancellations)
- **POST** — Claims and post-purchase (claim filing, adjudication, webhooks, portal UX, email)
- **EMIT** — Enterprise merchant integrations (SFTP, invoice automation, data feeds)
- **DE** — Data engineering (data pipelines, reporting infrastructure)
- **Analytics** — Reporting, dashboards, SOP platform outputs
- **Invoicing** — Invoice/TDF structure, credit memos, finance reconciliation

**Type guidance:**
- **Enhancement** — Extends or modifies existing Extend functionality
- **Feature** — Net-new capability that doesn't exist in the platform
- **Config** — Store-level or merchant-specific configuration (no eng required, or minimal)
- **Integration** — System-to-system connection work (SFTP, webhook endpoint, OMS hookup)

---

## Ad Hoc Single-Row Update Flow

When Samir asks to update or add context to a specific existing row (rather than running a full research sweep), use this abbreviated flow:

1. **Read the existing row from the live sheet first.** Always fetch the current sheet content before generating any update — even for a single-row change. This ensures immutable fields are preserved verbatim and Notes content is appended correctly.

2. **Only touch mutable fields.** Apply updates only to fields that have new evidence-backed changes: Notes (append), Next Steps, Status, Ticket, Priority, Team. Never touch Requirement, Description, Type, or Impacted Objects unless Samir explicitly instructs it.

3. **Output as a TSV file — never inline text.** Ad hoc row updates must always be written as a `.tsv` file saved to `/sessions/.../mnt/Claude/` using the naming convention:
   ```
   {merchant_name}_{ticket_or_requirement_slug}_update_{MM-DD-YY}.tsv
   ```
   Include a header row followed by the single updated data row. Provide a `computer://` link so Samir can open and paste directly into the sheet.

4. **Never provide a row update as inline chat text.** Inline text does not paste cleanly into Google Sheets. Always use a file.

**Ticket validation rule:** When Samir says "Ticket X is missing context" or asks to update a specific ticket's row, cross-check that the described context actually belongs to that ticket's scope before executing. If the content doesn't match the ticket (e.g., Core glasses context being attributed to an AI glasses ticket), flag the mismatch and confirm the correct ticket before writing anything.

---

## Sheet → Jira Flow (Source of Truth Rule)

**The Google Sheet is always the source of truth. Jira is downstream.**

The correct update flow is always:
1. Update the sheet row first (via TSV file output)
2. Update the Jira ticket after, using the updated sheet content as the source

Never update a Jira ticket directly without first updating the corresponding sheet row. If Samir asks to update a Jira ticket, always output the updated sheet row first and confirm it's been pasted before pushing to Jira.

**Reverse sync (Jira → Sheet):** If a Jira ticket is updated by another team member (POST, PEX, etc.) with new engineering context, that context should flow back into the sheet Notes field on the next skill run. When researching, check Jira ticket comments and status changes and treat them as evidence for sheet updates.

---

## Workflow

### Step 1: Locate the Merchant's Custom Requirements Tab

1. Read the merchant's project file. Resolve the path using the skill's own location: `{skill_base_dir}/../../../outputs/memory/projects/{merchant}.md` (where skill_base_dir is the directory containing this SKILL.md, e.g. `/sessions/abc123/mnt/.claude/skills/custom-requirements`). This resolves to `/sessions/abc123/mnt/outputs/memory/projects/{merchant}.md`.
2. Look for the **Implementation Plan & RAID Log** Google Sheets file ID or link stored there
3. If not present, locate it in Google Drive using this folder structure:
   - **Root merchants folder**: `https://drive.google.com/drive/folders/16w2ilKVgPtXgwKEmYeHE6-viNsGrBZTG`
   - Navigate into the appropriate **status subfolder** (In Progress, Live, On Hold, or similar)
   - Then into the **merchant-specific subfolder**
   - Then into the **onboarding subfolder**
   - The file title will contain the merchant name + "RAID" + "Plan" — e.g., "[Merchant] Extend Implementation Plan & RAID Log"
   - Once found, save the file ID and sheet tab GID into the merchant's project file for future reference
4. Read the **custom requirements tab** of the spreadsheet — this is the tab named something like "Custom Requirements", "Custom Req", or "Custom Requirements / Tickets"
   - For Warby Parker: file `1gXwm_Fsd4WTPQrCYXAYf_5hoWMeTGYy8dKBhstIZysI`, tab `gid=382619125`
5. Capture **all existing rows in full** — these are the baseline. Preserve them verbatim; only apply targeted, evidence-backed updates on top.
6. Note the total count of existing rows so you can identify what's new vs. updated in your summary.

### Step 2: Deep Research

Read `research-guide.md` at `{skill_base_dir}/../../../Claude/memory/research-guide.md` and follow **Sections 0b through 7** for all research mechanics: key merchant artifacts (notes doc, Implementation Plan, RAID log, custom requirements sheet), Salesforce, Calendar, GDrive, Slack, Gmail (including Chorus and Gemini dual-method), Zoom, Jira, alias discipline, dynamic contacts list, completed-action checks, and cross-source reconciliation.

**Custom-requirements-specific overrides:**

- **Extended lookback:** Use **8 weeks** for Slack and GDrive instead of the guide's 3-week and 4-week defaults. Requirements surface from onsite notes, early discovery calls, and kickoff conversations that may be weeks old. Gmail 30-day default applies as-is.
- **Research focus:** Hunt for anything that implies Extend needs to build, configure, or modify something specifically for this merchant. Flag any "we need to build X," "X is required," or "can Extend support Y?" language — don't wait for something to be explicitly labeled as a requirement.
- **Jira reverse sync:** When checking Jira, read ticket comments and status changes — engineering context added by other teams (POST, PEX, MINT) since the last run should flow back into the sheet Notes field.

**What constitutes a custom requirement:**

A custom requirement belongs in this tracker if Extend needs to build, modify, configure, or integrate something specifically for this merchant that isn't already part of the standard implementation. Ask: *Would a different merchant getting a standard Extend integration need this? If not, it's custom.*

Requirements belong here:
- Product enhancements (portal changes, new webhook fields, API parameters)
- Net-new features scoped for this merchant's use case
- Merchant-specific configuration (claim flow rules, email suppression, fulfillment flags)
- Integrations with the merchant's systems (OMS, SFTP, POS, deep links)

Do not log:
- Standard implementation steps (contract setup, test orders, UAT)
- Internal process tasks (scheduling reviews, sending templates)
- Items already captured that would duplicate an existing row
- Business decisions about what data the merchant will pass — these belong in the RAID log, not the requirements sheet

### Step 3: Check Jira for Ticket Updates

For every existing requirement with a blank Ticket column, search Jira using the requirement name and relevant project board. If a ticket has been filed, populate the Ticket field with an embedded markdown link: `[TICKET-123](https://helloextend.atlassian.net/browse/TICKET-123)`.

Also check status: if a ticket is Done/Closed, that's evidence to update the Status column to Complete.

### Step 4: Identify Updates to Existing Rows

Before proposing new requirements, review every existing row against your research findings. Only update fields when there is clear evidence from a source — never assume.

**Fields that may be updated:**

| Field | When to update |
|-------|---------------|
| **Status** | Evidence shows movement — ticket closed → Complete, blocker flagged → Blocked, work started → In Progress |
| **Ticket** | A Jira ticket was found that corresponds to this requirement |
| **Notes** | New context exists. **Always append — never overwrite.** Prepend new note with date prefix above existing: `MM/DD/YY: [new note] \| [existing notes preserved]` |
| **Next Steps** | New actions have been committed to or prior actions completed |
| **Priority** | Explicitly re-prioritized in a meeting or Slack thread |
| **Team** | Reassigned to a different team based on new information |

**Fields that must never be changed on existing rows:**
- **Requirement** — immutable (the name anchors the row)
- **Description** — treat as immutable; surface new context in Notes instead
- **Type** — only change if Samir explicitly instructs it
- **Impacted Objects** — only change if Samir explicitly instructs it

For each updated row, note what changed and the source that supported it — this goes in the summary.

### Step 5: Populate All Fields for New Requirements

For each new requirement identified, fill in every applicable field:

| Field | Guidance |
|-------|----------|
| **Requirement** | 3–8 words max. Scannable, Jira-title style. Bad: "There is a need for redirect after claim." Good: "Redirect URL post-claim approval" |
| **Description** | 1–2 sentences. What needs to be built, why it's needed, and key context for an engineering review. Be specific — include system names, stakeholders, or constraints that matter. |
| **Team** | Pick the Extend team that owns the build. When in doubt: claims/portal/webhook work → POST; contract lifecycle → PEX; SFTP/invoice automation → EMIT; API/order ingestion → MINT. |
| **Type** | Enhancement / Feature / Config / Integration |
| **Impacted Objects** | List the APIs, portals, webhooks, or systems touched. E.g., "Claims API, Customer Portal, Merchant Portal" |
| **Ticket** | Leave blank if not yet filed. If found in Jira, populate as: `=HYPERLINK("https://helloextend.atlassian.net/browse/TICKET-123","TICKET-123")` |
| **Priority** | High / Medium / Low. High = blocks go-live or a key workstream. Medium = important but not blocking. Low = nice-to-have or Phase 2+. |
| **Notes** | Prefix with today's date: `MM/DD/YY: [note]`. Include: where this came from (e.g., onsite, Slack, meeting notes), any relevant prior art or reference tickets (e.g., "Reference FUL-90 for pattern"), and any key constraints. |
| **Next Steps** | Concrete next actions — who needs to do what. E.g., "File POST ticket. Confirm WP redirect URLs for each context. Validate with WP eComm team." |
| **Status** | Default to Not Started unless there's evidence otherwise. |

**Requirement name discipline** is important — it's the first thing visible when filtering the tab. Keep it brief and scannable.

**Priority calibration:**
- High: Required for go-live of a specific phase (Phase 1 Core, Phase 2 AI, etc.)
- Medium: Needed for a complete integration but not immediately blocking
- Low: Deferred to a future phase or explicitly optional

### ⛔ Pre-Output Gate — Fill This Before Step 6

Do not present results until every required source from research-guide.md Section 9 is confirmed run. Fill in each line. Any blank (not a tool error) = go back and run it now.

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
- Slack DM D057X0PEFJS — Chorus recaps: ✅ N recaps for [merchant] / ✅ 0 / ❌ tool error
- Gmail Chorus fallback: ✅ N results / N/A (Slack DM had results) / ❌ tool error
- Gmail Gemini Method B fallback: ✅ N results / N/A (Method A had results) / ❌ tool error
- Zoom: ✅ N results / ✅ 0 / N/A (Recording Tools doesn't list Zoom) / ❌ tool error
```

### Step 6: Present Results

Always present in four parts:

**Part 1 — Summary header:**
```
## Custom Requirements Update — [Merchant Name]
Date: MM/DD/YY
Existing rows updated: X (list each by Requirement name, with a one-line note on what changed and why)
New requirements added: Y total
Jira tickets populated: Z (list each)

Items to flag: [surface anything open/Not Started with no recent activity — for Samir's awareness]
```

**Part 2 — Full replacement table:**

Output the **complete custom requirements table** — all rows in original order, with existing rows updated where applicable and new rows appended at the end. This table is designed to be copy-pasted directly into the sheet to replace current content.

Format with all 10 columns in order:
`Requirement | Description | Team | Type | Impacted Objects | Ticket | Priority | Notes | Next Steps | Status`

**Mark changed rows clearly:**
- Prefix Requirement with `[NEW]` for newly added rows
- Prefix Requirement with `[UPDATED]` for existing rows where one or more fields changed
- Unchanged rows have no prefix

Samir can remove the tags after reviewing and pasting.

**Part 3 — Change detail view:**
After the full table, list each changed row (both new and updated):

```
### [Requirement Name] — [NEW or UPDATED]
- Changed fields: [list only the fields that changed, with old → new where applicable]
- Source: [what evidence supported this — e.g., "Slack thread in #external-warby-extend 04/30", "Jira POST-123 filed"]
- Description: [full text — only for new requirements]
- Notes: [full text including newly appended note]
```

After presenting, confirm: *"The full table is ready to paste into the custom requirements tab — updated rows are marked [UPDATED], new rows are marked [NEW]. Review the changes below, then copy it over when you're ready. Let me know if anything needs adjusting."*

**Part 4 — TSV file output:**

Always generate a downloadable TSV file saved to the outputs directory using this naming convention:

```
{merchant_name}_custom_requirements_V{N}_{MM-DD-YY}.tsv
```

- Version number (`V1`, `V2`, etc.) increments with each run for the same merchant — check the outputs directory for existing versions and increment accordingly
- Columns must match the sheet exactly, in order: `Requirement | Description | Team | Type | Impacted Objects | Ticket | Priority | Notes | Next Steps | Status`
- Use tab separators (not commas) so Google Sheets pastes cleanly into columns
- Provide a `computer://` link so Samir can open and copy directly

---

## Important Constraints

- **Read the live sheet first, every time.** Always read the current tab content before making any updates — including ad hoc single-row changes. This preserves items Samir manually added, removed, or edited between runs. Never regenerate from scratch using prior TSV files as the baseline.
- **Notes is always append-only.** Never overwrite existing note content. Always prepend new notes with a date prefix above the existing text: `MM/DD/YY: [new note] | [prior notes]`. Historical notes must stay intact.
- **Updates require evidence.** Only update an existing field if research clearly supports it — a Slack message, email, Jira status, or meeting note. Do not infer or guess.
- **Immutable fields stay locked.** Requirement name, Description, Type, and Impacted Objects must not be changed on existing rows unless Samir explicitly instructs it.
- **Deduplicate new items.** Always compare proposed new requirements against existing entries before adding. If already captured, reference it in the summary but do not re-add.
- **Date format:** MM/DD/YY everywhere — in date fields and Notes prefixes.
- **Jira links:** In the Ticket column, always use a Google Sheets HYPERLINK formula with the ticket number as display text: `=HYPERLINK("https://helloextend.atlassian.net/browse/TICKET-123","TICKET-123")`. This renders as a clickable "TICKET-123" link when pasted into the sheet.
- **Slack — never skip the native MCP.** Always use `slack_search_channels` to find channels by name, then search them directly. Glean Slack results without snippets are a known failure mode — switch to native Slack tools before concluding research.
- **Team assignment:** When uncertain between POST and another team, default to POST for claims/portal/webhook work. When uncertain between MINT and another team, default to MINT for order ingestion and API integration work.
- **Output format:** All row outputs — whether single-row ad hoc updates or full replacement tables — must be delivered as TSV files with a `computer://` link. Never provide sheet rows as inline chat text.
- **Sheet is always SOT before Jira.** Never update a Jira ticket without first outputting the updated sheet row as a TSV file. Sheet → Jira is the only valid flow direction for ad hoc updates.
- **Validate ticket scope before updating.** If Samir references a specific ticket number for an update, verify the described context actually belongs to that ticket before executing. If content doesn't match the ticket's scope, flag the mismatch and confirm the correct ticket first.

---

## Step 7: Write Back to the Merchant Project File

After presenting results, silently update the merchant's project file at `{skill_base_dir}/../../../Claude/memory/projects/{merchant}.md` to keep it in sync.

- **Last researched**: update to today's date in the Overview section
- **Open Tickets**: sync any newly discovered or filed Jira tickets — add with hyperlink and description. Mark any tickets now showing as Done.
- **Key Decisions**: if any decisions about a requirement's scope, ownership, or phase were confirmed during research (e.g., "confirmed Phase 2", "POST owns this"), append to `## Key Decisions`. Format: `- **[MM/DD/YY]:** [what was decided]`. Only add net-new decisions not already captured. **If this section does not exist, create it** before `## Already Resolved`.
- **Recurring Topics**: add any net-new active requirement themes surfaced during research (e.g., "claims portal customization", "SFTP data feed", "SSO for Merchant Portal").
- **Open Action Items**: if research surfaced committed next actions on requirements (e.g., "Samir to file POST ticket for redirect URL requirement"), add to `## Open Action Items`. Format: `- [Action description] — **[Owner]**`. **If this section does not exist, create it** before `## Already Resolved`.

After updating, note at the end of your output:
`Project file updated: [brief summary of what changed]`
