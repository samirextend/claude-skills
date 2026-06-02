---
name: ccerp
description: >
  Automates the end-to-end CCERP (Customer Commit & Enhancement Request Process) ticket creation workflow for Extend merchant implementations. Use this skill whenever Samir says anything like "create CCERP tickets", "file CCERP", "run CCERP", "update CCERP tickets", "assess requirements for CCERP", or any variation of creating/reviewing/filing Jira tickets for merchant custom engineering requirements. Also trigger when the user asks to sync Jira tickets back to the custom requirements sheet, or to check which requirements still need tickets filed. This skill should be used proactively whenever merchant requirements and Jira ticket creation are mentioned together.
---

# CCERP Ticket Creation Skill

Automates assessing, drafting, and creating CCERP Jira Stories for a merchant, using the merchant's custom requirements spreadsheet as the source of truth and the merchant's project MD for rich implementation context.

## Key Facts

**Jira:**
- cloudId: `aacaf7fa-64aa-47bb-98b7-a6234b6ba8fe`
- Project key: `CCERP`
- Board: https://helloextend.atlassian.net/jira/software/c/projects/CCERP/boards/118
- Child issue type: Story (always)
- Parent issue type: Epic
- Default status: Intake — appropriate for all new tickets, including placeholders
- Assignee: always unassigned on child stories
- Workflow: Intake → In Progress → In Review → Approved / Backlog / Archived
- Reviewed Mondays 4pm CST by Solutions + Product leadership

**Custom fields:**
| Field | customfield ID | Values |
|---|---|---|
| Customer | `customfield_10147` | Array of strings e.g. `["WarbyParker"]` |
| Request Type | `customfield_10148` | `Customer Enhancement` / `Customer Request` / `Customer Commit` |
| Product Type | `customfield_10335` | `Product Protection` / `Shipping Protection` / `Product & Shipping Protection` / `PPA` |
| XFNL Teams | `customfield_10812` | Array of objects: `[{"value": "POST"}]` — valid values: POST, MINT, XAD, PEX, DES, SE, RAD, SECENG |
| Epic Name | `customfield_10011` | Short label for the epic (e.g. "Warby Parker PP") — Epic only |
| Go-live Date | `customfield_10431` | ISO date string e.g. `"2026-07-13"` — Epic only |
| Merchant Go Live Blocker | `customfield_10911` | Array: `[{"value": "Yes"}]` or `[{"value": "No"}]` |

**Sheet column order:**
`Requirement | Description | Team | Type | Impacted Objects | Ticket | Priority | Notes | Next Steps | Status`

**HYPERLINK formula format for Ticket column:**
`=HYPERLINK("https://helloextend.atlassian.net/browse/CCERP-XXX","CCERP-XXX")`

---

## Merchant Reference

Project MD path: `{skill_base_dir}/../../outputs/memory/projects/{merchant}.md`

**Warby Parker specifics:**
- Parent epic: CCERP-576
- Customer label: `WarbyParker`
- Product Type: `Product Protection`
- Sheet: file ID `1gXwm_Fsd4WTPQrCYXAYf_5hoWMeTGYy8dKBhstIZysI`, tab `gid=382619125`
- Phase 1 go-live: July 13, 2026 (Core eyewear, Springfield OMS)
- Phase 2 go-live: ~September 2026 (AI glasses, Oracle ERP/OMG OMS)
- PHI/HIPAA constraints on AI glasses lens encoding
- Okta SAML/SSO for Merchant Portal
- WP owns claims fulfillment; Extend reimburses at flat rate per plan

For other merchants: sheet location is found via their project MD or the Drive lookup in Step 1.

---

## Workflow

### Step 1 — Read Context

**1a. Read the merchant project MD** (`{skill_base_dir}/../../outputs/memory/projects/{merchant}.md`)

Extract: CCERP parent epic key (if stored), sheet file ID/link (if stored), go-live dates, phase structure, OMS/tech stack, team contacts, open decisions, portal/integration specifics. This context is what makes ticket descriptions rich without re-running research.

**1b. Locate the custom requirements sheet**

The sheet is the source of truth — do not proceed without it. Resolve in this order:

1. **Check the project MD** for a stored sheet file ID, link, or reference to the merchant's Implementation Plan & RAID Log doc
2. **Search Google Drive** if not found in the project MD:
   - Root merchants folder: `https://drive.google.com/drive/folders/16w2ilKVgPtXgwKEmYeHE6-viNsGrBZTG`
   - Navigate into the appropriate status subfolder (In Progress, Live, On Hold, etc.) → merchant subfolder → onboarding subfolder
   - File title contains merchant name + "RAID" + "Plan" — e.g., "[Merchant] Extend Implementation Plan & RAID Log"
   - The custom requirements tab is named something like "Custom Requirements", "Custom Req", or "Custom Requirements / Tickets"
3. **If found:** Read all rows in full. Save the file ID and tab GID back to the project MD for future runs.
4. **If not found anywhere — stop immediately** and tell the user:

   > "No custom requirements sheet found for {merchant}. The CCERP skill depends on this as its source of truth. Please run the **custom-requirements skill** first to generate and populate it, then re-run CCERP."

   Do not attempt to draft tickets from memory, and do not ask the user to re-enter requirements manually.

### Step 2 — Resolve the Parent Epic (CRITICAL — do this before anything else)

The parent epic is the merchant-level Epic on the CCERP board that all child Stories must be filed under. Never create child Stories without a confirmed parent epic — orphaned or wrongly-parented tickets cause board confusion.

**Resolution flow:**

1. **Check project MD first.** If the merchant's project MD contains a CCERP parent epic key (e.g., `CCERP-576`), use that as the candidate.

2. **Search Jira regardless.** Search the CCERP project by merchant name to verify the epic exists and to catch any discrepancy. Use `searchJiraIssuesUsingJql` with a query like `project = CCERP AND issuetype = Epic AND text ~ "{merchant name}"`.

3. **Confirm with the user.** Even if you find a match, confirm it on the first run for a merchant: _"I found CCERP-XXX — '{summary}'. Is this the right parent epic for {merchant}?"_ Do not proceed to drafting child stories until confirmed.

4. **If no epic found:** Tell the user and offer to create one. Draft the epic for review before creating — do not auto-create. See **Parent Epic Structure** below.

5. **After confirmation or creation:** Write the epic key back to the merchant's project MD so future runs can reference it directly. Add a line like: `CCERP Parent Epic: CCERP-XXX`.

**Parent Epic Structure:**

When creating a parent epic, use these fields — simpler than Stories, no description template required:

```
issueTypeName: Epic
summary: "{Merchant Name} {Product} Onboarding"  (e.g. "Warby Parker PP Onboarding")
description: "Custom requirements for {Merchant Name} {Product} {channel} Onboarding"
customfield_10011 (Epic Name): "{Merchant Name} {Product short}"  (e.g. "Warby Parker PP")
customfield_10147 (Customer): ["{MerchantLabel}"]
customfield_10431 (Go-live Date): "{ISO date of primary go-live}"
priority: Low  (epics are container tickets, not work items)
```

Do NOT set XFNL Teams, Request Type, or Product Type on the epic — these belong on child Stories only.

### Step 3 — Check Existing Child Tickets

Search the CCERP project for child Stories under the confirmed parent epic. For every row in the sheet with a blank Ticket column, check whether a ticket already exists. If it does, note the ticket number — populate the Ticket column rather than creating a duplicate.

### Step 4 — Ticket Readiness Assessment

Evaluate each row. The bar should lean toward filing — Intake exists precisely for requirements that aren't fully scoped yet. Only hold back when a ticket truly can't be articulated.

**Create a ticket when:**
- The requirement is specific enough to write a meaningful description
- Team, priority, and impacted objects can be identified
- No ticket already exists for this row
- Sheet Status is Not Started or In Progress (both are fine for Intake)

**Flag for user review (don't create yet) when:**
- An active architecture decision is unresolved and the ticket can't be scoped at all
- Notes explicitly say "no near-term scoping needed" or equivalent
- The requirement is so vague that no acceptance criteria can be written — even placeholder ones

**Skip silently when:**
- A ticket already exists (populate Ticket column instead)
- Row is the parent epic row

Present the readiness assessment — what you'll create, what you're flagging and why — and let the user override before drafting.

### Step 5 — Draft Tickets

For each ticket, fill all fields:

**Summary:** `{Merchant Name} — {Requirement Name}` — always prefix with merchant name.

**Request Type inference:**
- `Customer Enhancement` — modifies or extends existing Extend functionality
- `Customer Request` — net-new capability that doesn't exist in the platform
- `Customer Commit` — contractually committed deliverable

**Merchant Go Live Blocker logic:**

Set `customfield_10911` to `[{"value": "Yes"}]` or `[{"value": "No"}]` on every Story. Use this logic to determine the value:

- **Yes** — the ticket is directly required for a phase go-live (Phase 1 or Phase 2). Signals: explicitly tied to launch functionality, blocks customer-facing flow, or the sheet/notes indicate it must be resolved before go-live
- **No** — the ticket is important but not blocking launch. Signals: Phase 2 item when filing under Phase 1, placeholder ticket, enhancement to existing flow, or explicitly noted as non-blocking

If you're unsure based on the available context, ask the user before creating.

**XFNL team mapping from sheet Team column:**
- POST → `POST` (claims, portal, webhooks)
- MINT → `MINT` (merchant integrations, order ingestion)
- PEX → `PEX` (contracts, proration, activation)
- EMIT or Invoicing → `SE` (SFTP, invoice automation)
- RAD → `RAD` (reporting, data)
- XAD → `XAD`

**Description — always use this exact structure:**

```
**Executive Summary:** {1-2 sentences. What is being built and why — written for leadership who have 30 seconds on a Monday review call. Plain language, no jargon.}

---

**User Story**

As a {persona}, {I need / I should be able to} {capability} so that {outcome}.

---

**Context**

- {Bullet 1 — merchant-specific context, OMS, go-live phase, stakeholders}
- {Bullet 2}
- {Bullet 3-6 as needed — open questions, phase dependencies, prior art tickets}

---

**Acceptance Criteria**

- {Testable, specific criterion}
- {Another criterion}
- {For placeholder tickets: "TBD — to be defined as [X] is resolved"}

---

**Other Information / Considerations**

- {Constraints, dependencies, related ticket numbers, phase notes}
```

**Portal terminology — never mix these up:**
- MyExtend = Extend's branded customer-facing portal
- Merchant Portal = associate-facing claim intake portal
- These are different products serving different users

**Consistency rule:** Every ticket gets the same five sections in the same order. No new headers, no skipped sections, no deviation — even for simple or placeholder tickets.

### Step 6 — Present Drafts for Review

Show a summary table first:

| Row | Summary | Priority | XFNL | Request Type |
|---|---|---|---|---|

Then show each full draft with metadata + description. Ask for approval and accept targeted feedback on individual tickets before creating anything.

### Step 7 — Create Tickets

Once approved, create all tickets in parallel using `createJiraIssue` with `contentFormat: markdown`, parented to the confirmed epic. Report each ticket number as confirmed.

### Step 8 — Output HYPERLINK Formulas

Output a plain list for the sheet Ticket column, in sheet row order:

```
=HYPERLINK("https://helloextend.atlassian.net/browse/CCERP-XXX","CCERP-XXX")
```

- One formula per line
- Blank line for any skipped rows so the user can copy-paste the full list starting from the first data row
- Label which row number each formula corresponds to
- If rows need to be pasted in two batches due to a gap, call that out clearly

---

## Quality Checklist

Before presenting any draft, verify:
- [ ] Parent epic confirmed before any child Stories are drafted or created
- [ ] Every ticket has an Executive Summary
- [ ] All five description sections are present and in order
- [ ] Portal terminology is correct (MyExtend vs. Merchant Portal)
- [ ] Summary is prefixed with merchant name
- [ ] Priority matches the sheet
- [ ] XFNL team is set correctly
- [ ] Request Type is appropriate for the ticket type
- [ ] No duplicate tickets created for rows that already have a Jira ticket
- [ ] Parent epic key written to project MD after confirmation
- [ ] Merchant Go Live Blocker set on every Story (Yes/No — ask if unsure)

---

## Step 9 — Write Back to the Merchant Project File

After tickets are created and confirmed, silently update the merchant's project file at `{skill_base_dir}/../../outputs/memory/projects/{merchant}.md`.

- **CCERP Parent Epic**: confirm the epic key is stored (already done in Step 2, but verify it's present). Format: `CCERP Parent Epic: CCERP-XXX`
- **Open Tickets**: add each newly created CCERP Story with its ticket number, summary, and URL. Format: `- [CCERP-XXX](https://helloextend.atlassian.net/browse/CCERP-XXX) — [one-line description of what this ticket covers]`
- **Key Decisions**: if any decisions about requirement scope, phase assignment, or go-live blocker status were confirmed during the review (e.g., "CCERP-586 confirmed Phase 1 go-live blocker", "SSO requirement deferred to Phase 2"), append to `## Key Decisions`. Format: `- **[MM/DD/YY]:** [what was decided]`. Only add net-new decisions. **If this section does not exist, create it** before `## Already Resolved`.

After updating, note at the end of your output:
`Project file updated: [brief summary — e.g., "3 CCERP tickets added to Open Tickets, parent epic CCERP-576 confirmed"]`
