# JD Expert: Hybrid Replies — Build Guide

**Builds on:** your current JD Expert v2 build with the question queue in place:
- the JD Drafter prompt (shortened version);
- Draft JD Update with the queue globals `JDOpenItems`, `JDResolved` and `JDNextItem`;
- Start JD Update with its `NextQuestion` pass-through (`Topic.DraftNext`);
- Show JD Draft.

**Goal:** the orchestrator writes the conversational parts of each reply naturally, while the drafted JD wording stays exact.
- **The topic** shows the drafted changes **word for word**, in one message, only when there are new changes to show.
- **The orchestrator** writes everything else in its own words: the lead-in when no changes are shown, the notes, and the one question.
- **One question per turn** stays enforced in code: the topic never sends a question, and `NextQuestion` still comes from the queue.

**Build in:** DEV.

> **Markers used in this guide**
> - Section numbers like **5.11** refer to the nodes in `JD-Expert-Conversational-Update-Guide.md` (Part 5, Draft JD Update) and **8.5** / **8.13** to Start JD Update (Part 8).
> - "Queue guide" means `JD-Expert-Question-Queue-Guide.md`.
> - Paste every formula through Notepad first (curly quotes and non-breaking spaces cause "Unexpected character").

---

## Contents

- [Part 0 — How it works](#part-0--how-it-works)
- [Part 1 — Back up and clean up](#part-1--back-up-and-clean-up)
- [Part 2 — JD Drafter prompt: the intro line](#part-2--jd-drafter-prompt-the-intro-line)
- [Part 3 — Draft JD Update: outputs](#part-3--draft-jd-update-outputs)
- [Part 4 — Draft JD Update: nodes](#part-4--draft-jd-update-nodes)
- [Part 5 — Start JD Update: pass the outputs through](#part-5--start-jd-update-pass-the-outputs-through)
- [Part 6 — Agent instructions](#part-6--agent-instructions)
- [Part 7 — Tests](#part-7--tests)
- [Appendix A — Troubleshooting](#appendix-a--troubleshooting)

---

## Part 0 — How it works

### 0.1 Message order

Topic messages are sent while the topic runs; the orchestrator always replies after the topic ends. So the order on every turn is: **topic message (if any) → orchestrator reply.**

The design uses that order. The first bubble is the draft and the second bubble is the conversation:

> **Manager:** 5-7 years, required.
>
> **Agent** *(topic message, word for word)*: For Facilities Coordinator, the food pickup becomes a new duty and the experience requirement moves to 5-7 Years:
>
> **Principal Duties**: Adds collecting catering deliveries.
> - Add: Collects catering deliveries from the lobby and delivers them to conference rooms.
>
> **Work Experience**: Raises the experience range.
> - Add: 5-7 years of facilities or office services experience. Required.
> - Remove: 2-4 years of facilities or office services experience. Preferred.
>
> No change needed: Education, Certifications/Licenses, KSAs
>
> **Agent** *(orchestrator, own words)*: 5-7 years is the same range the Facilities Manager uses. Want to keep it, or adjust it so the two levels stay distinct?

When there are no new changes to show (a question, an answer, a hard stop, a title or business-reason update), the topic sends **nothing** and the orchestrator writes the whole reply.

### 0.2 What each turn produces

| Outcome (new output) | Topic message | Orchestrator reply |
|---|---|---|
| `drafted`: new section changes | Intro + changes + "No change needed" (word for word) | Notes (statements) → NextQuestion. Never repeats the changes |
| `needs_info`: a clarify item is open | None | NextQuestion only |
| `updated`: ready, no section display (justification, reasons, title, FLSA request) | None | Intro → Notes → NextQuestion |
| `no_change`: question, answer, thanks | None | Intro (the answer) → NextQuestion |
| `not_feasible` / `hard_stop` | None | Intro (why) → Notes → NextQuestion |
| `plan_failed`: line plan failed twice | None | "I couldn't build that draft cleanly." → NextQuestion |

### 0.3 Design review: why each choice

| Decision | Why |
|---|---|
| The **drafter's intro** heads the changes message, not the orchestrator's | The orchestrator replies *after* the topic message, so its lead-in would land below the changes ("Here's the draft:" after the draft). The drafter writes the intro as a lead-in to the changes it just made, so it always matches them |
| Changes stay in a **topic message** | `JDProp*` and `JDChangesFull` are what HR receives. The manager must see exactly that text. Nothing the orchestrator writes is JD wording |
| **Notes and NextQuestion** go to the orchestrator | These are the conversational parts. The orchestrator can phrase them naturally without touching JD wording |
| **No question in any topic message** | Questions come only from `NextQuestion`, which the queue sets in code. That keeps one question per turn |
| **Outcome** is computed in code | It tells the orchestrator exactly what was already shown. It never has to guess whether to restate anything |
| An intro containing **"?"** is dropped (replaced by a fixed header) | Stops a stray question from slipping into the verbatim message |
| **Notes** keep the `?` filter and are blank on `needs_info` (except hard stops) | Notes can't become a second question or preview a pending item |
| "drafted" needs **touched sections** *and* text to show | A justification-only turn doesn't send a message with just an "Also in your draft" line |
| **Start JD Update** passes the outputs through | When Start redirects to Draft, Draft's outputs return to Start, not to the orchestrator |
| **Show JD Draft** is unchanged | It already shows the full recap as a message and sets `NextQuestion` from the queue |

### 0.4 Time
About **1–1.5 hours**:
- Parts 1–2: about 15 minutes
- Parts 3–5: about 45 minutes
- Parts 6–7: about 30 minutes

---

## Part 1 — Back up and clean up

### 1.1 Back up
1. **make.powerapps.com** → **DEV** → **Solutions** → your JD solution → **Export solution** → **Unmanaged** → **Export** → **Download**.
2. Copilot Studio → JD Expert → **Overview** → **Instructions** → copy everything into Notepad → save as `JD-Agent-Instructions-before-hybrid.txt`.

### 1.2 If you started the all-outputs version
Skip this if you didn't build any of it.
1. **Draft JD Update** → **Details** → **Outputs**: delete `DraftChanges` and `Unchanged` if they exist. Keep `Outcome`, `Intro` and `Notes` if they exist; Part 3 updates their descriptions.
2. Delete any **Set a variable value** nodes for `Topic.DraftChanges` or `Topic.Unchanged`.
3. **Start JD Update** → **Outputs**: delete `DraftChanges` and `Unchanged` if they exist, and any mappings or nodes for them.
4. **Save** both topics.

✅ **Part 1 done when:** the backup .zip and the instructions file are saved.

---

## Part 2 — JD Drafter prompt: the intro line

The intro now does one of two jobs: it heads the changes message word for word, or the orchestrator relays it in its own words.

1. **Tools** → **JD Drafter** → open the prompt.
2. Find the `- intro:` line in your MESSAGE section.
3. Replace the whole line with:
   ```text
   - intro: one natural sentence. Never a question. No filler. On a needs_info turn, intro is "". When you changed sections this turn, the intro is shown word for word directly above those changes, so write it as their lead-in and mention the job title on the first draft (for example "For Facilities Coordinator, the food pickup becomes a new duty and the experience requirement moves to 5-7 Years:"). Otherwise it's the answer the agent relays to the manager. Intro and notes never mention, hint at, or preview anything in open_items; only the ask does.
   ```
4. **Save.**

The output format didn't change, so you don't need to refresh the prompt node.

✅ **Part 2 done when:** the prompt saves, and an editor test with a first draft returns an intro ending in a colon that names the job.

---

## Part 3 — Draft JD Update: outputs

### 3.1 Create the three new outputs
1. **Topics** → **Draft JD Update** → **Details** → **Input and output** (or **Outputs**).
2. For each row: **Create a new variable** → **Name** → **Data type: String** → **Description** (paste) → done. If the output already exists (from 1.2), click it and replace the description.

| Name | Description (paste) |
|---|---|
| `Outcome` | What this turn did. drafted: the new changes were already shown to the user word for word; don't repeat, summarize or comment on them, and don't add a lead-in. needs_info: reply with NextQuestion only. updated, no_change, not_feasible, hard_stop, plan_failed: reply from Intro, then Notes, then NextQuestion. |
| `Intro` | What to tell the user first, in your own words. Keep its meaning and add no new facts. Blank means no lead-in. |
| `Notes` | Short statements the user should know, one per line. Mention each briefly as a statement, never as a question. Don't add opinions about level, title, pay type, overtime or classification. Blank means no notes. |

### 3.2 Update the NextQuestion description
1. In the same panel, click the existing **NextQuestion** output.
2. Replace its description with:
   ```text
   The one question to ask, last in your reply and only once. If it starts with a sentence of context, keep that sentence with the question. Never add another question.
   ```
   The old "Make it your entire reply" wording would conflict with Intro and Notes.

### 3.3 Update the trigger description
1. **Details** → **Trigger** → **Description**.
2. Replace "It shows its own results; don't repeat them." (or your current equivalent) with:
   ```text
   It shows any new drafted changes itself; build the rest of your reply from its outputs (Outcome, Intro, Notes, NextQuestion) and never repeat the changes.
   ```
3. **Save.**

✅ **Part 3 done when:** Draft JD Update has four outputs (`Outcome`, `Intro`, `Notes`, `NextQuestion`) with the descriptions above.

---

## Part 4 — Draft JD Update: nodes

### 4.1 Remove the old reply nodes
1. Scroll to the end of the main path, below the queue block (`JDOpenItems` / `JDResolved` / `JDNextItem`).
2. Delete the condition block that sends `Topic.Reply` (the "is not Blank" condition and its **Send a message**): **…** → **Delete**.
3. Leave the `Topic.Reply` **Set a variable value** node in place. 4.3 reuses it.

### 4.2 Outcome
1. Directly **below the queue block** (above the `Topic.Reply` node): **+** → **Variable management** → **Set a variable value**.
2. **Set variable** → `Topic.Outcome` → **To value** → **Formula** → paste → **Insert**:
   ```powerfx
   With({d: ParseJSON(Topic.DraftOut.text)}, If(Topic.Status = "ready" && Not(Topic.PlanOK), "plan_failed", Topic.Status = "ready" && CountRows(Table(d.touched)) > 0 && Not(IsBlank(Topic.ChangesShown)), "drafted", Topic.Status = "ready", "updated", Topic.Status))
   ```

### 4.3 The changes message text
1. Click the existing `Topic.Reply` node (now below `Topic.Outcome`).
2. Click the formula → **Ctrl+A** → paste → **Insert**:
   ```powerfx
   With({d: ParseJSON(Topic.DraftOut.text)}, With({i: Trim(Text(d.message.intro))}, If(IsBlank(i) || ("?" in i), "Here's the draft for **" & Global.JDJobTitle & "**:", i) & Char(10) & Char(10) & Topic.ChangesShown & If(CountRows(Table(d.unchanged)) > 0, Char(10) & Char(10) & "No change needed: " & Concat(Table(d.unchanged), Text(ThisRecord.Value.section), ", "), "")))
   ```

### 4.4 Send it only when there are new changes
1. Below the `Topic.Reply` node: **+** → **Add a condition** → **…** → **Change to formula** → paste:
   ```powerfx
   Topic.Outcome = "drafted"
   ```
2. Inside its first branch: **+** → **Send a message** → **{x}** → `Topic.Reply`.
3. Leave **All other conditions** empty.

### 4.5 Intro
1. **Below the condition block** (on the main line, after the branches rejoin): **+** → **Set a variable value**.
2. **Set variable** → `Topic.Intro` → **Formula** → paste → **Insert**:
   ```powerfx
   With({d: ParseJSON(Topic.DraftOut.text)}, With({i: Trim(Text(d.message.intro))}, If(Topic.Outcome = "plan_failed", "I couldn't build that draft cleanly.", Topic.Outcome = "drafted" || Topic.Outcome = "needs_info" || ("?" in i), "", i)))
   ```

### 4.6 Notes
1. Below 4.5: **+** → **Set a variable value**.
2. **Set variable** → `Topic.Notes` → **Formula** → paste → **Insert**:
   ```powerfx
   With({d: ParseJSON(Topic.DraftOut.text)}, If(Topic.Status = "needs_info" && CountRows(Table(d.hard_stops)) = 0, "", Concat(Filter(Table(d.message.notes), Not("?" in Text(ThisRecord.Value.note))), "- " & Text(ThisRecord.Value.note), Char(10))))
   ```

### 4.7 Check the order
On the main line, after the 5.11 block, the order is:
1. Queue block (`JDOpenItems`, `JDResolved`, `JDNextItem`)
2. `Topic.Outcome` (4.2)
3. `Topic.Reply` (4.3)
4. Condition `Topic.Outcome = "drafted"` → **Send a message** `Topic.Reply` (4.4)
5. `Topic.Intro` (4.5)
6. `Topic.Notes` (4.6)
7. `Topic.NextQuestion` (unchanged, from queue guide 3.7)

Leave the guard branches at the top of the topic (no job loaded) as they are. They set only `NextQuestion`, and the other outputs stay blank.

**Save.**

✅ **Part 4 done when:** the topic saves without errors and no message node sends anything except `Topic.Reply` inside the "drafted" branch.

---

## Part 5 — Start JD Update: pass the outputs through

When Start redirects to Draft, Draft's message still goes straight to the manager, but its outputs return to Start. Start passes them on, the same way it already passes `NextQuestion` through `Topic.DraftNext`.

### 5.1 Create Start's outputs
1. **Topics** → **Start JD Update** → **Details** → **Outputs**.
2. **Create a new variable** three times (**Data type: String**), with the **same names and descriptions as 3.1**: `Outcome`, `Intro`, `Notes`.
3. Click Start's existing **NextQuestion** output and replace its description with the 3.2 text.
4. **Save.**

### 5.2 Map the redirect in the 8.5 branch
1. Find the **Go to another topic → Draft JD Update** node in the **8.5** branch (same job + ChangeRequest).
2. Under the node's **Outputs**, for each Draft output: click it → **Create new** → enter the name:

| Draft output | Save as (new Topic variable) |
|---|---|
| Outcome | `DraftOutcome` |
| Intro | `DraftIntro` |
| Notes | `DraftNotes` |
| NextQuestion | `Topic.DraftNext` (already mapped; leave it) |

3. Find the node below that redirect that sets `Topic.NextQuestion` from `Topic.DraftNext`. Directly below it, add three **Set a variable value** nodes:

| Set variable | Formula |
|---|---|
| `Topic.Outcome` | `Topic.DraftOutcome` |
| `Topic.Intro` | `Topic.DraftIntro` |
| `Topic.Notes` | `Topic.DraftNotes` |

### 5.3 Map the redirect in the 8.13 branch
1. Find the **Go to another topic → Draft JD Update** node in the **8.13** hand-off.
2. Under its **Outputs**, map Outcome, Intro and Notes: click each → **Select a variable** → pick the `Topic.DraftOutcome`, `Topic.DraftIntro` and `Topic.DraftNotes` you created in 5.2. NextQuestion stays → `Topic.DraftNext`.
3. Below the node that sets `Topic.NextQuestion` from `Topic.DraftNext` in this branch, add the same three **Set a variable value** nodes as in 5.2 step 3.

> If both redirects share one `Topic.NextQuestion` node below the condition block, add the three Set nodes once, directly below that shared node, instead of in each branch.

**Save.**

✅ **Part 5 done when:** Start has four outputs, both redirects map all four, and the topic saves without errors.

---

## Part 6 — Agent instructions

1. **Overview** → **Instructions** → **Edit**.
2. Delete these sentences (or your wording of them) in **UPDATING A JD**:
   - "Your turn ends there: never add your own message after Start JD Update or Draft JD Update."
   - "It … shows its own results … Don't repeat, summarize, or comment on what it showed or asked."
   - From the queue guide (Part 5): "If NextQuestion starts with a sentence of context, keep that sentence and the question. Don't add thanks, a lead-in, or a second question."
3. Keep your NextQuestion step for **Show JD Draft** ("ask NextQuestion once…"); that topic still shows its own message.
4. Click at the end of the UPDATING A JD section → **Enter** twice → paste:
   ```text
   REPLYING AFTER START JD UPDATE OR DRAFT JD UPDATE
   These topics return Outcome, Intro, Notes and NextQuestion. Build your reply from them, in this order:
   1. Intro, if not blank: one or two sentences in your own words. Keep its meaning; add no new facts.
   2. Notes, if not blank: each as a short statement. Never turn a note into a question.
   3. NextQuestion: last, once, as the only question in your reply. If it starts with a sentence of context, keep it.
   When Outcome is drafted, the changes were already shown word for word: never repeat, summarize, reword, or comment on them, and don't add a lead-in such as "Here's the draft".
   When Outcome is needs_info, reply with NextQuestion only (a few words of lead-in at most), and don't mention anything else that's pending.
   Never write or rewrite JD wording, and never give your own view on level, title, pay type, overtime, or classification.
   ```
5. **Save.** Check you're still under about 8,000 characters.

✅ **Part 6 done when:** the instructions are saved and contain no rule that says "never add your own message" after Start or Draft.

---

## Part 7 — Tests

In the test pane, click **Start new test session** before each group. Keep the **activity map** and **Variables** (`JDChangesFull`, `JDNextItem`) open.

### 7.1 Message order and one question

| # | Say | Expect |
|---|---|---|
| H1 | *(find and confirm Facilities Coordinator)* → I'd like to add to the purpose that this role also picks up food from the front office, and that the minimum experience is now 4 years. | **No topic message.** One orchestrator reply: only the food question |
| H2 | Catering deliveries from the lobby, brought to the conference rooms. | **No topic message.** Only the experience question |
| H3 | 5-7 years, required. | **Bubble 1 (topic):** intro naming the job, the Add/Remove lines, "No change needed: …". **Bubble 2 (orchestrator):** any notes as statements, then one question. It doesn't restate the changes |
| H4 | *(after H3)* Keep it. | **No topic message.** A short natural acknowledgment, then "ready to submit" or the next waiting item |

### 7.2 Accuracy

| # | Check | Expect |
|---|---|---|
| H5 | After H3, compare bubble 1 with `Global.JDChangesFull` | The Add/Remove lines match character for character |
| H6 | *(new session)* Describe a change in the first message, then confirm the job (Start → Draft) | Same two-bubble shape as H3. If bubble 2 is generic or missing the question, check Part 5 mappings |
| H7 | We're doing this because of the new catering contract | Outcome `updated`; **no topic message**; the orchestrator confirms in one line and asks the next question |

### 7.3 Other paths

| # | Say | Expect |
|---|---|---|
| H8 | Make this role exempt | **No topic message.** The hard stop explained in a sentence or two (HR decides; Evelyn Martinez), then the FLSA offer as the only question |
| H9 | Drop the work experience change | **Bubble 1:** intro + the "Also in your draft" line. **Bubble 2:** one question |
| H10 | What have I changed so far? | Show JD Draft's recap (unchanged), then the waiting item or "ready to submit" |
| H11 | They'll have sales goals now | Bubble 1 with any section change; bubble 2 has the Sales note as a **statement**, no sales question |

### 7.4 Tuning

| If you see | Fix it in |
|---|---|
| The orchestrator restates or summarizes the changes after bubble 1 | Outcome description (3.1) and the REPLYING block (Part 6). Check that `Topic.Outcome` is "drafted" in Variables |
| Two questions in one reply | Check Intro/Notes in Variables. If they're clean, the orchestrator added one: tighten rule 3 in Part 6. If Notes has a question without "?", tighten the prompt's notes rule |
| Bubble 1's lead-in doesn't match the changes | The intro line (Part 2); add a calibration example showing a good lead-in |
| "Here's the draft for **…**:" appears often | The drafter's intro is blank or contains "?" (4.3 fallback). Check the intro line (Part 2) |
| No bubble 1 on a real draft | `Topic.Outcome` isn't "drafted". Check `touched` in the drafter output and `Topic.ChangesShown` |
| The orchestrator gives its own opinion on level or classification | The last sentence of the REPLYING block (Part 6) and the Notes description (3.1) |

✅ **Part 7 done when:** H1–H11 pass.

---

## Appendix A — Troubleshooting

| Problem | Cause / fix |
|---|---|
| Start's redirect shows "output not mapped" or the reply after Start is generic | Part 5: each redirect must map Outcome, Intro, Notes and NextQuestion |
| "Assigned String, expected Boolean" (or similar) on an output | The variable was created with the wrong type. Delete it and recreate it as **String** |
| Bubble 1 appears on a needs_info turn | The 4.4 condition isn't `Topic.Outcome = "drafted"`, or the Send message is outside the branch |
| The orchestrator says "Here's the draft" after bubble 1 | Outcome description (3.1) and the "When Outcome is drafted" line (Part 6) |
| Plan-failure turns sound like success | Check `Topic.Intro` = "I couldn't build that draft cleanly." on that turn (4.5); keep the REPLYING rule "keep its meaning" |
| Drafter timeouts return | Unrelated to this change: the orchestrator's reply is short. Check `JDRelatedLevels` size and the RequestHistory length (`Right(…, 3000)`) |
