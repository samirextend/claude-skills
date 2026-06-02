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

## Step 1 — Normalize and deduplicate

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
