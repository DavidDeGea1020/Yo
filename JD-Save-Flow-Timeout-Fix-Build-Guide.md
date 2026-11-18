# Save JD Update Request: Timeout Fix Build Guide

**What this fixes:** the agent gives up on a flow that hasn't answered within about **100 seconds**. Redline Purpose alone can take around 2.5 minutes on a long Purpose section, so the agent times out even though the flow finishes.

**How:** the flow saves the request and answers the agent straight away. Everything slow happens **after** the answer: redlining, building the Word doc, the HRBP email, and a new email that sends the manager their redlined doc. Steps placed after the response action keep running after the agent has its reply.

**The new order:**

| # | Step | Status |
|---|---|---|
| 1 | Trigger (inputs from the agent) | unchanged |
| 2 | Create item (saves the request, gets the Request ID) | moved up if needed |
| 3 | **Respond to the agent** (Request ID only) | moved up |
| 4 | **Scope `Build And Send`**, holding: | new |
| 4a | Redline steps (in parallel) | moved into the scope |
| 4b | Build the Word doc (populate template, create file) | moved into the scope |
| 4c | `Save Doc Link` (Update item) | new, only if needed |
| 4d | Send the HRBP review email (child flow) | moved into the scope |
| 4e | `Email Manager Their Doc` | new |
| 5 | **`If Build Failed`**: alert HR and tell the manager | new |

Parts:
1. Back up and map the flow
2. Save the request first
3. Respond to the agent early
4. Wrap the slow steps in a scope
5. Run the redlines in parallel (optional, but faster)
6. Email the manager their document
7. Failure alert
8. Update the Copilot topic
9. Tests
10. Troubleshooting

---

## Before you start

- Work in **DEV**. Move the solution to UAT/QA the usual way when it's done.
- Use the action names in this guide exactly, because expressions refer to them by name. Spaces turn into underscores: `Create item` → `outputs('Create_item')`.
- Where this guide says "your redline steps" or "your doc steps", use the names your flow already has. Only the new actions get new names.

---

## Part 1: Back up and map the flow

### 1.1 Make a backup copy
1. **Solutions** → your solution → **Cloud flows** → click **Save JD Update Request** to open its details page.
2. Top bar → **Save As** → **Flow name:** `Save JD Update Request - Backup` → **Save**. The copy is created off and outside the solution. Leave it off; it's only a fallback.

### 1.2 Map what's there
1. Open **Save JD Update Request** → **Edit**.
2. Write down, top to bottom, the name of every step. Group them like this:

| Group | Typical steps |
|---|---|
| **A. Inputs and setup** | Trigger, Initialize variable, Composes that read trigger inputs |
| **B. Save** | Create item |
| **C. Redlines** | Redline Purpose, Redline Principal Duties, and the rest |
| **D. Doc** | Populate a Microsoft Word template, Create file, any Compose that builds the doc name or link |
| **E. HRBP email** | Run a Child Flow → Send HR Review Email |
| **F. Response** | Respond to the agent (older flows: Return value(s) to Power Virtual Agents) |

3. Click the **response action** (group F). List each output and what fills it:
   - Outputs filled from the **trigger** or **Create item** (for example RequestID) → **keep**.
   - Outputs filled from groups **C** or **D** (doc link, doc name, download URL) → these **go** in Part 3.
4. Click **Create item**. Note whether any field is filled from group C or D (for example HRDocLink, HRDocName). If so, you'll need Part 2.2.

---

## Part 2: Save the request first

Create item must run before the response, so the agent can get the Request ID. It must not depend on the redlines or the doc.

### 2.1 If Create item already sits above the redlines and uses nothing from them
Skip to Part 3.

### 2.2 If Create item uses doc fields, or sits below the redlines
**Step 1: Move Create item up**
- **New designer:** drag **Create item** by its title bar and drop it on the **+** line directly **after the last group A step** (above the first redline step).
- **Classic designer:**
  1. On **Create item**: **⋯** → **Copy to my clipboard**.
  2. Click **+** after the last group A step → **Add an action** → **My clipboard** → click the copied action.
  3. Delete the original **Create item** (**⋯** → **Delete**).
  4. Rename the copy back to exactly `Create item`.

**Step 2: Clear the doc fields in Create item**
1. In **Create item**, clear every field filled from a redline or doc step (for example **HRDocLink**, **HRDocName**). Click into each one and delete the value.
2. **Save**. If Save shows an "invalid reference" error, another field still points at a redline or doc step. Clear it too, and note it for Step 3.

**Step 3: Add `Save Doc Link`**
This goes in after the doc is created. You'll place it inside the scope in Part 4, so for now just note the values. It's an **Update item** with:

| Field | Value |
|---|---|
| **Site Address / List Name** | same as Create item (**JD Update Requests**) |
| **Id** | **Dynamic content** → Create item → **ID** |
| **Title** | **Dynamic content** → Create item → **Title** |
| Each doc field you cleared in Step 2 | the same value it had in Create item before |
| **RequestStatus Value** | **Dynamic content** → Create item → **RequestStatus Value** |
| **TitleReviewRequested** | **Dynamic content** → Create item → **TitleReviewRequested** |
| Every other Yes/No or Choice column | Create item's value for that column |

> Update item resets any Yes/No or Choice column you leave blank back to its default. That's the TitleReviewRequested flip you hit before. Map every one of them to Create item's value.

---

## Part 3: Respond to the agent early

### 3.1 Remove the doc outputs
1. Click the **response action**.
2. On each output filled from a redline or doc step (doc link, doc name, download URL), click **⋯** → **Delete**.
3. Write down the output names you deleted. The topic uses them, and you'll clean that up in Part 8.
4. Check that **RequestID** (or whatever carries the request number) is still an output, filled from **Create item → ID**. If there isn't one:
   1. **+ Add an output** → **Text** → name `RequestID`.
   2. **Value:** **Expression** → `string(outputs('Create_item')?['body/ID'])` → **Add**.

### 3.2 Move the response directly under Create item
- **New designer:** drag the response action and drop it on the **+** line directly under **Create item**.
- **Classic designer:**
  1. **⋯** → **Copy to my clipboard** → **+** under **Create item** → **My clipboard** → click it.
  2. Delete the original response action.
  3. Rename the copy back to the exact original name.

It must sit on the **main line**: not inside a condition, scope or loop.

### 3.3 Save
**Save**. If you get an error saying the response "must be runAfter" a redline or doc step, an output still uses that step. Go back to 3.1.

---

## Part 4: Wrap the slow steps in a scope

The scope groups everything after the response, so one failure check (Part 7) covers all of it.

### 4.1 Add the scope
1. Click **+** directly under the **response action** → **Add an action** → search `Scope` → **Scope** (Control).
2. Rename it `Build And Send`.

### 4.2 Move the slow steps into it
Move these into **Build And Send**, keeping their order:
1. All redline steps (group C)
2. All doc steps (group D)
3. **Save Doc Link**, if you have it from Part 2.2. To add it now: inside the scope, click **+** directly under the last doc step (Create file) → **SharePoint** → **Update item** → rename it `Save Doc Link` → fill it in from the table in Part 2.2 Step 3.
4. The HRBP email child flow call (group E). It must come **after** Save Doc Link, because the child flow reads the doc link from SharePoint.

How to move them:
- **New designer:** drag each step into the scope, onto the **+** inside it. Start with the first redline step, then drop each next step under the previous one.
- **Classic designer:** drag each step into the scope's **Add an action** area. If a step won't drag, use **Copy to my clipboard** → paste it inside the scope → delete the original → rename the copy back.

### 4.3 Check the shape
Top to bottom, you should now have:
1. Trigger and setup steps
2. **Create item**
3. **Respond to the agent**
4. **Build And Send** (redlines → doc → Save Doc Link → HRBP email)

Nothing else below. **Save**.

---

## Part 5: Run the redlines in parallel (optional, but faster)

The doc only waits for the slowest redline, not the total of all of them. Do this only if no redline step uses another redline step's output. Open each one and check its inputs.

1. Inside **Build And Send**, find the first redline step, for example **Redline Purpose**.
2. Hover over the arrow **above** it → **+** → **Add a parallel branch** → add any placeholder action (for example a **Compose**). This creates a second branch beside it.
3. Drag the next redline step (for example **Redline Principal Duties**) into the new branch, and delete the placeholder Compose.
4. Repeat for each remaining redline step, so each one has its own branch side by side.
5. The first doc step (for example **Populate a Microsoft Word template**) must sit **below** all the branches, where they join. A step placed after parallel branches waits for every branch to finish.
6. **Save**.

> If the flow designer won't let you drag a step into a branch, leave the redlines in sequence. Part 3 already fixes the timeout; Part 5 only speeds up the emails.

---

## Part 6: Email the manager their document

The manager no longer gets a download link in the agent's reply. They get the redlined doc by email once it's ready.

### 6.1 Get the doc file
1. Inside **Build And Send**, click **+** directly under the **HRBP email child flow** step.
2. **Add an action** → **SharePoint** → **Get file content** → rename `Get Doc For Manager`.
   - **Site Address:** the site where the doc is saved (same as **Create file**).
   - **File Identifier:** **Dynamic content** → **Create file** → **Id**. Use the **Create file** step's actual name if you renamed it.

### 6.2 Send the email
1. Under **Get Doc For Manager**: **+** → **Add an action** → **Office 365 Outlook** → **Send an email from a shared mailbox (V2)** → rename `Email Manager Their Doc`.
2. **Original Mailbox Address:** `HRConnect@citynational.com`
3. **To:** the manager's email, from the same source your flow already uses for the requester. For example, the trigger's requester email input, or **Create item → RequestedBy** if that column holds the email.
4. **Subject:** **Expression** →
```
concat('Your JD update request #', outputs('Create_item')?['body/ID'], ' - ', outputs('Create_item')?['body/Title'])
```
5. **Body:** click **</>** (code view) → paste:
```
<p>Hi,</p>
<p>Your job description update request <b>#@{outputs('Create_item')?['body/ID']}</b> for <b>@{outputs('Create_item')?['body/Title']}</b> was submitted to HR.</p>
<p>Your redlined copy is attached. Removed text is struck through and added text is highlighted.</p>
<p>Your HR Business Partner will review it. If anything needs to change, you'll get an email from HR.</p>
<p>Job Description Expert</p>
```
   The `@{...}` parts show as dynamic content after you switch back from code view.
6. **Show advanced options** (or **Advanced parameters** → tick **Attachments**):
   - **Attachments Name - 1:** **Dynamic content** → **Create file** → **Name**
   - **Attachments Content - 1:** **Dynamic content** → **Get Doc For Manager** → **File Content**
7. **Importance:** Normal.
8. **Save**.

> This doc is the same one the manager could already download. It holds none of the restricted fields (Grade, Job Code, Salary Type, Status), so it's safe to send.

### 6.3 Optional: send it even if the HRBP email fails
By default, if the HRBP email child flow fails, the steps under it inside the scope are skipped. If you want the manager to get their doc either way:
1. **Get Doc For Manager** → **⋯** → **Settings** (or **Run after**).
2. For the HRBP email step, tick **is successful** and **has failed**.

The failure alert in Part 7 still fires, because the scope as a whole fails.

---

## Part 7: Failure alert

The agent has already told the manager "submitted", so if anything in the scope fails, HR needs to know and the manager needs to hear something.

### 7.1 Add the failure branch
1. Click **+** directly **under** the **Build And Send** scope (outside it, on the main line) → **Add an action** → **Scope** → rename `If Build Failed`.
2. **If Build Failed** → **⋯** → **Settings** (or **Run after**) → for **Build And Send**: untick **is successful**, tick **has failed** and **has timed out** → **Done**.

### 7.2 Alert HR
1. Inside **If Build Failed**: **+** → **Send an email from a shared mailbox (V2)** → rename `Alert HR Build Failed`.
2. **Original Mailbox Address:** `HRConnect@citynational.com`
3. **To:** **Dynamic content** → **Environment variables** → **JD HR Review Mailbox**. You can also add your own address while testing.
4. **Subject:** **Expression** →
```
concat('JD update request #', outputs('Create_item')?['body/ID'], ' saved but not processed')
```
5. **Body** (code view):
```
<p>Request <b>#@{outputs('Create_item')?['body/ID']}</b> (@{outputs('Create_item')?['body/Title']}) was saved, but building the redlined document or sending the review emails failed.</p>
<p>Flow run: @{concat('https://make.powerautomate.com/environments/', workflow()?['tags']?['environmentName'], '/flows/', workflow()?['name'], '/runs/', workflow()?['run']?['name'])}</p>
<p>Open the run, fix the cause, and resubmit the run, or send the review email by hand.</p>
```

### 7.3 Tell the manager
1. Under **Alert HR Build Failed**: **+** → **Send an email from a shared mailbox (V2)** → rename `Tell Manager Delayed`.
2. **Original Mailbox Address:** `HRConnect@citynational.com`
3. **To:** the manager's email (same as in 6.2 step 3).
4. **Subject:** **Expression** →
```
concat('Your JD update request #', outputs('Create_item')?['body/ID'], ' is delayed')
```
5. **Body** (code view):
```
<p>Your job description update request <b>#@{outputs('Create_item')?['body/ID']}</b> was received, but we couldn't create your redlined document. HR has been notified and will follow up. You don't need to resubmit.</p>
```

### 7.4 Mark the run as failed
1. Under **Tell Manager Delayed**: **+** → **Control** → **Terminate** → **Status:** **Failed** → **Code:** `BuildFailed` → **Message:** `Doc build or review email failed after the agent was answered.`
2. **Save**.

This makes failed runs show as **Failed** in run history, instead of **Succeeded**.

---

## Part 8: Update the Copilot topic

The flow's outputs changed, so the topic's flow action needs refreshing, and the confirmation message must stop using the doc outputs.

### 8.1 Refresh the flow action
1. **Copilot Studio** → the JD Expert agent → **Topics** → open the topic that calls **Save JD Update Request** (the submission topic).
2. Click the flow's **Action** node → **⋯** → **Refresh** (or remove the node and add the flow again from **Call an action**).
3. Check that the inputs are still mapped, and that **RequestID** is still mapped to its variable.
4. Outputs you deleted in Part 3.1 disappear. Any variable they filled (for example a doc link variable) is no longer set.

### 8.2 Fix the confirmation message
1. Find the **Message** node (or adaptive card) after the flow action that shows the download link or doc name.
2. Remove the download link, button or doc-name variable.
3. Replace the text with, for example:
   > Your update request **#{Topic.RequestID}** was submitted to HR. You'll get an email in a few minutes with your redlined document attached. Your HR Business Partner will review it and follow up by email if anything needs to change.

   Insert the RequestID variable with **{x}**, using your topic's actual variable name.
4. If an adaptive card shows the link, delete the link element or button from the card JSON.

### 8.3 Check for other references
1. Topic toolbar → **⋯** → **Variables** (or the **Variables** panel). Find any doc link or doc name variable you used.
2. Make sure no other node (a condition, a message, the agent instructions) uses it. Remove it from any that do.
3. **Save** → **Publish**.

---

## Part 9: Tests

| # | Test | Expected |
|---|---|---|
| 1 | Submit with a **long Purpose** (the kind that took 2.5 minutes) | The agent confirms within a few seconds with the Request ID. No timeout. |
| 2 | Same run, in **Run history** | The run shows **Running** for a few minutes, then **Succeeded**. The **Respond** step's timestamp is seconds after the start. |
| 3 | Same run, SharePoint item | Doc link and name are filled. **TitleReviewRequested** and **RequestStatus** are unchanged. |
| 4 | Same run, HRBP mailbox | The HRBP review email arrives with the doc, as before. |
| 5 | Same run, manager's inbox | **Email Manager Their Doc** arrives with the .docx attached. It opens in desktop Word. |
| 6 | A short request with **no title review** | Same as tests 1–5. TitleReviewRequested stays **No**. |
| 7 | **Forced failure**: inside **Build And Send**, add a temporary **Compose** with Expression `int('x')` right after the first redline step, then submit | The agent still confirms. HR gets **Alert HR Build Failed**, the manager gets **Tell Manager Delayed**, and the run shows **Failed**. **Delete the temporary Compose afterwards** and save. |

---

## Part 10: Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| The agent still times out | The response isn't directly under Create item, or a slow step still sits above it | Check the order (Part 4.3). The response must be on the main line, above the scope. |
| Save error: response "must be runAfter" a redline or doc step | A response output still uses a redline or doc step | Delete that output (Part 3.1). |
| Save error: "invalid reference" on Create item | A Create item field still uses a redline or doc step | Clear it and move it to Save Doc Link (Part 2.2). |
| TitleReviewRequested or RequestStatus changes after submit | Save Doc Link left a Yes/No or Choice column blank | Map each one to Create item's value (Part 2.2 Step 3). |
| HRBP email has no doc, or a "File Not Found" link | The HRBP child flow runs before Save Doc Link | Move the child flow step below Save Doc Link (Part 4.2). |
| Agent shows an error after you refreshed the flow | The topic still uses a deleted output variable | Part 8.2 and 8.3. |
| Manager email has no attachment | Attachments Content is blank, or uses the wrong step | **Attachments Content - 1** → Get Doc For Manager → **File Content**. |
| The redlines didn't speed up | They still run in sequence, or the doc step sits inside one branch | Part 5. The doc step must sit below all branches. |
| Failure alert fires on every run | **If Build Failed** has **is successful** ticked | Part 7.1 step 2: untick it. |
