# JD Expert v1: Split "Review and Submit" into Review → Submit

**Builds on:** your v1 **Review and Submit JD Update** topic, as built with *JD-Update-Feature-Build-Guide*, *JD-Review-Router-Build-Guide* and *JD-HR-Readiness-Followups-Guide*:
- the guards and title questions;
- **JD Final Review** → `Topic.F`;
- the title gate, `SummaryText` and `ReviewContext`;
- the follow-ups;
- the summary;
- the decision question with the router loop;
- the Submit / Make more changes / Cancel branches.

**Goal:**
- **Review JD Update** runs the final review and asks HR's follow-up questions as **plain questions with no buttons**, one at a time. It re-runs the review with the answers, shows the summary, then redirects to submission.
- **Submit JD Update** shows the decision buttons (**Submit to HR / Make more changes / Cancel this update**), runs the router for typed replies, and submits.
- A follow-up question never appears on the question node that has the Submit buttons.

**Build in:** DEV. Nothing in the flows or prompts changes.

---

## Contents
- [Part 0 — How it works](#part-0--how-it-works)
- [Part 1 — Before you start](#part-1--before-you-start)
- [Part 2 — Copy the topic](#part-2--copy-the-topic)
- [Part 3 — Build Submit JD Update (the copy)](#part-3--build-submit-jd-update-the-copy)
- [Part 4 — Trim Review JD Update (the original)](#part-4--trim-review-jd-update-the-original)
- [Part 5 — Instructions and descriptions](#part-5--instructions-and-descriptions)
- [Part 6 — Tests](#part-6--tests)
- [Appendix — Troubleshooting](#appendix--troubleshooting)

---

## Part 0 — How it works

**Before:** one topic did everything. Follow-up questions were put into `Topic.Ask`, which is the text of the decision question, so they appeared above the Submit / Make more changes / Cancel buttons.

**After:**
```
REVIEW JD UPDATE  (the orchestrator chooses it: "looks good", "I'm done", "submit")
  guards → title questions → ChangesText/SectionsChanged
  → JD Final Review → Topic.F → title gate → SummaryText → ReviewContext
  → Followups / NeedsInfo
       └─ NeedsInfo (first time only):
            intro message
            ┌─► [Plain question] follow-up #n   ← no buttons
            │     save the answer to Global.ManagerNotes
            └──── next follow-up (loop)
            then re-run JD Final Review with the answers
  → summary message
  → Go to another topic → SUBMIT JD UPDATE (passes the review data along)
  → End current topic

SUBMIT JD UPDATE  (redirect only)
  Topic.F = ParseJSON(FinalJSON)
  → Ask · Turns · QALog
  → [Question] {Topic.Ask}  buttons: Submit to HR · Make more changes · Cancel this update
       ├─ Submit to HR       → Save JD Update Request → reset → End all topics   (unchanged)
       ├─ Make more changes  → (unchanged)
       ├─ Cancel this update → (unchanged)
       └─ typed reply → JD Review Router loop  (unchanged, minus the follow-up parts)
```

**The shortcut:** you don't rebuild the decision question, the router loop or the Save flow mapping. You **copy the whole topic** with the code editor, then delete the review half from the copy and the submission half from the original. Every formula keeps working, because the copy gets the same variable names, now passed in as inputs.

---

## Part 1 — Before you start

### 1.1 Back up
**make.powerapps.com** → **DEV** → **Solutions** → your JD solution → **Export solution** → **Unmanaged** → **Export** → **Download**.

### 1.2 Check two names in Review and Submit JD Update
1. **The prompt output variable:** click the **JD Final Review** tool node and note its output. The readiness guide uses `Topic.Final`, read by the next node as `Topic.F = ParseJSON(Topic.Final.text)`. If yours is different, use your name wherever this guide writes `Topic.Final`.
2. **The decision question node:** the **Ask a question** node whose message is `{Topic.Ask}`, with the buttons **Submit to HR / Make more changes / Cancel this update**.

✅ **Part 1 done when:** the backup is downloaded and you know both names.

---

## Part 2 — Copy the topic

1. **Copilot Studio** → **DEV** → **Job Description Expert** → **Topics** → open **Review and Submit JD Update**.
2. Top right: **…** (More) → **Open code editor**.
3. Click inside the code → **Ctrl+A** → **Ctrl+C** → close the code editor (don't change anything).
4. **Topics** → **+ Add a topic** → **From blank**.
5. Top right: **…** → **Open code editor** → click inside → **Ctrl+A** → **Ctrl+V** (this replaces the blank topic's code with the copy).
6. **Save** (top right). If it asks to stay in the code editor, close it.
7. **Details** (top bar) → **Name:** `Submit JD Update` → **Save**.

You now have two identical topics. **Turn the copy off** for now (**Topics** list → toggle), so the orchestrator doesn't pick the copy while you edit it.

✅ **Part 2 done when:** a topic named **Submit JD Update** exists with the same nodes as the original, and it's off.

---

## Part 3 — Build Submit JD Update (the copy)

### 3.1 Trigger: redirect only
1. Open **Submit JD Update** → select the **Trigger** node → **…** → **Change trigger** → **It's redirected to**.
2. **Save.**

Submission can then only be reached through Review JD Update, so every submit always follows a fresh review.

### 3.2 Delete the review half
Delete these nodes, working **from the bottom of this group upward**: node → **…** → **Delete**. Deleting a condition also deletes its branches.
1. The **summary message** ("Here's your update request: {Topic.SummaryText} …").
2. The **NeedsInfo** condition (with its follow-up branch), plus the `InFollowups`, `NeedsInfo` and `Followups` Set nodes.
3. The **ReviewContext** and **SummaryText** Set nodes.
4. The **title-support gate** condition.
5. The **Topic.F** Set node and the **JD Final Review** tool node.
6. The **ChangesText** / **SectionsChanged** nodes.
7. The **title fallback questions** and the **no-changes guard**.

**Keep** these:
- the three loop Set nodes: `Ask` = `"What would you like to do?"`, `Turns` = `0`, `QALog` = `"None"`;
- the decision question;
- everything below it: the router branch and the Submit / Make more changes / Cancel branches.

Red errors will appear wherever the remaining nodes use variables you just deleted (`Topic.F`, `Topic.ChangesText`, …). The next step fixes them.

### 3.3 Add the inputs
**Details** → **Inputs**. Three inputs came with the copy: `IndirectConfirmations`, `ProposedTitle`, `TitleJustification`. Leave them. Add four more with **Create a new variable**, each **Text**, with **Should prompt user: Off**. They must be spelled **exactly** like the variables they replace:

| Name | Description |
|---|---|
| `FinalJSON` | Internal. The JD Final Review result, passed in by Review JD Update. |
| `ChangesText` | Internal. Passed in by Review JD Update. |
| `SectionsChanged` | Internal. Passed in by Review JD Update. |
| `ReviewContext` | Internal. Passed in by Review JD Update. |

**Save.** Most of the red errors disappear, because the remaining nodes now find `Topic.ChangesText`, `Topic.SectionsChanged` and `Topic.ReviewContext` again.

### 3.4 Recreate `Topic.F` at the top
1. Directly below the **Trigger**, above the `Ask` node: **+** → **Set a variable value** → **Create new** → name it `F` (Topic) → **Formula**:
   ```powerfx
   ParseJSON(Topic.FinalJSON)
   ```
2. **Save.** Every formula that used `Topic.F.…` (the Save flow mapping, the level and title fields) works again.

### 3.5 Remove the follow-up parts of the router
Follow-ups are now asked in Review JD Update, so the router no longer needs them:
1. **The router node's PendingQuestion input** (in the question's **All other conditions** branch): **Formula** → `"None"`.
2. **The hr_note branch:** if it has the follow-up fork (a condition on `Topic.InFollowups` that moves to the next follow-up), delete that condition and keep the normal path: save the note, send the answer, set `Ask`, **Go to step** → question.
3. **The Submit to HR branch:** if it has the soft gate (a condition that checks unanswered follow-ups before submitting), delete that condition and keep the normal submit path.

### 3.6 Fix any remaining red references
Open the topic's error list: the red badge at the top, or each red node. Fix them as follows:

| Error mentions | Fix |
|---|---|
| `Topic.Final.text` (e.g. in the Save flow's **FinalReviewJSON** input) | Replace with `Topic.FinalJSON` |
| `Topic.InFollowups`, `Topic.FUIndex`, `Topic.Followups` | Leftovers from 3.5. Delete the node or condition that uses them |
| Anything else | Tell me the variable name. It's probably another value Review created that needs to be passed in as an input |

**Save.** The topic should show no errors.

✅ **Part 3 done when:** Submit JD Update saves with no errors and starts with `F` → `Ask` → `Turns` → `QALog` → the decision question.

---

## Part 4 — Trim Review JD Update (the original)

### 4.1 Rename
Open **Review and Submit JD Update** → **Details** → **Name:** `Review JD Update` → **Save**. Keep its trigger, **The agent chooses**, and its description.

### 4.2 Delete the submission half
From the bottom up:
1. The **condition under the decision question**, with all its branches: Submit / Make more changes / Cancel and the router branch.
2. The **decision question** node.
3. The three loop Set nodes: `Ask`, `Turns`, `QALog`. Only Submit needs them now.
4. The `InFollowups` Set node, if you have one.

**Save.**

### 4.3 Rebuild the follow-up branch as plain questions
Open the **NeedsInfo** condition's **True** branch. **Keep:**
- `Global.FollowupsDone` = `true`
- `Topic.FUIndex` = `1`
- the intro message ("Before I put this together for HR, one quick question…")

**Delete:** the `Topic.InFollowups` = `true` node, the `Topic.Ask` = `First(Topic.Followups).Q` node, and the **Go to step** → question node.

Then, below the intro message, still inside the True branch:

1. **Loop head:** **+** → **Add a condition** → **…** → **Change to formula** →
   ```powerfx
   Topic.FUIndex <= CountRows(Filter(Topic.Followups, !IsBlank(Q)))
   ```
2. **Inside that condition's first branch**, in order:
   1. **Ask a question**:
      - **Message** → **{x}** → **Formula**:
        ```powerfx
        Index(Filter(Topic.Followups, !IsBlank(Q)), Topic.FUIndex).Q
        ```
      - **Identify:** **User's entire response**
      - **Save user response as:** **Create new** → `Topic.FUAnswer`
      - **…** → **Properties** → **Question behavior** → **Skip behavior:** **Ask every time**
   2. **Set a variable value** → `Global.ManagerNotes` →
      ```powerfx
      If(IsBlank(Global.ManagerNotes), "", Global.ManagerNotes & Char(10)) & "- " & Index(Filter(Topic.Followups, !IsBlank(Q)), Topic.FUIndex).Q & " Answer: " & Topic.FUAnswer
      ```
   3. **Set a variable value** → `Topic.FUIndex` → `Topic.FUIndex + 1`
   4. **Topic management** → **Go to step** → click the **loop head** condition from step 1.
3. **Below the loop head condition** (where its branches rejoin, still inside the NeedsInfo True branch): **Topic management** → **Go to step** → click the **JD Final Review** tool node.

This re-runs the review with the answers. JD Final Review already receives `Global.ManagerNotes` and `FollowupsDone` (readiness guide, Part 3). Because `FollowupsDone` is now true, the second run returns `readiness: "ready"`, `NeedsInfo` is false, and the topic continues to the summary.

The follow-ups are open questions with no buttons. The manager types an answer, or anything else they want.

### 4.4 Redirect to submission
Below the **summary message** (the last node now):
1. **+** → **Topic management** → **Go to another topic** → **Submit JD Update**.
2. Map its inputs (each **Formula**):

| Submit input | Value |
|---|---|
| FinalJSON | `Topic.Final.text` |
| ChangesText | `Topic.ChangesText` |
| SectionsChanged | `Topic.SectionsChanged` |
| ReviewContext | `Topic.ReviewContext` |
| IndirectConfirmations | `Topic.IndirectConfirmations` |
| ProposedTitle | `Topic.ProposedTitle` |
| TitleJustification | `Topic.TitleJustification` |

3. Below it: **+** → **Topic management** → **End current topic**.
4. **Save.**

> If **Submit JD Update** doesn't appear in the topic list while it's off, turn it on first (Part 4.5), then come back to this step.

### 4.5 Turn Submit JD Update on
**Topics** list → **Submit JD Update** → toggle **On**. It's redirect-only, so the orchestrator can't pick it on its own.

✅ **Part 4 done when:** Review JD Update saves with no errors and ends with summary → Go to Submit JD Update → End current topic.

---

## Part 5 — Instructions and descriptions

1. **Overview** → **Instructions**: anywhere the text says **Review and Submit JD Update**, change it to **Review JD Update**. Topics inserted with `/` update automatically; plain-text names don't. **Save.**
2. **Review JD Update** → **Trigger description:** keep it, and make sure it still covers "done", "looks good", "submit" and "send it to HR". The orchestrator should always come here, never to Submit directly.
3. **Submit JD Update:** no description is needed (redirect only).
4. **Publish** the agent.

---

## Part 6 — Tests

Start a **new test session** for each test.

| # | Do | Expect |
|---|---|---|
| T1 | A small update with nothing for HR to ask → "Looks good, submit it" | Review: summary → Submit: the decision question with buttons → **Submit to HR** → submitted, link, End all topics |
| T2 | An update that triggers follow-ups (e.g. a new client portfolio) → "I'm done" | Intro message → the first follow-up as a **plain question, no buttons** → answer → the next follow-up (if any) → summary → the decision question with buttons |
| T3 | After T2, check `Global.ManagerNotes` in **Variables** | It contains `- <question> Answer: <your answer>` for each follow-up |
| T4 | At the decision question, type **What happens after I submit?** | The router answers; the question is asked again; nothing is submitted |
| T5 | **Make more changes** → change a section → "done" | Review runs again with a fresh Final Review. Follow-ups are **not** asked again (`FollowupsDone`) |
| T6 | **Cancel this update** | The Cancel or Switch confirmation, as before |
| T7 | Submit T2, then open the HR email | Same as before: level block, indirect cards, HR summary, and manager notes with the follow-up answers |

---

## Appendix — Troubleshooting

| Problem | Fix |
|---|---|
| The code-editor paste won't save | Check you pasted into a **new, blank** topic and replaced all of its code (Ctrl+A first). If it still fails, send me the error |
| Errors remain in Submit after 3.3 | An input name doesn't exactly match (capitals count), or a variable from the review half is still used. See 3.6 |
| The same follow-up is asked over and over | 4.3: the question's **Skip behavior** must be **Ask every time**, and `FUIndex + 1` must sit above the Go to step |
| The follow-ups are asked again after **Make more changes** | `Global.FollowupsDone` must be set to `true` at the start of the NeedsInfo branch (keep that node) |
| The decision question shows a follow-up again | The `Topic.Ask` = follow-up node is still in Review, or the router's PendingQuestion isn't `"None"` (3.5) |
| Duplicate "Submitted!" message | Neither topic should have **outputs**. Submit still ends with **End all topics** |
| The orchestrator calls Submit directly | Its trigger must be **It's redirected to** (3.1) |
