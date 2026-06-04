---
name: agenda-builder
description: >
  Build a fully researched meeting agenda on demand for any merchant implementation call and
  output it directly in the conversation. Use this skill whenever the user says things like
  "build me an agenda for [merchant]", "create an agenda", "prep an agenda for my [merchant]
  call", "what should we cover in the [merchant] meeting", "ad hoc agenda for [merchant]",
  or any variation of wanting a pre-meeting agenda produced right now. Also use it when the
  user just says "build an agenda" or "prep for my [merchant] call" without mentioning
  automation or docs. This skill does NOT insert into Google Docs and does NOT send a Slack
  DM — it simply outputs the agenda here in the conversation, ready to copy or use directly.
---

## ⚠️ Read This Skill First — Every Time

Even when resuming a session, continuing from a summary, or mid-task: read this skill in full before producing any agenda output. Do not rely on session memory or prior runs. The output format, filter rules, and required sections must be verified against this document every time.

Specific failure mode to avoid: building the agenda without reading this skill and skipping required sections (Last Call Recap, Decisions Needed Today, inline bolded action items, What's Next). These deviations will be caught immediately. Read first — always.

### 🚫 No-Skip Research Discipline (non-negotiable — applies in BOTH modes)

A "fresh" or recently-updated project file is NEVER a reason to skip live research. The project file is a baseline to reconcile against, not a substitute for the sweep. There is no "scoped pulse on a fresh baseline" shortcut — that is not a valid mode and must not be used.

- In Standard Mode, Steps 3, 3b, and 3c run in full, every time. This explicitly includes the **1:1 Slack DM reads** for each Extend-side contact on the project (Jordan, Lindsay, Morgan, Sara, etc.) and the **GDrive/Glean doc scan**. These are the highest-signal sources and the ones most often skipped — they are mandatory, not optional.
- The ONLY valid reason to mark a research-guide section as not-run is a hard tool error. "Already had enough context" and "baseline looks current" are never valid.
- Do NOT label output a "full sweep" in Research Sources unless every required section actually ran with evidence. If any section did not run, label it **"PARTIAL SWEEP"** and list the exact sources skipped and why. Never present partial research as complete.
- Apply the internal/external filter (Step 4) to EVERY item before it lands on an agenda: internal-only items (Extend QA, internal bug/validation tickets) belong on internal calls, not external merchant calls.

---

# Agenda Builder

Produce a structured, fully researched agenda for an upcoming merchant implementation meeting
and display it directly in the conversation. No Google Doc insertion, no Slack message — just
a clean, ready-to-use agenda.

---

## Mode Selection — Read Before Step 1

This skill operates in two modes. Detect which one applies before proceeding.

### Standard Mode
Use when the meeting is a **recurring cross-functional call** (e.g., weekly tech sync, implementation check-in, status review) with no specific topic scope. The whole merchant relationship is on the table. Run the full research sweep (Steps 3, 3b, 3c) and apply the full external/internal filter (Step 4).

**Trigger signals:** "build an agenda for [merchant]", "prep for my [merchant] weekly", meeting title is the recurring call name with no specific subject

### Focused Mode
Use when the meeting has a **stated topic or specific workstream** — the call exists to go deep on one subject, not sweep all open items. Research is targeted, not comprehensive. The agenda covers only what's relevant to that topic.

**Trigger signals:**
- User names a subject: "agenda for the training session", "agenda for the VGC / visa credit discussion", "agenda for the contracts review"
- Meeting title has a clear non-recurring subject (e.g., "PP: Visa Credit Discussion", "Retail/CX Training", "Claims Architecture Review")
- User explicitly says "focused", "tight", or "this is a pointed call"

**When in doubt:** if the meeting title contains a specific subject that is not the recurring weekly cadence name, use Focused Mode.

---

## Before you start

Project files are stored in the persistent workspace at `memory/projects/` — three levels up
from this skill's base directory. Construct the path as:
`{base_dir}/../../../Claude/memory/projects/{merchant}.md`

For example, if base_dir is `/sessions/abc123/mnt/.claude/skills/agenda-builder`, the
project files are at `/sessions/abc123/mnt/Claude/memory/projects/`.

Available merchants: `carparts.md`, `rooms-to-go.md`, `rugsusa.md`, `glamnetic.md`,
`505-mattress.md`, `caleres.md`, `city-beauty.md`, `warby-parker.md`

Also check: `{base_dir}/../../../Claude/memory/context/agenda-prep.md` if it exists — supplemental
context on how Samir prefers agendas structured.

Each project file contains: notes doc URL, GDrive subfolder link, Slack channel(s), key
contacts, open tickets, recurring topics, already-resolved items, and doc formatting rules.

**If a project file is missing or unreadable:** note it in the Research Sources section and
proceed using live sources only. Do not block on it.

---

## Step 1 — Clarify what's needed

If the user hasn't already specified, determine:
- **Which merchant** — if ambiguous, ask
- **Meeting type** — external (merchant is present) or internal (Extend-only)?
- **Topic/focus** — if this is a focused call, what is the specific subject?

Use calendar context, the meeting name, or conversation context to infer these if possible —
don't ask for things you can figure out yourself.

**Finding the calendar event — search broadly, not just by merchant name:**
Meeting titles rarely contain the full merchant name. Always run multiple calendar searches in
parallel to find the right event:
1. Merchant full name (e.g., "Warby Parker")
2. All aliases from the project file's "Also search as" section (e.g., "WP", "warbyparker")
3. If those return nothing, search by the merchant's email domain using the `participants`
   filter (e.g., `participants: ["@warbyparker.com"]`) — this catches meetings where the
   merchant name doesn't appear in the title at all but merchant attendees are on the invite
4. As a last resort, pull all events for the target date and scan for any meeting with
   merchant-domain attendees

The attendee list from the calendar invite is the authoritative source for the Attendees
field in the agenda output. Never fall back to guessing from the project file if the invite
was found.

## Step 2 — Meeting type detection

| Signal | Type |
|--------|------|
| Any non-@extend.com attendee | **External** — merchant is present |
| All @extend.com + "internal" in title + matching project file | **Internal** — Extend-only |

---

## Step 3 — Research (Standard Mode only)

*Skip to Step 3F if you are in Focused Mode.*

Read `research-guide.md` at `{skill_base_dir}/../../../Claude/memory/research-guide.md` and follow **Sections 0b through 7** in full before building the agenda. Complete all research before writing any output — having enough context from one source never justifies skipping another.

**Agenda-specific overrides and notes:**

- **Target meeting lookup (Step 1) is separate from the guide's calendar sweep.** Step 1 finds the specific meeting and its attendee list. The guide's Section 2 provides broader upcoming/recent meeting context. Both run independently.
- **Recurring agenda items:** Scan the Agenda sections of the last 2–3 notes doc entries for standing topics that appear week-over-week (e.g., "SIT/UAT status", "Open engineering items"). These belong on the agenda even without an explicit open question — their recurrence is the signal.
- **Jira output split:** Split tickets into Open and Recently Completed (last 2 weeks). Always include recently completed — they provide useful "what just shipped" context for both meeting types.
- **Recording Link field:** Note subject + date of any Chorus or Gemini email found — use it to populate the Recording Link field in the agenda output.
- **Notes doc subagent instructions — return everything, not just labeled action items:** When spawning a subagent to read the implementation notes doc, the prompt must explicitly instruct it to return EVERY item that looks like an open question, unresolved discussion point, embedded action, or follow-up — including items that appear as sub-bullets or question lines inside another section's notes. Do not rely only on clearly labeled `→ Action:` lines. Summaries naturally compress and drop embedded items; the subagent must be instructed to surface them all.

**Mandatory last-call action item inventory (run before writing the agenda):**

After completing the notes doc research, produce an explicit inventory of every action item and open question from the last 2 meeting entries. For each item:
- Mark ✅ confirmed closed — with the evidence (email, Slack message, Jira status change) that confirms it's done
- Mark ⏳ still open — and ensure it appears on today's agenda as its own item or sub-bullet

No item from the last call is allowed to disappear silently. If you cannot confirm it closed and it's not on the agenda, that is a gap. Every named action item — including those embedded as question lines or discussion sub-bullets inside another section, not just top-level `→ Action:` items — must be accounted for.

**SE open action item check (mandatory):** The solution engineer (e.g., Jordan Bell for CarParts) regularly carries technical action items that are embedded as sub-bullets inside other agenda sections rather than surfaced as top-level items. Before finalizing the agenda, explicitly ask: are there any open action items assigned to the SE from the last 2 meeting entries that haven't been confirmed closed? If yes, they must appear on the agenda.

---

## Step 3b — Pre-Call Pulse (Standard Mode only — run immediately before writing output)

After completing the full research sweep, run a same-day check for very recent activity
that could change what's on the agenda or how it's framed:

- **Slack — named channels:** Search the merchant's external channel and internal channel for
  messages sent in the last 4 hours. Also check DMs with key Extend team members joining the call.
  Look for: resolved blockers, new merchant asks, last-minute updates or decisions.

- **Slack — alias keyword sweep (critical — do not skip):** In addition to named channels,
  run a keyword search across ALL accessible Slack channels using each alias from the project
  file's "Also search as" section (e.g., `pton`, `onepeloton`, `warbyparker`). Search the last
  48 hours. This catches internal conversations about the merchant that happen in non-merchant-named
  channels — these are a common source of pre-call decisions that named-channel searches miss entirely.

- **Internal alignment DMs:** Search Samir's DMs with each Extend-side contact listed in
  the merchant's project file (Key Contacts → Extend side) for the merchant name and aliases
  in the last 72 hours. Do not use a fixed list — pull the names from the project file so
  this check is scoped to whoever is actually working this merchant. Internal pre-call
  alignment — decisions, answers, resolved blockers — almost always surfaces here first,
  before it makes it into a notes doc or named channel. Missing a DM thread is a direct
  cause of stale agenda items.

- **Gmail:** Run `from:[merchant domain] newer_than:1d` to catch any email sent today.

If anything material surfaces (a resolved action item, new merchant request, same-day decision),
update the agenda accordingly. Mark these findings in Research Sources with ⚡ to flag they
were caught in the pre-call pulse.

---

## Step 3c — Research Verification (Standard Mode only — mandatory before writing output)

Before writing the agenda, spawn a verification subagent using the Agent tool. Pass it your accumulated Research Sources entries and the following prompt:

> "You are a research audit agent. Read the research guide at `[research-guide path]` — specifically Section 9's required entries list. Then check the following Research Sources entries: [paste your accumulated entries]. Flag every required entry that is: (a) missing entirely, (b) collapsed into another entry, (c) marked ✅ but lacks specific evidence (e.g., 'Gemini notes found' without stating which method and which aliases ran), or (d) marked ❌ without a stated hard tool error. Return a gap list. If nothing is missing, say 'all clear'."

The research guide path is: `{base_dir}/../../../Claude/memory/research-guide.md`

**If the subagent flags any gaps:** fill them before proceeding. Do not write the agenda until the subagent returns all clear or every flagged gap has been addressed.

This step is non-negotiable. Its entire purpose is to catch skipped steps that would otherwise silently degrade the agenda quality.

---

## Step 3F — Focused Mode Research (Focused Mode only)

*Run this instead of Steps 3, 3b, and 3c. Do not run the full research guide sweep.*

The goal is targeted context — enough to go deep on this specific topic, nothing more.

**Step 3F-1: Read the project file**
Read the merchant project file in full. Extract only what is directly relevant to the meeting topic:
- Open action items touching this topic
- Key decisions already made about this topic
- Key contacts involved in this workstream
- Relevant Slack channels and notes doc URLs
- Any Jira tickets or recurring topics related to this subject

**Step 3F-2: Read the notes doc (topic-targeted)**
Open the implementation notes doc. Scan the last 3–4 meeting entries for any discussion of this specific topic. Do not summarize all open items — pull only what's relevant to the meeting focus.

**Step 3F-3: Run targeted Slack search**
Search the merchant's Slack channels (internal + external) for the topic keywords — last 2 weeks. Use the meeting title words, the subject matter keywords, and any names of attendees involved in this workstream. Look for: recent decisions, open questions, action items taken or pending.

**Step 3F-4: Run targeted Gmail search**
Search Gmail for the topic keywords — last 14 days. Include both directions (from merchant and from Extend team). Look for: email threads on this subject, documents shared, decisions made async.

**Step 3F-5: Run targeted Jira search (if applicable)**
If the topic involves engineering, product, or custom requirements, search Jira for tickets related to the topic. Use `text ~ "[topic keywords]"` scoped to relevant projects. Skip if the topic is purely operational (e.g., training logistics, legal contracts).

**Step 3F-6: Fetch any shared docs directly relevant to the topic**
If the calendar invite, Slack, or Gmail surfaced a specific doc (e.g., a training deck, a VGC proposal doc, a contract draft) — read it. This is often the most important source for a focused call.

**What to skip in Focused Mode:**
- Full research guide sweep (Sections 0b–7)
- Verification subagent
- Salesforce (unless the topic is commercial/deal structure)
- Zoom search
- GDrive broad folder sweep (only fetch specific docs surfaced in Step 3F-6)

**Pre-call pulse in Focused Mode:**
Run a minimal version — search Slack for the topic keywords in the last 4 hours only. Skip the broad channel sweep. Mark anything caught with ⚡.

---

## Step 4 — Apply the agenda filter (Standard Mode)

*Skip to Step 4F if you are in Focused Mode.*

Extract open/unresolved items from the last 4 weeks across all research sources.

**Hard rule: never include items listed in the merchant's "Already Resolved" section,
regardless of meeting type.**

**Hard rule — reconcile completed action items before writing the agenda:**
Before carrying any action item forward as "open," cross-check it against ALL research
sources — especially Slack DMs, email threads, and Jira comments — to confirm it is still
actually open. Action items get completed in DMs and email threads constantly, and those
completions almost never make it back into the notes doc. A specific pattern to watch for:
if a DM or email shows someone confirming they've done something ("done", "got access",
"sent it", "it's live", "just shared"), that action item is closed — do NOT carry it
forward as open even if it appeared as an open item in an older source. The failure mode
to avoid: reading a DM that says "nice, looks like you got GH access" and still writing
"confirm GitHub access" as an open next step in the agenda. Always ask: does the most
recent evidence across all sources confirm this is still pending, or has it been resolved?

**Additional rule — validate Chorus action items against Gmail before carrying forward:**
Chorus post-call recap emails often list action items by name (e.g., "Samir Ali to resend
updated project plan to Umesh Mundhe"). Before carrying any such item forward as open in
the agenda, run a targeted Gmail search to verify it hasn't already been executed via email:
`to:[recipient-email] [keyword] newer_than:14d`
For example, if the Chorus recap says "Samir to resend project plan to Umesh Mundhe", search
`to:umundhe@carparts.com project plan newer_than:14d`. A Chorus action item appearing in
a recap is not confirmation it's still open — it may have been completed via email within
days of the call. Only carry it forward as open if the Gmail search returns no completion
evidence. Apply this check to every named Chorus action item before it appears in the agenda.

**Active vs. passive mentions — this distinction matters:**
Recency alone is not enough to include a topic. Before adding any item, ask: is this topic
showing signs of *active, ongoing work*? Look for a concrete signal — an open question
still awaiting an answer, a decision still pending, a committed action not yet confirmed
complete, a blocker that's still in play. If a topic appears in a recent doc only as
background context ("we previously explored X"), as historical summary, or as a reference
to something that was once discussed and concluded, it is a passive mention — skip it even
if the source is within 4 weeks. The goal is to surface what needs to happen at this
meeting, not everything that has ever been touched.

### External meeting filter

**Include** — the merchant needs to be involved:
- Items where the merchant needs to take action
- Decisions requiring merchant input or sign-off
- Extend delivery timelines the merchant needs to plan around
- Active blockers on the merchant's side
- Open questions that require the merchant's answer or alignment

**Exclude** — Extend-internal only:
- Engineering tasks in progress that the merchant doesn't need to act on
- Internal QA/testing tasks
- Internal data analysis not yet ready to share
- Internal readiness checks before going to the merchant
- Ticket work with no merchant dependency

**Gray area**: If an internal task has a customer-facing outcome the merchant needs to plan
around, include only the outcome — not the internal task details.

**Jira tickets in external agendas — hard rules (non-negotiable):**
Jira tickets represent Extend-internal engineering work. In external agendas:

1. **Never reference a Jira ticket key, ID, or project prefix (e.g., MINT-6235, XAD-3838).** The merchant does not work in Jira and ticket keys have no meaning to them.
2. **Never give Extend engineering work its own agenda section just because the merchant asked a status question about it.** A merchant asking "how long does IN REVIEW take?" is a status question — answer it proactively async (in Slack or email), not as a dedicated agenda item.
3. **The bar for including an Extend delivery on an external agenda is: the merchant must need to take an action or block time.** Examples that pass: "WP needs to schedule UAT testing for this feature," "WP needs to plan their integration work around this delivery date." Examples that do not pass: "WP asked about the timeline," "WP will benefit from knowing this shipped."
4. **If an Extend delivery clears the bar, describe the merchant-facing outcome only** — no ticket names, no internal project labels, no status terminology ("IN REVIEW", "Ready For Work"). Write it as a plain Extend commitment: "Extend is building [feature]; WP will need to plan [testing/integration work] once it ships. Expected timeline: [X]."

**Example — wrong (for external agenda):**
> Extend Delivery Update: MINT-6235 and MINT-6236 in IN REVIEW — WP asked how long this takes

**Example — right (only if merchant has a concrete action):**
> Merchant Portal Enhancements: Testing Window (~3 min) — Extend has two merchant portal features shipping soon (agent/location ID filtering; post-claim redirect). Once shipped, WP should plan a testing pass in the demo store. Samir to share expected timeline.

If the merchant's need is only informational (status question, no action required), handle it in Slack or as a brief verbal mention on the call — not as a dedicated agenda item with its own time estimate.

**Permanent exclusion — bikes/ATVs (PRGMS-319):**
Product catalog expansion for bikes/ATVs (e.g., [PRGMS-319](https://helloextend.atlassian.net/browse/PRGMS-319)) is not Samir's
TAM workstream. Exclude from all agenda output — agenda items, Next Steps, Research Sources.
Do not reference this ticket or this work in any agenda regardless of recency.

### Internal meeting filter

Include everything in the external filter, plus:
- Extend engineering tasks and their current status
- Internal QA/testing tasks and findings
- Internal data analysis underway or ready to share
- Internal readiness checks
- Ticket work and blockers on the Extend side
- Action items owned by Extend team members
- Pre-call alignment topics: decisions the team needs to align on before the next merchant call

---

## Step 4F — Apply the agenda filter (Focused Mode)

**Primary rule: every agenda item must connect directly to the meeting topic.**

Before writing any item, ask: "Would this come up in a meeting specifically about [topic]?" If the answer is no — even if it's an important open item — leave it off. Samir knows those items exist; this meeting is not the place to surface them.

**What to include:**
- Open questions, decisions, or blockers that are *about this topic*
- Background context the attendees need to have a productive conversation on this topic
- Action items from prior meetings that were explicitly scoped to this topic
- Any new developments (Slack, email, docs) on this topic since the last time it was discussed

**What to exclude:**
- Open items from other workstreams, even if they're high priority
- General implementation status updates
- Items that are topically adjacent but not central (e.g., for a training/ops call, don't include COGS or contracts; for a VGC call, don't include OAuth setup)

**Last Call Recap in Focused Mode:**
Only include bullets directly relevant to this topic — prior decisions made, prior discussions that set context for today. Skip general implementation status. If there was no prior meeting specifically on this topic, say "First dedicated discussion on this topic" and briefly note any relevant prior context from the notes doc or Slack.

**Decisions Needed Today in Focused Mode:**
These should be tighter and more specific than in standard mode — ideally 1–2 concrete decisions that must come out of this specific meeting for work to proceed.

**Next Steps in Focused Mode:**
Only action items that are directly scoped to this topic. Do not carry forward open items from other workstreams.

---

## Step 4b — Pre-output ticket cross-check (Standard Mode — mandatory before writing any output)

*Run this immediately after completing Step 4 / the agenda filter, before writing a single
agenda bullet.*

**Purpose:** Catch open tickets that belong on this agenda but weren't surfaced by the
research sweep — most commonly tickets sitting in the project file that were overlooked
during Jira research.

**Steps:**

1. Pull the complete list of open tickets from the merchant's project file (`## Open Tickets` section).
2. For each ticket, identify its assigned owner (from the ticket description in the project file or from Jira research).
3. Cross-reference each owner against the confirmed attendee list for this meeting.
4. For each ticket where the owner is attending this call: check whether that ticket already appears in the drafted agenda.
5. If any such ticket is missing from the agenda — add it before proceeding to Step 5. Do not rationalize why it might not be needed. If the owner is on the call and the ticket is open, it belongs on the agenda.

**Also check:** Open Action Items from the project file whose owner is attending this call. Apply the same logic — if they're attending and the item is open, it must be on the agenda or explicitly accounted for.

**This step exists specifically to catch:** tickets assigned to attendees that are open in the project file but weren't caught during the Jira research sweep (e.g., a ticket that's been open for weeks, isn't in recent Jira results, but whose owner is sitting on this call today).

**In Focused Mode:** Run a scoped version — only check tickets and action items directly relevant to the meeting topic.

---

## Step 5 — Output the agenda in the conversation

Display the agenda directly here as clean markdown. Do not insert into Google Docs or send
a Slack DM. Do not wrap the agenda in a code block. Each top-level field (Recording Link,
Attendees, Agenda/Notes, Next Steps) must be separated by a blank line for clean rendering.
Do not add blank lines between individual agenda bullets.

⚠️ **JIRA LINKS — NON-NEGOTIABLE. CHECK BEFORE WRITING A SINGLE WORD OF OUTPUT.**

Every Jira ticket reference — in agenda items, context sentences, Next Steps, Research Sources,
anywhere in the output — must be a clickable hyperlink with a description. No bare ticket IDs.
No exceptions. If you are about to type a ticket key like ACE-243 or POST-7054 without a link,
stop and add the link first.

Required format for every ticket mention — link + parenthetical description, always:
`[TICKET-123](https://helloextend.atlassian.net/browse/TICKET-123) (2–4 word description)`

Examples:
- [ACE-243](https://helloextend.atlassian.net/browse/ACE-243) (claims resolve endpoint)
- [POST-7054](https://helloextend.atlassian.net/browse/POST-7054) (login loop blocker)
- [FUL-203](https://helloextend.atlassian.net/browse/FUL-203) (consolidated SO endpoint)

The parenthetical must appear on EVERY mention — including the main agenda item title bullet,
inline within context sentences, Next Steps lines, and Research Sources. This is especially
critical on the main bullet title, where Samir reads at a glance without diving into sub-bullets.

Never use a dash separator instead of parentheses. The parenthetical format is what makes the
description scannable mid-sentence. Dash format buries it.

If the same ticket is mentioned multiple times in the agenda, include the link and parenthetical
every time — never shorten to a bare ticket key on repeat mentions.
A ticket referenced without a link and parenthetical is an error that will be caught immediately.

**Output formatting for Google Docs copy-paste compatibility:**
- Font intent: Arial 11 throughout (normal text for all body content)
- Meeting date + title: output as plain bold text — `**[Month DD, YYYY] [Title of Meeting]**`. Do NOT use `# ` prefix and do NOT use `@` before the date. Both cause the entire pasted content to render as a heading style in Google Docs, overriding body formatting. Samir will manually set the heading style on the title line after pasting.
- Inline action items (`→ Action:` lines): bold — wrap the entire line in `**` including the arrow
- Agenda item titles (the top-level label for each agenda block): plain text, no `**` wrapping — e.g. `* The OOS Scenario — Framing the Problem` not `* **The OOS Scenario — Framing the Problem**`. This prevents the chat renderer from outputting them as heading elements that paste incorrectly.
- All other text: normal, no heading markers anywhere in the output

**Google Docs paste note:**
Everything pastes as normal Arial 11 text. Bold action items and italic questions paste correctly with no adjustments needed. Samir manually sets the title line to his preferred heading style after pasting — no other manual fixes required.

Use this exact format every time:

**[Month DD, YYYY] [Title of Meeting]**

**Recording Link:** [search Gmail for a Chorus (external) or Gemini (internal) email matching this meeting — paste URL if found, otherwise "TBD"]

**Attendees:**
* Merchant: [name, title] (list all non-@extend.com attendees)
* Extend: [name, title] (list all @extend.com attendees)

**📋 Last Call Recap**

Before the agenda items, include a 3–5 bullet recap of where things stand since the last meeting. This is a quick as-is status snapshot — not a history lesson. Keep each bullet to one sentence.

Primary source: the implementation notes doc (read the most recent meeting entry). Secondary: the merchant project file (Key Decisions, Open Action Items). Supplementary: Slack, Jira, Gmail if the notes doc is thin.

Format:
* ✅ [Decision or item confirmed complete since last call]
* ⏳ [Item still open — tag with "on today's agenda" if it will be discussed]
* ⏳ [Another open carry-forward — "on today's agenda" if applicable]

Do not fabricate context. If you cannot confirm something from a source, leave it out. Do not infer that a meeting occurred based on cadence alone — only report what the notes doc or research confirms actually happened.

**In Focused Mode:** scope the Last Call Recap to this topic only (see Step 4F).

**🎯 Decisions Needed Today**

List 2–3 items where a specific decision or answer must land on this call — not general discussion topics, but concrete choices that need to be made or confirmed by a named party. These are the must-resolve items going into the meeting.

Format:
* [Decision description] — [who needs to decide or answer]

Omit this section entirely if there are no hard decisions required today. Do not pad it with items that are just "good to discuss."

**Agenda/Notes:**
* [Agenda item title — short, scannable label, plain text no bold] (~X min)
  * **→ Action: [Owner] — [action description]**  ← first sub-bullet when an action exists
  * *What's the open question / decision needed — always italicized*
  * Background context with **key term** and **critical date or owner** bolded inline.
* [Agenda item 2 title] (~X min)
  * ...
* ...

Next Steps / Action Items:
* [Open action item from previous meetings/sources — owner if known]
* ...

**A few notes on each field:**

- **Date + Title**: Use the meeting date and name from the calendar invite or user's request.
  For internal meetings, prefix the title with `[Internal]` (e.g., `[Internal] CarParts Engineering Review`).

- **Recording Link**: Search Gmail for an automated Chorus or Gemini email tied to this
  specific meeting. Paste the link if found. If not yet available (meeting hasn't happened),
  put "TBD".

- **Attendees**: Pull from the calendar invite attendee list if available; otherwise use
  known contacts from the merchant project file. Separate into Merchant (non-@extend.com)
  and Extend (@extend.com).

  **Critical — always include the event organizer.** Calendar APIs return the organizer
  in a separate `organizer` field, not inside the attendees array. The organizer will be
  silently missing from the attendees list even if they are actively on the call. Always
  check the `organizer` field on the calendar event and add that person to the appropriate
  Extend or Merchant attendee line if they are not already present. This is a common failure
  mode — e.g. the Growth Strategy Manager who created the invite gets dropped entirely.

  **New attendees flag:** After listing all attendees, add a line if anyone is joining for the first time:
  `🆕 New to this call: [Name(s)] — [brief context, e.g. "Extend GSM joining for first time" or "WP OMS lead"]`
  Check by comparing the invite list against attendees from the last 2–3 meetings in the notes doc or calendar history.
  Omit this line if there are no first-time attendees.

- **Agenda/Notes**: Bulleted list of open topics, filtered per Step 4 or Step 4F.
  Each main bullet must use the following sub-bullet structure — do not use flat single bullets:

  **Detail level — use discernment, not exhaustiveness:**
  Each agenda item should give Samir enough context to run the conversation — what the topic is,
  what decision or answer is needed, and who owns it. Do NOT list every sub-issue, sub-question,
  or detail from the underlying source. If a reference document exists (a UAT notes doc, a Jira
  spec, a program requirements sheet, a test spreadsheet), link it inline in the background
  context sub-bullet and let the reader click through for detail. The agenda item itself stays
  high-level. Example: "Address agentic UAT issues before Aurelia sees demo — see [UAT Notes doc](url)"
  is correct; listing every individual issue from that doc as sub-bullets is not.
  The test: would listing this detail help Samir run the call, or would it bog him down?
  If it would bog him down, link the source and move on.

  ```
  * [Agenda item title — plain text, no bold] (~X min)
    * **→ Action: [Owner] — [action description]**  ← first sub-bullet when an action exists
    * *What's the open question / decision needed — always italicized*
    * Background context with **critical term** and **key date or owner** bolded inline.
  ```

  Every agenda item must have all three sub-bullets. If information for a sub-bullet is
  unavailable, write "Unknown — needs clarification on the call." Do not collapse items
  back into a single flat bullet under any circumstances.

  **One topic per agenda bullet — always:**
  If a single agenda bullet contains 2 or more distinct topics, questions, or items, split
  them into separate top-level agenda bullets — each with its own title, time estimate, and
  full sub-bullet structure. Never group two distinct topics into one bullet for brevity.
  The test: would a reader scanning the agenda miss that there are two separate things to
  discuss? If yes, split it. This applies regardless of how closely related the topics seem.

  **Sub-bullet order is fixed — do not vary it:**
  1. `→ Action` (bolded) — always first, so it's the first thing seen when scanning live
  2. Open question / decision needed — *always italicized*
  3. Background context — plain text with **key phrases bolded** inline

  If an agenda item has no action (e.g., a pure discussion or update item), start with the
  open question instead and omit the → Action sub-bullet entirely for that item.

  **Inline bold — key phrase rule:**
  Within the background context sub-bullet, bold 1–3 genuinely critical terms per sub-bullet:
  blockers, owners, hard deadlines, go/no-go decisions, or specific product names that carry
  real meaning. Do not bold for decoration — only bold what Samir needs to see at a glance
  while running the call. If no term is genuinely critical, leave the sub-bullet plain.

  **Italic question — always applied:**
  The open question sub-bullet is always italicized, no exceptions. This creates a clear
  three-level visual hierarchy: **bold action** → *italic question* → plain context. The
  italic is the signal: "this is the thing I need to answer on this call."

  **Nested sub-bullets for multi-point context:**
  Never chain 3 or more distinct points in a single sentence using semicolons. If a context
  sub-bullet contains 3+ distinct items (tradeoffs, steps, options, failure modes), break
  them into nested sub-bullets with a bold lead phrase per item:
  ```
      * **Lead phrase** — explanation
      * **Lead phrase** — explanation
      * **Lead phrase** — explanation
  ```
  Use this any time a sentence would otherwise read: "X; Y; Z" or "A, B, and C, each of which..."
  Two items in a sentence are fine — three or more always get nested bullets.

  **Time estimates:** Append `(~X min)` to every agenda item title. Base the estimate on
  complexity and the total meeting duration from the calendar invite — the sum of all item
  estimates should roughly equal the meeting length. Items with hard decisions get more time;
  informational items get less. Round to the nearest 5 minutes.

  **Distinct Jira ticket sub-bullets:** When multiple Jira tickets within a single agenda
  item each cover a distinct issue, failure mode, or workstream — break each into its own
  sub-bullet with its own Jira hyperlink and parenthetical description. Do not reference
  multiple ticket links inline within a single sentence. One ticket = one sub-bullet line
  (or one sentence). If tickets are closely related steps in the same work (e.g., sequential
  fixes), they may share a sentence only if combining them does not obscure what each one is.
  Every ticket on every sub-bullet still requires the `[TICKET](url) (description)` format.

  **Inline action items**: For any agenda item that has a corresponding action item in the
  Next Steps / Action Items section, include it as the FIRST sub-bullet within the agenda
  item block — not the last. This surfaces the action immediately when scanning live.
  Wrap the entire line in `**` so it renders bold:
  `* **→ Action: [Owner] — [action description]**`
  If a single agenda item has multiple corresponding action items, stack them as consecutive
  first sub-bullets before the open question and context. Do not omit this inline copy —
  the action item must appear both here and in the Next Steps section.

  **Action items that resolve during the call — mark them complete:**
  If an agenda item's action was answered or resolved during the meeting (confirmed by Becky,
  Ankita, or any attendee on the call), mark it as ✅ Done in the agenda rather than leaving
  it as an open next step. Do not carry a resolved item forward into Next Steps. The agenda
  is a live document — completed items should be visibly closed, not silently left open.

- **Next Steps / Action Items**: Pull open action items from all research sources (notes doc,
  Slack, Gmail, Chorus/Gemini). These are carry-forwards going *into* the call, not new ones.
  Include owner where known. Label this section `**Next Steps / Action Items:**` in the output.
  In Focused Mode, include only items scoped to this topic.

- **What's Next**: Add a `**What's Next**` section at the very end of the agenda (after Next Steps).
  List all upcoming internal and external meetings in the next 3 weeks from Calendar research.
  Include both Extend-internal meetings and external calls with the merchant — both tracks matter.
  Format each entry as:
  `* [Date, Day, Time TZ] — [Meeting name] ([Internal] or [External])`
  If no meetings are scheduled, write "No meetings scheduled in the next 3 weeks."
  Always include this section — never omit it.

After the agenda, always append a **Research Sources** section listing every source attempted
and its outcome. Use this exact format:

---
Research Sources:
✅ [Source name] — brief note on what was found
⚠️ [Source name] — partial (explain what was accessible vs. not)
❌ [Source name] — not accessed (brief reason: tool error / no results / skipped)

Sources to account for every time in Standard Mode: Merchant project file, Implementation notes doc,
Salesforce, GDrive merchant subfolder, Slack, Jira, Gmail (including Chorus + Gemini docs).

Sources to account for in Focused Mode: Merchant project file, Implementation notes doc (topic-targeted),
Slack (topic-targeted search), Gmail (topic-targeted search), relevant docs surfaced by topic,
Jira (if applicable to topic). Note the mode at the top of the Research Sources section:
`Mode: Focused — topic: [stated topic]`

This way Samir always knows exactly what informed the agenda and can flag anything that looks off.

## Step 6 — Update the merchant project file (MANDATORY — do not skip)

After outputting the agenda, silently update the merchant's project file to reflect what
was learned during research. This step is not optional — run it after every agenda output,
regardless of session length or how much changed. Do not ask for permission — just do it
and note what changed.

**Section discipline — read before touching anything:**

Sections fall into three categories. Never violate these rules — Samir manually edits the
project file and wholesale replacement silently destroys his edits.

| Section | Rule |
|---------|------|
| Key Decisions, Key Contacts, Zoom Docs, Slack channels, Already Resolved | **Append-only.** Only add net-new entries. Never overwrite or delete existing content. |
| Open Action Items, Open Tickets, Recurring Topics | **Live list.** Items are added when new, removed when confirmed closed. Removals must be explicit and evidence-backed — not inferred. |
| Last researched, Overview fields | **Replace in place.** |

---

**What to update:**

- **Last researched**: update to today's date in the Overview section.

- **Key Decisions**: for every decision surfaced during research — including decisions confirmed
  or made during meeting discussions (architecture choices, integration approach decisions, scope
  agreements) and not just top-level labeled decisions — append a corresponding entry.
  Format: `- **[MM/DD/YY]:** [what was decided]`. Decisions embedded as sub-bullets or question
  resolutions in the notes doc count — distill them to the decision itself, not the full discussion.
  Only add net-new decisions not already captured. Do not duplicate. Append-only — never overwrite.
  **If this section does not exist, create it** before `## Already Resolved`.

- **Open Action Items** — project MD is the authoritative baseline, live sources are the reconciliation layer:
  1. Read the current `## Open Action Items` section from the project file. This is your starting list.
  2. For each existing item, cross-check it against ALL live sources (notes doc, Slack, Gmail, Jira). If you find clear completion evidence — someone confirming "done", "sent it", "it's live", a closed Jira ticket, a Slack acknowledgment — that item is closed.
  3. **Closed items must be moved to `## Already Resolved`** — never silently dropped. Format the resolved entry as: `- [Action description] — resolved [MM/DD/YY] via [source]`.
  4. Items with no completion evidence carry forward verbatim. **Project MD framing wins — do not rewrite the description** unless you have specific new evidence to add (append a date-prefixed note if needed).
  5. Add net-new items found only in live sources (not already in project MD). Format: `- [Action description] — **[Owner]**`.
  6. **Conflict rule**: live source wins on completion status. Project MD wins on item framing and description. These two rules never conflict — one governs whether the item stays, the other governs how it's written.
  **If this section does not exist, create it** before `## Already Resolved`.

- **Recurring Topics**: add any net-new active topics surfaced during research that aren't already listed.
  Remove only items explicitly confirmed resolved — and move them to Already Resolved (see above).

- **Already Resolved**: archive section — items only move in, never out. When an action item or
  recurring topic is confirmed closed, move it here with resolution evidence. Format:
  `- [Item] — resolved [MM/DD/YY] via [source]`. Append-only — never delete existing entries.
  **If this section does not exist, create it** after `## Recurring Topics`.

- **Open Tickets**: sync with Jira results — add new tickets found (with hyperlink + description),
  remove tickets now showing Done (no need to archive these — Jira is the system of record).

- **Key Contacts**: add any new names/emails/titles discovered during research. Append-only —
  never remove or overwrite existing entries.

- **Slack channels**: if research surfaced active merchant discussion in a channel not already
  listed in the project file, add it. Append-only.

- **Zoom Docs**: if a Zoom notes doc was found during research, add it to the `## Zoom Docs`
  section with date, meeting title, and URL. Format:
  `- **[Date] — [Meeting Title]:** [URL] (file ID: \`[file_id]\`)`
  Append-only. **If this section does not exist, create it** after `## GDrive`.

---

**Write-back path**: The project file path is:
`{skill_base_dir}/../../../Claude/memory/projects/{merchant}.md`

For example, for CarParts:
`/sessions/[session-id]/mnt/Claude/memory/projects/carparts.md`

After updating, append a brief line below the Research Sources section:

Project file updated: [list what changed, e.g. "Last researched updated, 2 Key Decisions added, 1 action item closed → Already Resolved, 2 new action items added, 1 Recurring Topic removed → Already Resolved"]

If nothing changed, omit this line.
