---
name: go-live-check
description: >
  Run a targeted go-live readiness check for a named merchant. Researches all live sources
  (Slack, Gmail, GDrive, Jira, Calendar, Salesforce), determines the merchant's implementation
  dimensions (technical approach, channel, product), maps findings against the go-live checklist,
  and returns a ✅/⚠️/❌ scorecard with a clear verdict and named blockers. Use when Samir says
  "go-live check for [merchant]", "are we ready to go live with [merchant]", "is [merchant] ready
  to launch", "sanity check [merchant] before we go live", "what's still blocking [merchant]",
  "can we flip the switch on [merchant]", or any variation of wanting a pre-launch readiness
  assessment for a specific merchant.
---

# Go-Live Check Skill

## Purpose

Run a targeted go-live readiness check for a named merchant. Research all live sources,
determine the merchant's implementation dimensions (technical approach, channel, product),
map findings against the go-live checklist, and return a ✅/⚠️/❌ scorecard with clear
flags on anything that could block launch.

This skill is the operational entry point for go-live assessments — distinct from the
full deep-research sweep used by `add-merchant`. It's faster and output-focused: the goal
is a clear answer to "are we ready?" with named blockers, not a comprehensive history.

## Trigger Phrases

Use this skill when Samir says any of:
- "go-live check for [merchant]"
- "are we ready to go live with [merchant]"
- "go live readiness for [merchant]"
- "is [merchant] ready to launch"
- "sanity check [merchant] before we go live"
- "what's still blocking [merchant]"
- "can we flip the switch on [merchant]"

---

## Step 0 — Load Context

**Read all three files before doing anything else:**

1. Merchant project file: `memory/projects/[merchant].md`
   - If no project file exists, note it and proceed using live sources only
   - Extract: implementation type, channel, products in scope, contacts, aliases, GDrive folder IDs, Slack channels, open action items
2. Go-live checklist: `memory/go-live-checklist.md`
   - Identify which sections apply based on the merchant's dimensions (see below)
3. Research guide: `memory/research-guide.md`
   - This is the canonical reference for *how* to gather data — alias search discipline,
     DM reads, Chorus two-step, thread following, completed-action-check logic, tense/framing rules
   - Follow its methods exactly, with one override: use a **2-week lookback** instead of the
     guide's default 4-week window (go-live checks focus on current state, not full history)
   - The 5-day Gmail priority check from Section 5 Part 0 still runs at full depth regardless

**Determine the three implementation dimensions from the project file:**

| Dimension | Options |
|---|---|
| Technical approach | Platform (Shopify/BC/WC/SFCC) OR Custom/API (SDK + API/SFTP) |
| Channel | eComm OR In-Store OR Both |
| Product(s) | PP, SP, SOP — one or more |

**Applicable checklist sections by dimension:**

| Always apply | Section A (Commercial/Legal) + Section B (Universal Tech) + Section J (Enablement) + Section K (Go-Live Coordination) + Section L (Post-Launch) |
| Platform | Add Section C |
| Custom/API | Add Section D |
| eComm | Add Section E |
| In-Store | Add Section F |
| PP in scope | Add Section G |
| SP in scope | Add Section H |
| SOP in scope | Add Section I |

If dimensions are unclear from the project file, determine them from live sources during
research (Step 2) before scoring.

---

## Step 1 — Research

Follow `research-guide.md` Sections 0b–7 in full for all source-gathering mechanics:
alias discipline, DM reads, thread following, Chorus two-step, Gemini notes, Jira
project-scoped queries, completed-action check, tense/framing rules.

**Lookback override for this skill:** Use **2 weeks** everywhere the research guide says
4 weeks. The Gmail 5-day priority check (Section 5 Part 0) runs at full depth regardless.
Gmail broad search uses `newer_than:30d` as specified in the guide — do not shorten that.

Sources to cover: Salesforce, Calendar, GDrive (merchant folder + Gemini notes),
Slack (channel + DMs with all accumulated Extend contacts), Jira, Gmail (5-day priority
+ 30-day broad + Extend team threads + Chorus notes).

The accumulated contacts list still applies — expand it as each source is read and use
the full list for all Slack DM reads and Gmail team searches.

---

## Step 2 — Cross-Source Reconciliation

Apply research-guide.md Section 7 in full (staleness filter, active vs. passive mention,
completed-action check, Chorus action item Gmail verification).

Additionally apply these go-live-specific rules:

**Demo store vs. prod store:** If a test order is mentioned, confirm which store it hit.
Demo store test ≠ prod store test — these are two distinct checklist items. Mark ❌ if
only a demo store test is confirmed in any source.

**Named date rule:** "Before Memorial Day," "next week," "planning soon" do NOT satisfy
the go-live date checklist item. A specific calendar date is required. Mark ⚠️ if a
target window is given but no named date; mark ❌ if nothing specific was said.

---

## ⛔ Pre-Scoring Gate — Fill This Before Step 3

Do not score the checklist until every required source from research-guide.md Section 9 is confirmed run. Fill in each line. Any blank (not a tool error) = go back and run it now.

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
- Gmail Chorus: ✅ N results / ✅ 0 / ❌ tool error
- Gmail Gemini Method B: ✅ N results / ✅ 0 / ❌ tool error
- Zoom: ✅ N results / ✅ 0 / ❌ tool error
```

## Step 3 — Score the Checklist

For each applicable checklist section and item, assign:

| Score | Meaning |
|---|---|
| ✅ | Explicitly confirmed in a source — cite the source |
| ⚠️ | Partially confirmed or confirmed but with a caveat — explain what's missing |
| ❌ | Not done, or no signal found in any source |
| N/A | Genuinely not applicable to this merchant's implementation — explain why |

**Scoring rule:** No signal = ❌, not N/A. Only mark N/A when you can explain why the
item structurally doesn't apply (e.g., "carrier codes" is N/A for a STORIS in-store
merchant because there is no eComm shipping integration).

Always check the **Common Pre-Go-Live Miss List** explicitly — score every item on it.

---

## Step 4 — Output

Structure the output as follows:

---

### [Merchant Name] — Go-Live Readiness Check
**Date:** [today]
**Implementation:** [Technical approach] × [Channel] × [Product(s)]
**Checklist sections applied:** [A, B, D, F, G, J, K, L — or whichever apply]

---

#### 🚦 Readiness Verdict

One of:
- **READY** — all applicable items are ✅ or N/A; no blockers
- **READY WITH WATCH ITEMS** — technically ready but ⚠️ items exist that should be confirmed before/at go-live
- **NOT READY** — one or more ❌ blockers that must be resolved first

If not ready: list the specific ❌ blockers as the first thing after the verdict, clearly labeled **"Blockers:"**

---

#### Scorecard

Group by checklist section. Show each item and its score. For ✅ items, cite the source
(e.g., "confirmed in Slack #internal-505matts, May 1"). For ⚠️ and ❌ items, explain
what's missing and who owns it.

Only show N/A items in a collapsed group at the bottom — don't let them crowd the scorecard.

Example format:

```
**Section A — Commercial & Legal**
✅ MSA signed — Ironclad Complete (confirmed Salesforce, Apr 24)
✅ Sellers licenses — Key Fields confirmed (Salesforce, Apr 27)
❌ Go-live date — "before Memorial Day" given, no named calendar date confirmed
⚠️ Return policy window — 60-day referenced in notes; alignment with MFRM corporate 120-day policy unresolved (Slack, Apr 10 — Hank to explore internally, no follow-up found)

**Section B — Universal Technical**
✅ Prod store created and enabled — Jordan Bell confirmed (Slack, May 1)
❌ Test order in prod store — test order 91556773 confirmed against demo store (Apr 30); no prod store test order found in any source
✅ Order ID alignment — transactionId confirmed matching receipt (Apr 10 TKO notes)
...
```

---

#### Open Items Summary

A flat list of all ❌ and ⚠️ items only, with owner and recommended next action.
This is the "what to do next" section — make it actionable.

| # | Item | Score | Owner | Next Action |
|---|---|---|---|---|
| 1 | Test order in prod store | ❌ | Samir / Eric Withrow | Submit a test order against prod store and have Jordan Bell validate |
| 2 | Go-live date | ❌ | Hank | Follow up with merchant for a named date |
| ... | | | | |

---

#### Recent Activity (Last 2 Weeks)

Brief summary of what's happened recently — key decisions, milestones hit, notable emails
or Slack activity. 5–8 bullet points max. Helps Samir get up to speed quickly before a call.

---

#### Upcoming Meetings

Any calls scheduled in the next 3 weeks. If none, say so explicitly.

---

#### Research Sources

```
✅/⚠️/❌ [Source] — brief note
```

Cover: project file, go-live checklist, Salesforce, Calendar, GDrive notes doc, GDrive folder,
Gemini auto-notes, Slack channel, Slack DMs (list contacts), Jira, Gmail 5-day, Gmail 30-day, Gmail Chorus.

---

## Notes

- This skill is intentionally scoped to go-live assessment. If significant non-go-live issues
  surface during research (e.g., a major open CCERP item, a RAID risk), surface them briefly
  in a "Also Flagged" section at the end but don't let them crowd the scorecard.
- If the merchant has no project file, create one as part of this skill run using findings
  from live sources. Follow the same structure as existing project files in `memory/projects/`.

---

## Step 5 — Write Back to the Merchant Project File

After outputting the scorecard, silently update the merchant's project file at `{skill_base_dir}/../../../Claude/memory/projects/{merchant}.md`.

- **Last researched**: update to today's date
- **Open Action Items**: fully replace with all ❌ and ⚠️ checklist items that have a named owner and clear next action. Format: `- [Action description] — **[Owner]**`. Remove any items confirmed complete during research. **If this section does not exist, create it** before `## Already Resolved`.
- **Key Decisions**: append any go-live-related decisions confirmed during research (e.g., "confirmed go-live date", "test order in prod validated"). Format: `- **[MM/DD/YY]:** [what was decided]`. Only add net-new decisions not already captured. **If this section does not exist, create it** before `## Already Resolved`.
- **Open Tickets**: sync with Jira findings — add new open tickets, mark any now Done.
- **Recurring Topics**: add any net-new active themes surfaced (e.g., "prod store test order", "go-live date not named").

After updating, note at the end of your output:
`Project file updated: [brief summary of what changed]`
