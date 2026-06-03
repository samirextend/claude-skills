---
name: add-merchant
description: >
  Scaffold a new merchant project file by running a full research sweep across all sources
  (Salesforce, GDrive, Slack, Calendar, Jira, Gmail/Chorus/Gemini) and writing a populated
  project file to memory/projects/{merchant}.md. Use when the user says "add merchant",
  "create a project file for [merchant]", "onboard [merchant]", or "set up [merchant]".
---

# Add Merchant

Create a new merchant project file for the merchant named in the user's request.

## Path resolution

Project files live at `outputs/memory/projects/` relative to the workspace mount root. Derive the path
dynamically from `${CLAUDE_PLUGIN_ROOT}`:

`{CLAUDE_PLUGIN_ROOT}/../../../outputs/memory/projects/`

For example, if CLAUDE_PLUGIN_ROOT is `/sessions/abc123/mnt/.claude/skills/add-merchant`,
the project files are at `/sessions/abc123/mnt/outputs/memory/projects/`.

**Never hardcode a session ID in any path.** Sessions change; the relative structure does not.

---

## Step 1 — Intake questionnaire (run before any research)

Ask Samir the following questions before doing anything else. Wait for answers — do not assume or proceed without them. These answers directly shape the research scope and project file structure.

**Ask all at once in a single message — three questions only:**

1. **New or existing merchant?**
   - New = never live with Extend before
   - Existing = already live on at least one Extend product

2. **If existing: what is the net new work?**
   (e.g., "adding eComm offers", "new product line", "new channel")
   Keep it brief — one sentence is enough.

3. **Any context I should know upfront that won't show up in research?**
   (e.g., internal constraints, exec pressure, something that happened in a hallway conversation)
   If nothing, say "none."

Everything else — contacts, go-live target, existing program details, GDrive docs, Slack channels — will be found during the research sweep. Do not ask for information that research can provide.

Once answers are received, proceed to Step 2.

---

## Step 1b — Determine research mode

Based on the intake answers, set one of two modes for the rest of this skill:

**Mode A — New merchant (full sweep)**
Use when: merchant has no prior live Extend relationship.
Research scope: all time ranges per research-guide.md defaults. Cast the full net.

**Mode B — Existing merchant, net new work**
Use when: merchant is already live on at least one Extend product and this engagement is scoped to something new.
Research scope:
- Limit all research to the **last 90 days only** — do not surface historical contract data, old tickets, or prior implementation context unless it directly affects the net new work
- In Jira: search for tickets explicitly related to the net new scope only; exclude tickets tagged to the existing live program
- In Gmail/Slack: filter to conversations about the net new work; flag but do not deeply research threads about the existing live program
- In GDrive: read the notes doc for net new context only; note the existing program in a one-line summary and move on
- Project file structure: use two clearly separated sections (see Step 4)

State the mode explicitly before proceeding to research.

---

## Step 2 — Normalize and deduplicate

- Create a kebab-case filename (e.g., "Rugs USA" → `rugsusa.md`, "City Beauty" → `city-beauty.md`)
- Scan the `memory/projects/` directory to confirm no file for this merchant already exists
- If one exists, stop and tell the user — do not overwrite

---

## Step 2 — Full research sweep

Read `research-guide.md` at `{CLAUDE_PLUGIN_ROOT}/../../../Claude/memory/research-guide.md` and run the complete sweep — **Sections 0b through 7** in full. Every section is required. The goal is to produce a project file rich enough that future skills can hit the ground running without compensating for a shallow starting point.

**Do not abbreviate the sweep.** If a tool fails, note it and move on — never block on one source.

**Add-merchant-specific notes:**

- **Notes Doc vs. Gemini doc:** The GDrive implementation notes doc (manually maintained by Samir, typically in the Onboarding/Sales subfolder) is what goes in the `Notes Doc` field. The Gemini auto-generated doc is a separate artifact — use its content to seed Recurring Topics and Key Contacts, but do not use its URL as the Notes Doc.
- **Lookback:** Use the guide's defaults. Since there is no prior project file baseline, cast the full net across all time ranges specified in each guide section.
- **Contacts list:** Seed aggressively — every name, title, and email found across all sources goes into Key Contacts. Easier to prune later than to backfill.
- **Chorus emails:** Note the email date and subject — these confirm what calls have been recorded and help seed the meeting cadence field. Cross-reference with the notes doc for actual next-steps content.

---

## Step 3 — Read the template

Read the template at:
`${CLAUDE_PLUGIN_ROOT}/../../../.remote-plugins/plugin_01FnLu8ZwFqGwRvL57btXWdn/skills/implementation-management/references/merchant-project-template.md`

If that path is not found, fall back to the template at:
`${CLAUDE_PLUGIN_ROOT}/../../../Agenda Prep/merchant-project-template-updated.md`

Use it as the base structure for the new project file.

---

## Step 4 — Create the project file

Write the file to `{CLAUDE_PLUGIN_ROOT}/../../../outputs/memory/projects/{kebab-name}.md`.

Fill in every section where research returned data. For anything not found, leave the
placeholder comment from the template — do not fabricate or guess.

**Mode B only — two-section structure:**
When in Mode B (existing merchant, net new work), structure the project file with a clear
separation at the top so all downstream skills (agenda-builder, raid-log, etc.) know exactly
what to focus on:

```
## ⚠️ Existing Merchant — Net New Work
This project file covers **[net new work description]** only.
Existing live program: [brief one-liner, e.g., "PP In-Store live since 2024 — not in scope here"]
All research, action items, and agenda topics below are scoped to the net new work unless explicitly labeled otherwise.
```

Place this block immediately after the Overview section. Every downstream skill that reads this
file should treat it as the scope boundary — do not pull in historical data or existing program
topics unless Samir explicitly asks.

Specific population rules:
- **Last researched**: set to today's date in the Overview section
- **Key Contacts**: include everyone found across Salesforce, GDrive, Calendar, Slack, and Gmail
- **Notes Doc URL**: use the manually maintained implementation notes doc found in GDrive
  (typically in the Onboarding/Sales subfolder). Do not use the Gemini auto-generated doc.
- **Slack**: list primary channel + all secondary channels/group DMs discovered, with channel IDs
- **Open Tickets**: all open Jira tickets found in the sweep, formatted with hyperlink + description
- **Key Decisions**: seed with any decisions explicitly confirmed during research (from notes doc,
  Chorus, Gemini, or email). Format each as `- **[Date]:** [decision]`. Leave blank if no
  confirmed decisions found — do not guess or infer.
- **Open Action Items**: seed with any open action items found across Chorus recaps, notes doc,
  Slack, and Gmail. Format each as `- [Action description] — **[Owner]**`. Only include items
  with no evidence of completion in any source.
- **Recurring Topics**: seed with any themes that appeared across multiple sources
- **Already Resolved**: leave blank — nothing is resolved until confirmed resolved in a source
- **Jira Search Config**: if the generic text search returned a lot of noise, note a tighter JQL

---

## Step 5 — Confirm via Slack DM

Send a Slack DM to Samir Ali (user ID: `U02H2DBBE95`) with:
- Merchant name and the file path created
- What was pre-populated vs. what still needs to be filled in manually
- Any open questions that would improve the file (e.g., notes doc URL if not found,
  primary Slack channel if unclear, contacts that need confirmation)
- A one-line summary of what each source contributed (e.g., "Jira: 3 open tickets seeded;
  Chorus: 2 recap emails found; Slack: 2 channels discovered")
