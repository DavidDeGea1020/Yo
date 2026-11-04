# JD Expert: One Question per Turn (Question Queue) — Build Guide

**Builds on:** your current JD Expert v2 build:
- the JD Drafter prompt;
- Reset JD Update, Draft JD Update and Show JD Draft;
- the `NextQuestion` output pattern.

**Problem it fixes:** the drafter can return a question in `message.ask` and another question or confirmation in `message.notes` (often from a finding, such as "the work experience requirement overlaps with the Manager level"). The notes print first and the orchestrator then asks `NextQuestion`, so the manager gets two questions, and the second one sounds like the first was already answered.

**Goal:**
- Every question or confirmation the manager needs to answer becomes an item in **one queue** (`open_items`). This covers clarifications before drafting and finding-style confirmations after drafting.
- **Only the first item is asked each turn.** The rest wait, in order, and come up on later turns.
- Answers are captured in the manager's words (`resolved`), ready for submission.
- **Notes become statements only.** They're kept for hard stops and the brief indirect-field and FLSA mentions, whose questions already wait for submit by design.
- `NextQuestion` is set **in code** from the first open item, so even if the model wanders, only one question can reach the manager.

**Build in:** DEV.

> **Markers used in this guide**
> - `{InputName}` inside prompt text: delete that text, type `/`, and pick the input with that name.
> - Section numbers like **5.7** or **5.10** refer to the nodes in `JD-Expert-Conversational-Update-Guide.md` (Part 5, Draft JD Update).
> - If you applied the earlier one-question prompt edits (the STATUS "one question per turn" paragraph, the new MESSAGE notes/ask lines, calibration example 16), this guide replaces them. The Reply formula with the `?` filter stays.

---

## Contents

- [Part 0 — How it works](#part-0--how-it-works)
- [Part 1 — JD Drafter prompt](#part-1--jd-drafter-prompt)
- [Part 2 — Reset JD Update: three new globals](#part-2--reset-jd-update-three-new-globals)
- [Part 3 — Draft JD Update](#part-3--draft-jd-update)
- [Part 4 — Show JD Draft](#part-4--show-jd-draft)
- [Part 5 — Agent instructions (small edit)](#part-5--agent-instructions-small-edit)
- [Part 6 — Tests](#part-6--tests)
- [Appendix A — Troubleshooting](#appendix-a--troubleshooting)
- [Appendix B — What this sets up for submission](#appendix-b--what-this-sets-up-for-submission)

---

## Part 0 — How it works

### 0.1 The conversation

> **Manager:** I'd like to add to the purpose that this role also picks up food from the front office, and that the minimum experience is now 4 years.
> **Agent:** Happy to help update the Facilities Coordinator JD.
> **Agent:** What does picking up food from the front office involve? For example, collecting catering deliveries from the lobby and bringing them to meeting rooms, or something else?
>
> **Manager:** Catering deliveries from the lobby, brought to the conference rooms.
> **Agent:** Thanks, that's clear.
> **Agent:** On experience: our approved ranges are Minimum 1 Year, 2-4 Years and 5-7 Years, and 4 years isn't one of them. Do you want to move to 5-7 Years, or keep 2-4 Years and make it Required?
>
> **Manager:** 5-7 years, required.
> **Agent:** *(shows the draft: Principal Duties and Work Experience changes)*
> **Agent:** 5-7 years matches the Facilities Manager's experience range. Keep it, or adjust it to keep the two levels distinct?

One question per reply, in the order the manager raised things. The draft appears once nothing blocks it, and findings that need the manager's call are asked after the draft, one at a time.

### 0.2 The two kinds of item

| Kind | When | Effect on the draft |
|---|---|---|
| **clarify** | The drafter can't draft part of the request without an answer: an unclear task; an unconfirmed role, title, team, system or number; a value that isn't allowed (an unapproved experience range); position vs. person; an incumbent-only credential; unexplained role drift | No drafting while any clarify item is open (status `needs_info`) |
| **review** | The drafter drafted it, but the manager should confirm or adjust it: level drift, overlap with a neighboring level, scope creep (in-scope version drafted), title-type fit, duplicates, consistency gaps, a missing business reason, the FLSA review offer | The draft is shown; the review item is the question |

**Not queued** (statements in notes only): hard stops, indirect fields (People Management, Sales/Non-Sales, RM, NMLS) and FLSA triggers. Their questions are asked at submit, as today.

### 0.3 The pieces

```
Draft JD Update
  ├─ JD Drafter  ◄── OpenItems input (new): waiting items + answered items
  │     returns open_items[] + resolved[] (new keys)
  ├─ Status: forced to needs_info while any clarify item is open (code)
  ├─ Save queue every turn (new):
  │     Global.JDOpenItems   waiting items, as lines
  │     Global.JDResolved    answers, appended
  │     Global.JDNextItem    the one question to ask next
  ├─ Reply: intro + changes + statement-only notes
  └─ NextQuestion = JDNextItem, or message.ask when nothing is waiting (code)

Show JD Draft: NextQuestion = JDNextItem when something is waiting
Reset JD Update: clears the three new globals
```

**Why the queue lives in globals:** on `needs_info` turns, `JDDraft` isn't saved. Waiting questions must survive those turns, so they're saved separately, every turn, whatever the status.

### 0.4 Time
About **1.5–2.5 hours**:
- Part 1: about 45 minutes, plus editor tests
- Parts 2–5: about 45 minutes
- Part 6: about 30 minutes

> **Order matters.** After Part 1, Draft JD Update will show an error in the test pane (the prompt has a new input the node doesn't map yet). Test only in the prompt editor until Part 3 is done.

---

## Part 1 — JD Drafter prompt

### 1.1 Back up
1. Copilot Studio → JD Expert → **Tools** → **JD Drafter** → open the prompt.
2. Click into the **Instructions** box → **Ctrl+A** → **Ctrl+C** → paste into Notepad → save as `JD-Drafter-Instructions-before-queue.txt`.

### 1.2 Add the input
1. In the prompt editor: **+ Add content** → **Text**.
2. **Name:** `OpenItems`
3. **Sample value:**
   ```
   WAITING FOR AN ANSWER (oldest first):
   (none)
   ALREADY ANSWERED:
   (none)
   ```
4. Click outside the input to close it.

### 1.3 INPUTS section: reference the new input
1. Find the `=== INPUTS ===` section and the block that ends with `{CurrentDraft}`.
2. Click at the end of that block and press **Enter** twice.
3. Paste:
   ```text
   QUESTIONS FOR THE MANAGER (items waiting for an answer, oldest first, and items already answered):
   {OpenItems}
   ```
4. Select `{OpenItems}` → **Delete** → type `/` → pick **OpenItems**.

### 1.4 THE RULES THAT NEVER BEND: the FLSA offer
1. In the **Hard stop** bullet, find:
   ```text
   Offer once to record an FLSA review request with their concern (FLSA REVIEW).
   ```
2. Replace it with:
   ```text
   Offer once to record an FLSA review request: add a review item (about "FLSA review request") asking whether they'd like one and what their concern is (FLSA REVIEW).
   ```

### 1.5 TITLES: missing parts go through the queue
1. In `=== TITLES ===`, find:
   ```text
   Ask for whichever of the three parts is missing, one at a time, and help draft supporting section changes.
   ```
2. Replace it with:
   ```text
   Add whichever of the three parts is missing to open_items as clarify items, one per part, and help draft supporting section changes.
   ```

### 1.6 BUSINESS REASON: the missing reason goes through the queue
1. In `=== BUSINESS REASON ===`, find:
   ```text
   If it's still missing after your first draft, make it your "ask", once ("What's driving this change? HR needs the business reason."). Never ask twice in a row.
   ```
2. Replace it with:
   ```text
   If it's still missing after your first draft, add it once as a review item at the end of open_items ("What's driving this change? HR needs the business reason."). Remove it as soon as a justification appears.
   ```

### 1.7 Replace the whole HOW TO TALK ABOUT CONCERNS section
1. Select from `=== HOW TO TALK ABOUT CONCERNS (severity) ===` down to the line just above `=== STATUS ===`.
2. Paste:
   ```text
   === HOW TO TALK ABOUT CONCERNS (severity) ===
   Record every concern in findings (for HR). How the manager hears it depends on severity:
   - block: hard stops only. Leave that part out, say why in one sentence in notes, continue.
   - confirm: indirect fields and FLSA review triggers. One brief statement in notes, once. Their questions are asked at submit.
   - advise: level drift, overlap with a neighboring level, role drift, scope creep, title-type fit, duplicates, consistency gaps. Never a note: add it to open_items (clarify if you couldn't draft that part, otherwise review).
   - fix silently: style, format, verb choice. Just write it well.
   Never re-raise anything already answered (QUESTIONS FOR THE MANAGER). Never lecture.
   ```

### 1.8 Add the QUESTIONS FOR THE MANAGER section
1. Click at the start of the `=== STATUS ===` line → press **Enter** twice → move up into the blank space.
2. Paste:
   ```text
   === QUESTIONS FOR THE MANAGER (one per turn) ===
   Anything that needs the manager's answer is an open item. Ask exactly one per turn; the rest wait in open_items.
   Two kinds:
   - clarify: you can't draft part of the request without the answer: an unclear task; an unconfirmed role, title, team, system, or number; a value that isn't allowed (such as an experience range that isn't approved); position vs. person; an incumbent-only credential; role drift the manager hasn't explained; a missing part of a title request.
   - review: you drafted it, but the manager should confirm or adjust it: level drift, overlap with a neighboring level, scope creep (you drafted the in-scope version), title-type fit, duplicates, consistency gaps, a missing business reason, the FLSA review offer. Write one sentence of context, then one question.
   Not open items: indirect fields, FLSA triggers, and hard stops. Those stay as short statements in notes.
   Rules:
   1. Start from the items in QUESTIONS FOR THE MANAGER. Keep their order and wording unless something changed.
   2. If the LATEST REQUEST answers the first item (or says to skip it or leave it for HR), remove it from open_items, add it to resolved with the answer in the manager's words, and apply the answer.
   3. Add new items at the end. Clarify items always come before review items.
   4. Remove an item if the change it was about is dropped. Never re-add an item listed as already answered.
   5. If the LATEST REQUEST doesn't answer the first item, handle the request and keep that item first; it is asked again.
   6. Never ask a question that assumes another open item was answered.
   ```

### 1.9 Replace the whole STATUS section
1. Select from `=== STATUS ===` down to the line just above `=== MESSAGE`.
2. Paste:
   ```text
   === STATUS ===
   - "ready": you changed the draft: sections, title, justification, reasons, or FLSA review request (new or revised). Return the complete draft.
   - "needs_info": a clarify item is open. Don't change the draft this turn; set confirm_first to true.
   - "no_change": a question, a thank-you, something already covered, or an answer that keeps the draft as it is. Answer it in the message.
   - "not_feasible": not something a JD update can do (about a person, a restricted field, belongs to a different job). Explain briefly and offer the closest thing you can do.
   - "hard_stop": the whole request is a hard stop.
   For every status except "ready", return the CURRENT DRAFT's changes, indirect, title, and flsa unchanged. Return open_items and resolved on every status.
   While any clarify item is open, the status is "needs_info". Once every clarify item is answered, draft everything the manager has asked for in WHAT THE MANAGER HAS SAID, using the answers.
   ```

### 1.10 Replace the whole MESSAGE section
1. Select from `=== MESSAGE` down to the line just above `=== CALIBRATION EXAMPLES ===`.
2. Paste:
   ```text
   === MESSAGE (the manager reads this; the changes are displayed separately) ===
   - intro: one natural sentence that leads with the answer (mention the job title on the first draft). Never a question. No filler.
   - notes: zero to two short statements, only for block concerns and the brief indirect/FLSA mentions. A note is never a question and never repeats an open item.
   - ask: the question of the first item in open_items, exactly as written there. If open_items is empty: "Want to adjust anything, or is it ready to submit?" It is the only question in the whole message.
   Don't list or repeat the changes in the message. The line numbers in THE JOB are for your line plan only; never mention them to the manager (quote the duty instead). Warm, plain, expert, brief.
   ```

### 1.11 Calibration examples
1. If you added an example 16 for the earlier fix, delete it.
2. Click at the end of your last calibration example → **Enter** → paste (renumber if your last example isn't 15):
   ```text
   16. One message, two unclear parts ("add that they pick up food from the front office, and minimum experience is now 4 years": the task is unclear and 4 years isn't an approved range) → two clarify items in the order mentioned; needs_info; ask only the food question; notes []. Next turn, the food answer goes to resolved and the experience question becomes the ask. Draft once both are answered.
   17. Work Experience raised to a range a higher neighboring level also uses → draft it, and add a review item: "This matches the [that level's title] experience range. Keep it, or adjust it to keep the levels distinct?" It is the ask; nothing about it in notes.
   18. A review item is open and the manager says "make the new duty shorter" → revise the duty; keep the review item first and ask it again.
   ```

Your existing examples that say "advise" now follow 1.7 automatically, so you don't need to edit them.

### 1.12 OUTPUT section: three edits
1. Find the **confirm_first** line and replace it with:
   ```text
   - confirm_first: true when any clarify item is open (status is then needs_info); otherwise false.
   ```
2. Click at the end of the **justification** line → **Enter** → paste:
   ```text
   - open_items: every item still waiting for the manager's answer, in order (clarify first), each {"about": "short label", "kind": "clarify" or "review", "question": "one sentence of context, then one question"}. Return it on every status; [] when nothing is waiting.
   - resolved: items the LATEST REQUEST answered, each {"about": "the item's label", "answer": "the manager's answer in their words"}; otherwise [].
   ```
3. Find the **message** line and replace it with:
   ```text
   - message: intro, notes (each as {"note": "text"}), ask (the first open item's question, or the default next step).
   ```

### 1.13 Output format
1. **Output** (JSON) → **Edit format**.
2. In the example JSON, click at the end of the `"justification": "",` line → **Enter** → paste:
   ```json
   "open_items": [ { "about": "Work Experience overlap", "kind": "review", "question": "The 5-7 year requirement matches the Manager level's range. Keep it, or adjust it to keep the levels distinct?" } ],
   "resolved": [ { "about": "Food pickup", "answer": "Collects catering deliveries from the lobby and brings them to the conference rooms." } ],
   ```
3. Check that the line just above (`"justification": "",`) still ends with a comma and the pasted block ends with a comma before `"message"`.
4. **Apply**.

**Save** the prompt.

### 1.14 Test in the prompt editor

| # | Set inputs | Expect |
|---|---|---|
| 1 | **LatestRequest** and **RequestHistory** = `I'd like to add to the purpose that this role also picks up food from the front office, and that the minimum experience is now 4 years.` (RequestHistory with `- ` in front); **OpenItems** = the sample | `status` `needs_info`, `confirm_first` true; `open_items` = two clarify items, food first; `message.ask` = the food question only; `message.notes` = [] |
| 2 | **OpenItems:** put test 1's two items under WAITING, one per line, as `- [clarify] About: question`. **RequestHistory:** add `- Catering deliveries from the lobby, brought to the conference rooms.` **LatestRequest** = that answer | `resolved` has the food answer; `open_items` has only the experience item; `ask` = the experience question; still `needs_info` |
| 3 | **OpenItems:** experience under WAITING, `- Food pickup: Catering deliveries…` under ALREADY ANSWERED. Add to RequestHistory and set **LatestRequest** = `5-7 years, required` | `ready`; food task + Work Experience drafted; if **AdjacentLevels** has a level using 5-7, a review item that's also the `ask` |
| 4 | Paste test 3's output into **CurrentDraft**; keep its review item under WAITING; **LatestRequest** = `Make the new duty shorter` | Only that duty changes; the same review item is first and is the `ask` |
| 5 | Fresh inputs; **LatestRequest** = `Make this role exempt` | `hard_stop`; the hard stop is a statement in `notes`; the FLSA offer is the review item and the `ask` |

✅ **Part 1 done when:** all five behave as described.

---

## Part 2 — Reset JD Update: three new globals

### 2.1 Open the topic
**Topics** → **Reset JD Update**.

### 2.2 Add one Set variable node per global
Scroll to the last node (`JDFlsaRequest`). For each row below, in order:
1. Below the last node: **+** → **Variable management** → **Set a variable value**.
2. **Set variable** → **Create new** → click the new variable to open **Variable properties**:
   - **Variable name:** the name from the table.
   - **Usage:** **Global (any topic can access)**.
3. **To value** → **Formula** → `""` → **Insert**.

| # | Variable | Holds |
|---|---|---|
| 25 | `JDOpenItems` | Items waiting for an answer, one per line: `- [kind] about: question` |
| 26 | `JDResolved` | Answers so far, one per line: `- about: answer` |
| 27 | `JDNextItem` | The one question to ask next (blank when nothing is waiting) |

**Save.**

✅ **Part 2 done when:** `JDOpenItems`, `JDResolved` and `JDNextItem` appear in the variable picker of any topic.

---

## Part 3 — Draft JD Update

Open **Topics** → **Draft JD Update**. Keep the topic **On**.

### 3.1 Remove the old JD Drafter node
The prompt has a new input, so the node must be refreshed.
1. Find the **JD Drafter** node (5.6), between the **Log the request** node (5.5) and the **Status** node (5.7).
2. Note the node directly above it (the `Global.JDRequestLog` node). You'll add the new node right below it.
3. On the JD Drafter node: **…** → **Delete**.

> Nodes below that read `Topic.DraftOut` may show errors now. That's expected; they clear once 3.2 is done.

### 3.2 Add it back and map the inputs
1. Below the `Global.JDRequestLog` node: **+** → **Add a tool** → **JD Drafter**.
2. For each input: click the input field → **…** (or **{x}**) → **Formula** → paste → **Insert**.

| Input | Formula |
|---|---|
| JobContext | `Global.JDContext` |
| AdjacentLevels | `If(IsBlank(Global.JDRelatedLevels), "(none)", Global.JDRelatedLevels)` |
| CurrentDraft | `If(IsBlank(Global.JDDraft), "(none yet)", Global.JDDraft)` |
| RequestHistory | `Right(Global.JDRequestLog, 3000)` |
| LatestRequest | `Topic.UpdateRequest` |
| OpenItems | *(below)* |

OpenItems formula (one line; paste through Notepad):
```powerfx
"WAITING FOR AN ANSWER (oldest first):" & Char(10) & If(IsBlank(Global.JDOpenItems), "(none)", Global.JDOpenItems) & Char(10) & "ALREADY ANSWERED:" & Char(10) & If(IsBlank(Global.JDResolved), "(none)", Global.JDResolved)
```

3. **Outputs** → **Save output as** → **Select a variable** → pick the existing `Topic.DraftOut`. If it created a new variable instead, click it and rename it `DraftOut`.

### 3.3 Re-point the retry
1. Find the **Go to step** node inside the 5.8 retry condition (`Topic.Tries` = `1` branch).
2. Click it → click the new **JD Drafter** node.

### 3.4 Status: force needs_info while a clarify item is open
1. Click the **Set a variable value** node for `Topic.Status` (5.7, step 1).
2. Click the formula → select all → paste:
   ```powerfx
   With({d: ParseJSON(Topic.DraftOut.text)}, If(Boolean(d.confirm_first) || CountRows(Filter(Table(d.open_items), Text(ThisRecord.Value.kind) = "clarify")) > 0, "needs_info", Text(d.status)))
   ```
3. **Insert**.

If the drafter marks something "clarify" but still says `ready`, the draft isn't saved or shown, and only the question goes out.

### 3.5 Save the queue every turn
Place this block **below the 5.10 node** (`Global.JDHardStops`) and **above the 5.11 condition** (`Topic.Status = "ready" && Topic.PlanOK`).

1. **+** → **Add a condition** → **…** → **Change to formula** → paste:
   ```powerfx
   Topic.Status <> "ready" || Topic.PlanOK
   ```
   This skips saving only when the draft failed its plan check (the failed output may refer to a draft that wasn't built).
2. Inside its **first branch**, add three **Set a variable value** nodes, in order. For each: **Set variable** → pick the global → **To value** → **Formula** → paste → **Insert**.

**3.5a** `Global.JDOpenItems`:
```powerfx
Concat(Table(ParseJSON(Topic.DraftOut.text).open_items), "- [" & Text(ThisRecord.Value.kind) & "] " & Text(ThisRecord.Value.about) & ": " & Text(ThisRecord.Value.question), Char(10))
```

**3.5b** `Global.JDResolved`:
```powerfx
Global.JDResolved & Concat(Table(ParseJSON(Topic.DraftOut.text).resolved), "- " & Text(ThisRecord.Value.about) & ": " & Text(ThisRecord.Value.answer) & Char(10))
```

**3.5c** `Global.JDNextItem` (the first clarify item if any, otherwise the first item, otherwise blank):
```powerfx
With({o: Table(ParseJSON(Topic.DraftOut.text).open_items)}, If(CountRows(o) = 0, "", If(CountRows(Filter(o, Text(ThisRecord.Value.kind) = "clarify")) > 0, Text(First(Filter(o, Text(ThisRecord.Value.kind) = "clarify")).Value.question), Text(First(o).Value.question))))
```

3. Leave **All other conditions** empty.

### 3.6 Reply: notes are statements only
1. Find the **Set a variable value** node for `Topic.Reply` (the message body, without the ask).
2. If you already applied the `?` filter, skip to 3.7. Otherwise click the formula → select all → paste:
   ```powerfx
   With({d: ParseJSON(Topic.DraftOut.text)}, If(Topic.Status = "ready" && Not(Topic.PlanOK), "I couldn't build that draft cleanly.", Text(d.message.intro) & If(Topic.Status = "ready" && Not(IsBlank(Topic.ChangesShown)), Char(10) & Char(10) & Topic.ChangesShown, "") & If(Topic.Status = "ready" && CountRows(Table(d.unchanged)) > 0, Char(10) & Char(10) & "No change needed: " & Concat(Table(d.unchanged), Text(ThisRecord.Value.section), ", "), "") & With({ns: Filter(Table(d.message.notes), Not("?" in Text(ThisRecord.Value.note)))}, If(CountRows(ns) > 0, Char(10) & Char(10) & Concat(ns, "- " & Text(ThisRecord.Value.note), Char(10)), ""))))
   ```
3. **Insert**.

If your current Reply formula is different from this, replace only its notes part (the last `If(CountRows(Table(d.message.notes)) > 0, …)`) with:
```powerfx
With({ns: Filter(Table(d.message.notes), Not("?" in Text(ThisRecord.Value.note)))}, If(CountRows(ns) > 0, Char(10) & Char(10) & Concat(ns, "- " & Text(ThisRecord.Value.note), Char(10)), ""))
```

### 3.7 NextQuestion: the first open item, set in code
1. Find the **Set a variable value** node at the end of the main path that sets `Topic.NextQuestion` (the topic's output). Leave the NextQuestion nodes in the guard branches as they are.
2. Click the formula → select all → paste:
   ```powerfx
   If(Topic.Status = "ready" && Not(Topic.PlanOK), "Could you say the change again, maybe a little differently?", If(IsBlank(Global.JDNextItem), Text(ParseJSON(Topic.DraftOut.text).message.ask), Global.JDNextItem))
   ```
3. **Insert**.

> If your plan-failure question is worded differently, keep your wording in place of `"Could you say the change again, maybe a little differently?"`.

**Save** the topic.

✅ **Part 3 done when:** the topic saves without errors, and the JD Drafter node shows all six inputs mapped.

---

## Part 4 — Show JD Draft

When the manager asks to see the draft, the next question should be the waiting item if there is one, so the queue isn't lost.

1. **Topics** → **Show JD Draft**.
2. Find the **Set a variable value** node that sets `Topic.NextQuestion` on the main path (after the draft view). Leave the no-job guard branch as it is.
3. Click the formula → select all → paste:
   ```powerfx
   If(Not(IsBlank(Global.JDNextItem)), Global.JDNextItem, IsBlank(Global.JDDraft), "What's changing about the job?", "Want to change anything else, or is it ready to submit?")
   ```
4. **Insert** → **Save**.

> If your two fallback questions are worded differently, keep your wording in place of the last two strings.

✅ **Part 4 done when:** the topic saves without errors.

---

## Part 5 — Agent instructions (small edit)

`NextQuestion` can now carry one sentence of context before the question ("5-7 years matches the Facilities Manager's range. Keep it, or adjust it?"). The orchestrator should keep both and not add its own lead-in.

1. **Overview** → **Instructions** → **Edit**.
2. Find the step that tells the agent to ask `NextQuestion` once (UPDATING A JD, step 5 in your current instructions).
3. At the end of that step, add:
   ```text
   If NextQuestion starts with a sentence of context, keep that sentence and the question. Don't add thanks, a lead-in, or a second question.
   ```
4. **Save**.

**Optional:** **Draft JD Update** → **Details** → **Outputs** → **NextQuestion** → **Description**. Add the same sentence at the end. Repeat for **Show JD Draft** and **Start JD Update**, if their NextQuestion descriptions are separate.

✅ **Part 5 done when:** the instructions are saved and still under about 8,000 characters.

---

## Part 6 — Tests

In the test pane, click **Start new test session** before each group. Keep **Variables** open on the globals `JDOpenItems`, `JDResolved` and `JDNextItem`.

### 6.1 The queue

| # | Say | Expect |
|---|---|---|
| Q1 | *(find and confirm the Facilities Coordinator job)* → I'd like to add to the purpose that this role also picks up food from the front office, and that the minimum experience is now 4 years. | **One** question (food). No bulleted question above it. `JDOpenItems` has two lines; `JDNextItem` = the food question; no draft shown |
| Q2 | Catering deliveries from the lobby, brought to the conference rooms. | **One** question (experience range). `JDResolved` has the food answer; still no draft |
| Q3 | 5-7 years, required. | The draft (Duties and/or Purpose + Work Experience). Then one question: a review item (for example an overlap with a higher level) or "ready to submit" |
| Q4 | *(if Q3 asked a review item)* Keep it. | No question about that item again; the next item, or "ready to submit"; `JDResolved` gains a line |
| Q5 | *(with a review item open)* Make the new duty shorter. | The duty is revised; the same review question is asked again |
| Q6 | *(with an item open)* What have I changed so far? | Show JD Draft lists the draft, then asks the waiting item |

### 6.2 Other paths

| # | Say | Expect |
|---|---|---|
| Q7 | Make this role exempt | Hard stop as a statement; the FLSA review offer as the only question |
| Q8 | Retitle it Senior Facilities Coordinator | One question for the first missing part (justification or supporting duties), never two |
| Q9 | They'll have sales goals now | A Sales note (statement); **no** sales question now (asked at submit) |
| Q10 | Scrap this update → Yes, discard → start a new update | `JDOpenItems`, `JDResolved` and `JDNextItem` are blank in the new update |
| Q11 | *(draft with no open items)* Looks good | "Ready to submit" path, no leftover question |

### 6.3 Tuning

| If you see | Fix it in |
|---|---|
| A question still appears in the bulleted notes | Check the 3.6 `?` filter; then MESSAGE (1.10) |
| A finding is a statement instead of a question | HOW TO TALK ABOUT CONCERNS (1.7): it's being treated as confirm, not advise. Add a calibration example for that kind of finding |
| The same answered question comes back | Check `JDResolved` holds the answer and the OpenItems mapping (3.2); then rule 4 in 1.8 |
| A waiting question disappears without being answered | Check the 3.5 condition and that `JDOpenItems` survives a `needs_info` turn; then rule 1 in 1.8 |
| The draft never appears | A clarify item is stuck open: check `JDOpenItems`. If the item should be a review, tighten its kind in 1.8 |
| The orchestrator adds "Thanks —" or a second question | Part 5 |

✅ **Part 6 done when:** Q1–Q11 pass.

---

## Appendix A — Troubleshooting

| Problem | Cause / fix |
|---|---|
| "missing the following properties: structuredOutput" or "open_items" | The JD Drafter node wasn't refreshed after the output format changed. Redo 3.1–3.3 |
| Every test-pane message fails after Part 1 | The new OpenItems input isn't mapped yet. Finish Part 3 |
| `ParseJSON` error on `open_items` or `resolved` | 1.13 wasn't applied, or a comma is missing in the example JSON. Reopen **Edit format**, fix it, **Apply**, then redo 3.1–3.3 |
| "Unexpected character" when pasting a formula | Curly quotes or non-breaking spaces from the rendered guide. Paste through Notepad |
| The retry loop never reruns the drafter | 3.3 wasn't done; the Go to step still points to the deleted node |
| `JDNextItem` is set but the orchestrator asks something else | 3.7 formula not applied, or the NextQuestion description/instructions (Part 5) |
| The drafter takes noticeably longer | The OpenItems input is small, so check `JDRelatedLevels` size first (see the main guide's troubleshooting) |

---

## Appendix B — What this sets up for submission

- **Open items at submit:** Submit JD Update should check `Global.JDNextItem` first. If it isn't blank, ask that item (through Draft JD Update) before the submit questions, one at a time.
- **Answers for HR:** `JDResolved` holds every answer in the manager's words. It's a good source for the pending **ManagerNotes** decision: for example, tag answers to level items as `[Level] …` so they become the level response in the HR email.
- **Indirect fields:** unchanged. Their questions (`indirect[].question`) are still asked at submit, one at a time, as planned.
- **Reset:** Reset JD Update already clears the three new globals, so Submit's final Reset covers them.
