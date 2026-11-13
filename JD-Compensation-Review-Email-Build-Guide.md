# Compensation Review Email: Build Guide

**What you'll build:** when HR clicks **Approve** in the HR review email, the request goes to the Compensation team in a designed email. That email shows everything HR saw, plus who approved it, HR's notes, the current classification and the **related job levels with their grades**. Comp then chooses **Approve** (it goes on to Operations) or **Return To HR**.

The parts, in the order you build them:

| Part | What | Where |
|---|---|---|
| 1 | Columns and statuses | SharePoint |
| 2 | Compensation mailbox setting | Solution (environment variable) |
| 3 | Upload the Comp template | SharePoint (JD Templates) |
| 4 | Point HR's Approve button at HRConnect | Send HR Review Email |
| 5 | Save the related job titles at submission | Save JD Update Request + agent |
| 6 | Create the child flow (a copy of the HR email flow) | Send Comp Review Email |
| 7 | Recipient, template, subject, preheader | Send Comp Review Email |
| 8 | Related job levels lookup | Send Comp Review Email |
| 9 | Blocks | Send Comp Review Email |
| 10 | Repeats (including Related rows) | Send Comp Review Email |
| 11 | Placeholders | Send Comp Review Email |
| 12 | Send, respond, connections | Send Comp Review Email |
| 13 | Test the child flow on its own | — |
| 14 | The flow that runs on HR's Approve | Process HR Approval Reply |
| 15 | Tighten the decline flow's status check | Process HR Decline Reply |
| 16 | End-to-end tests | — |

Appendix A lists the Comp template's new placeholders, Appendix B covers troubleshooting, and Appendix C explains how to remove the old script-based lookup if you built it.

---

## How it works

```
HR review email → [Approve]
   Outlook opens: To HRConnect@citynational.com
                  Subject [APPROVE] JD Request #42 - Facilities Coordinator II
                  Body    Notes for the Compensation team (optional)…  [Write above this line…]
        ↓ HR sends
Process HR Approval Reply  (HRConnect, subject contains [APPROVE])
   checks: internal sender · request # found · request exists · status Submitted or Returned to HR
   Run a Child Flow → Send Comp Review Email (RequestID, HRApprovedBy, HRApprovalNotes)
        ├─ success → mark With Compensation → confirm to HR
        └─ failure → bounce to HR, status unchanged
Send Comp Review Email
   gathers the same data as the HR email + related levels (from RelatedTitles, with grades)
   fills JD Compensation Review Email.html → sends to the Comp mailbox with the redlined doc
```

The agent never sees grades. The related titles are saved when the request is submitted, and the grades are looked up later, on the HR side only.

### Conventions used in this guide
- **Expression:** click the field → choose the **Expression** tab (or the **fx** icon) → paste → **Add** (or **OK**).
- **Rename** a step by clicking its title. Use the names **exactly** as written, because other expressions refer to them by name, with spaces turned into underscores (`Get Request` → `body('Get_Request')`).
- **Run after:** on a step → **⋯** → **Settings** (or **Configure run after**). It controls whether the step runs when the step above it fails or is skipped.
- **Your site, your lists:** "your site" is the SharePoint site with **JD Update Requests**, your **JD list** and the **JD Templates** library.

### Before you start
- **Send HR Review Email** (the HR child flow) works end to end, including the document links and attachment.
- **Process HR Decline Reply** exists. Part 14 reuses its approach.
- You have **JD Compensation Review Email.html** (the final version, with Classification and Related Job Levels near the top and the "When You Approve, Fill In" box).

---

## Part 1: SharePoint columns and statuses

### 1.1 New columns on JD Update Requests
Open the **JD Update Requests** list → **+ Add column** for each:

| Name | Type | Settings |
|---|---|---|
| `HRApprovedBy` | Single line of text | — |
| `HRApprovedOn` | Date and time | **Include time:** Yes |
| `HRApprovalNotes` | Multiple lines of text | **More options** → **Use enhanced rich text:** Off |
| `RelatedTitles` | Multiple lines of text | **More options** → **Use enhanced rich text:** Off |

Create each without spaces in its name, so the internal name stays clean. You can change the display name afterwards in **Column settings** → **Edit**.

### 1.2 Request statuses
1. Click the **RequestStatus** column header → **Column settings** → **Edit**.
2. Under **Choices**, add any of these that are missing:
   - `With Compensation`
   - `Returned to HR`
   - `Sent to Operations`
3. **Save**.

---

## Part 2: Compensation mailbox setting

1. **make.powerautomate.com** → **Solutions** → your JD solution → **+ New** → **More** → **Environment variable**.
2. **Display name:** `JD Compensation Mailbox`
3. **Data type:** Text
4. **Default value:** the Comp team's mailbox or distribution list. **In DEV and UAT, use your own address** so test emails don't reach the Comp team.
5. **Save**.

An environment variable can have both a **Default value** and a **Current value**, and the current value wins. If you change the address later, check both, then turn the flows that use it **off and on**.

---

## Part 3: Upload the Comp template

1. Open the **JD Templates** library → **Upload** → **Files** → choose **JD Compensation Review Email.html**. Keep that exact name.
2. If asked, **Replace** the existing file.
3. Don't open or save it in SharePoint or Word, because that rewrites the HTML.
4. Repeat in each environment (DEV, UAT, QA). Templates aren't part of the solution.

---

## Part 4: Point HR's Approve button at HRConnect

The Approve button becomes a `mailto:` to **HRConnect@citynational.com** with an `[APPROVE]` subject and an optional notes prompt.

1. **Solutions** → your solution → **Send HR Review Email** → **Edit**.
2. Open the **Token List** Compose.
3. Replace the whole `ActionUrl_Approve` row with:
```
  {"k": "ActionUrl_Approve",  "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[APPROVE] JD Request #', body('Get_Request')?['ID'], ' - ', body('Get_Request')?['Title'])), '&body=', encodeUriComponent(concat('Notes for the Compensation team (optional). Leave blank if none.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
```
4. Replace the text of the `ActionHelpText` row with:
```
Approve opens an email that sends this request to Compensation, with optional notes. Decline opens an email for your reasons; the manager receives them with AI-suggested fixes.
```
5. **Save**.

The sentence `Leave blank if none.` and the line `[Write above this line.` are exactly what Part 14 looks for. If you reword them here, reword them there too.

**Check:** submit a test request through the agent. In the HR email, click **Approve**. Outlook should open a message to HRConnect with the `[APPROVE] JD Request #NN - …` subject and the notes prompt. Emails sent before this change keep their old Approve link.

---

## Part 5: Save the related job titles at submission

The agent already has the job's related levels (`Global.RelatedLevels`, from your Related Levels flow). You save their **titles** on the request, and the Comp flow looks up their grades later.

### 5.1 Find the title key in the related levels JSON
1. Open your **Related Levels** flow → **Run history** → a recent run → **Select related** → **Show raw outputs**.
2. Note two things:
   - **Does the output start with `[`** (a list)? Or is it wrapped in an object, like `{"levels": [ … ]}`?
   - **Which key holds the job title** in each item, e.g. `"title"`, `"Job Title"` or `"JobTitle"`. This guide calls it `TITLEKEY`.

### 5.2 Save JD Update Request: a new input
1. **Save JD Update Request** → **Edit** → click the trigger → **+ Add an input** → **Text** → `RelatedLevels`.
2. **⋯** next to the new input → **Make the field optional**.

### 5.3 Pull out the titles
1. Click the **+** directly **above Create item** → **Data Operation** → **Select** → rename `Related Title List`.
2. **From** → **Expression**. Type `if(empty(`, switch to **Dynamic content** and click the trigger's **RelatedLevels**, switch back to **Expression** and type `), json('[]'), json(`, insert **RelatedLevels** again, and type `))`. It should read:
```
if(empty(triggerBody()?['text_XX']), json('[]'), json(triggerBody()?['text_XX']))
```
   (`text_XX` is whatever key the designer inserted.) If 5.1 showed an object wrapper, put the key after the last `json(…)`, for example `json(triggerBody()?['text_XX'])?['levels']`.
3. **Map** → **Switch Map to text mode** (the **T** icon) → **Expression**:
```
trim(string(item()?['TITLEKEY']))
```
   Replace `TITLEKEY` with the key from 5.1.
4. Below it: **+** → **Compose** → rename `RelatedTitles` → **Expression**:
```
join(body('Related_Title_List'), decodeUriComponent('%0A'))
```

### 5.4 Store them
1. Open **Create item** → **Advanced parameters** → **Show all**. If **RelatedTitles** isn't listed, re-select **List Name** to reload the columns, then **re-check every field**.
2. **RelatedTitles** → **Dynamic content** → **Outputs** of **RelatedTitles**.
3. **Save**.

### 5.5 Send it from the agent
1. Copilot Studio → **Topics** → **Submit JD Update** → click the **Save JD Update Request** node.
2. **⋯** → **Refresh**. If **RelatedLevels** isn't listed, add it from **+ Add input** / **Optional inputs**.
3. **RelatedLevels** → **{x}** → `Global.RelatedLevels`.
4. **Save** the topic → **Publish** the agent.

`Global.RelatedLevels` is cleared in the reset that runs **after** this node, so it still has its value here.

### 5.6 Check
| Submit an update for… | Expect |
|---|---|
| A job **with** other levels | **RelatedTitles** on the new item lists one related title per line |
| A job **without** other levels | **RelatedTitles** is empty, and the flow and HR email work as normal |

---

## Part 6: Create the child flow (a copy of the HR email flow)

Copying keeps every data step that already works: the request, JD record, classification, flags, level and title details, change summary, doc links and attachment.

### 6.1 Make the copy
1. **Solutions** → your solution → **Cloud flows** → click **Send HR Review Email** to open its details page.
2. Top bar → **Save As** → **Flow name:** `Send Comp Review Email` → **Save**. The copy is created **off** and **outside** the solution.
3. **Solutions** → your solution → **Add existing** → **Automation** → **Cloud flow** → **Outside Dataverse** (or **Outside solutions**) → tick **Send Comp Review Email** → **Add**.
4. Open **Send Comp Review Email** → **Edit**. Make every change from here to Part 12 in this copy. Don't touch the original.

### 6.2 Inputs
1. Click the trigger (**Manually trigger a flow**). Keep **RequestID**.
2. **+ Add an input** → **Text** → `HRApprovedBy`.
3. **+ Add an input** → **Text** → `HRApprovalNotes` → **⋯** → **Make the field optional**.

### 6.3 Values from the inputs
Below the **SiteUrl** Compose, add three **Compose** actions:

| Name | Inputs |
|---|---|
| `HRApprovedBy` | **Dynamic content** → the trigger's **HRApprovedBy** |
| `HRApprovalNotes` | **Expression:** type `coalesce(`, insert the trigger's **HRApprovalNotes** from Dynamic content, type `, '')` |
| `HRApprovedDate` | **Expression:** `convertFromUtc(utcNow(), 'Eastern Standard Time', 'MMMM d, yyyy')` |

### 6.4 Check the step order
Right under the trigger, the first steps must be **Initialize variable (Html)** and **Initialize variable (Attachments)**. **Respond to a PowerApp or flow** must be the **last** step. If it ends up near the top, every step below it is skipped in tests (see Appendix B).

---

## Part 7: Recipient, template, subject, preheader

### 7.1 Recipient
1. Find the **HRMailbox** Compose → rename it `CompMailbox` → clear **Inputs** → **Dynamic content** → **Environment variables** → **JD Compensation Mailbox**.
2. Find **Send HR Email** → rename it `Send Comp Email` → **To** → **Dynamic content** → **CompMailbox**.

### 7.2 Template
**Get Template** → **File Path** → folder icon → **JD Templates** → **JD Compensation Review Email.html**.

### 7.3 Subject
**Subject** Compose → replace its expression with:
```
concat(if(greater(length(outputs('FlagChips')), 0), '[Flagged] ', ''), 'Compensation Review: JD Update #', body('Get_Request')?['ID'], ' · ', body('Get_Request')?['Title'])
```

### 7.4 Preheader
**Preheader** Compose → replace its expression with:
```
concat('HR approved by ', outputs('HRApprovedBy'), ' · ', body('Get_Request')?['Title'], ' · ', if(greater(length(outputs('FlagChips')), 0), concat(length(outputs('FlagChips')), ' items flagged'), 'No flags raised'))
```

**Save**. Red errors on the old Approve and Decline Token List rows are expected at this point, and Part 11 removes them.

---

## Part 8: Related job levels lookup

This reads **RelatedTitles** from the request and gets those jobs, **with grades**, from your JD list. If there are no related titles, which is common, it finds nothing, the section is hidden and nothing fails.

```
(below the Classification Compose)
Related Title List → Related Title Filters → Get Related → Related Others → Related All → Related Rows
```

Two internal names come up below. Check them in your **Get JD** step if you're not sure:
- **The JD list's job title column**, written as `Title` below. That's the SharePoint default.
- **The JD list's job code column**, written as `JOBCODE_COL` below. It's the column your **Get JD** filter uses.

### 8.1 Related Title List
1. Click the **+** directly **below Classification** → **Data Operation** → **Filter array** → rename `Related Title List`.
2. **From** → **Expression**:
```
split(coalesce(body('Get_Request')?['RelatedTitles'], ''), decodeUriComponent('%0A'))
```
3. **Edit in advanced mode** → paste:
```
@and(not(equals(trim(item()), '')), not(equals(toLower(trim(item())), toLower(trim(string(body('Get_Request')?['Title']))))))
```
   This drops blank lines and the job's own title.

### 8.2 Related Title Filters
1. **+** → **Data Operation** → **Select** → rename `Related Title Filters`.
2. **From** → **Expression**: `body('Related_Title_List')`
3. **Map** → **Switch Map to text mode** → **Expression**:
```
concat('Title eq ''', replace(trim(item()), '''', ''''''), '''')
```
   Each title becomes `Title eq 'Facilities Coordinator I'`. Apostrophes in titles are doubled, so they can't break the filter.

### 8.3 Get Related
1. **+** → **SharePoint** → **Get items** → rename `Get Related`.
   - **Site Address:** your site · **List Name:** your JD list
   - **Filter Query** → **Expression**:
```
if(empty(body('Related_Title_Filters')), 'ID eq 0', concat('(', join(body('Related_Title_Filters'), ' or '), ') and Status eq ''Active'''))
```
   - **Top Count:** `50`

   When there are no related titles, the filter is `ID eq 0`, which safely returns nothing. If your job title column isn't `Title`, change `Title` in 8.2 to its internal name.

### 8.4 Related Others
1. **+** → **Data Operation** → **Filter array** → rename `Related Others`.
2. **From** → **Expression**: `body('Get_Related')?['value']`
3. **Edit in advanced mode**:
```
@not(equals(string(item()?['JOBCODE_COL']), string(body('Get_Request')?['JobCode'])))
```
   Replace `JOBCODE_COL` with the JD list's job code internal name.

### 8.5 Related All
1. **+** → **Compose** → rename `Related All` → **Expression**:
```
if(empty(outputs('JD')), body('Related_Others'), union(body('Related_Others'), createArray(outputs('JD'))))
```
   This adds the current job, so it shows highlighted in the table. It's skipped safely if the JD record wasn't found.

### 8.6 Related Rows
1. **+** → **Data Operation** → **Select** → rename `Related Rows`.
2. **From** → **Expression**: `outputs('Related_All')`
3. **Map** (key/value mode). Add four rows:

| Key | Value (Expression) |
|---|---|
| `title` | `string(item()?['Title'])` |
| `code` | Your **Classification** Compose's **Job Code** expression, with `outputs('JD')` changed to `item()` |
| `grade` | Classification's **Grade** expression, with `outputs('JD')` changed to `item()` |
| `stype` | Classification's **Salary Type** expression, with `outputs('JD')` changed to `item()` |

   For example, `"Grade": "@{outputs('JD')?['Grade']?['Value']}"` in Classification becomes `item()?['Grade']?['Value']` here.

   **Every value must use `item()`.** If one still says `outputs('JD')`, every row shows the current job's code, grade and salary type, and every row is marked "(This Job)".
4. **Save**.

---

## Part 9: Blocks

Open the **Block List** Compose. Add a comma after the current last row, then add these two rows before the closing `]`:
```
  {"name": "HRApprovalNotes", "keep": "@{if(empty(outputs('HRApprovalNotes')), 'no', 'yes')}"},
  {"name": "Related",         "keep": "@{if(greater(length(body('Related_Rows')), 1), 'yes', 'no')}"}
```
- **HRApprovalNotes** shows the notes card only when HR wrote notes.
- **Related** shows the section only when there's **at least one level besides the job itself**.

All the other rows stay as they are. **Apply Blocks** doesn't change.

---

## Part 10: Repeats (including Related rows)

The existing repeats (Reasons, Sections, Flags, Indirect) stay as they are. Add the Related repeat **after the last existing repeat** (the Set variable after **After Impacts**) and **before Token List**.

### 10.1 Related Snippet
**+** → **Compose** → rename `Related Snippet` → **Expression**:
```
if(contains(variables('Html'), '<!--REPEAT:Related-->'), first(split(last(split(variables('Html'), '<!--REPEAT:Related-->')), '<!--/REPEAT:Related-->')), '')
```

### 10.2 Related Html
1. **+** → **Data Operation** → **Select** → rename `Related Html`.
2. **From** → **Expression**:
```
sort(body('Related_Rows'), 'title')
```
3. **Map** → **Switch Map to text mode** → **Expression**:
```
replace(replace(replace(replace(replace(replace(outputs('Related_Snippet'), '{{RelRowBg}}', if(equals(string(item()?['code']), string(body('Get_Request')?['JobCode'])), '#FFF1CC', '#FFFFFF')), '{{RelNote}}', if(equals(string(item()?['code']), string(body('Get_Request')?['JobCode'])), ' (This Job)', '')), '{{RelTitle}}', replace(replace(replace(string(item()?['title']), '&', '&amp;'), '<', '&lt;'), '>', '&gt;')), '{{RelJobCode}}', replace(replace(replace(string(item()?['code']), '&', '&amp;'), '<', '&lt;'), '>', '&gt;')), '{{RelGrade}}', replace(replace(replace(coalesce(string(item()?['grade']), ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;')), '{{RelSalaryType}}', replace(replace(replace(coalesce(string(item()?['stype']), ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;'))
```
   The sort uses the lowercase `title` key you created in Related Rows, so every row is guaranteed to have it.

### 10.3 After Related
**+** → **Compose** → rename `After Related` → **Expression**:
```
if(contains(variables('Html'), '<!--REPEAT:Related-->'), concat(first(split(variables('Html'), '<!--REPEAT:Related-->')), join(body('Related_Html'), ''), last(split(variables('Html'), '<!--/REPEAT:Related-->'))), variables('Html'))
```

### 10.4 Save to Html
**+** → **Set variable** → rename `Set Html After Related` → **Name:** Html → **Value** → **Expression**: `outputs('After_Related')`

**Save**.

The `contains()` checks matter. When the Related block was removed in Apply Blocks, these steps leave the email alone.

---

## Part 11: Placeholders

Open the **Token List** Compose.

### 11.1 Remove
Delete these rows. The Comp template doesn't have these placeholders:
- `ActionUrl_Approve`
- `ActionUrl_Decline`
- `ActionHelpText`
- `ActionUrl_Changes` (if it's still there)

### 11.2 Add
Add these rows before the closing `]`, with a comma between rows:
```
  {"k": "HRApprovedBy",          "raw": "no",  "v": "@{outputs('HRApprovedBy')}"},
  {"k": "HRApprovedDate",        "raw": "no",  "v": "@{outputs('HRApprovedDate')}"},
  {"k": "HRApprovalNotes",       "raw": "no",  "v": "@{outputs('HRApprovalNotes')}"},
  {"k": "ActionUrl_CompApprove", "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[COMP-APPROVE] JD Request #', body('Get_Request')?['ID'], ' - ', body('Get_Request')?['Title'])), '&body=', encodeUriComponent(concat('Salary Grade: ', decodeUriComponent('%0D%0A%0D%0A'), 'FLSA Classification: ', decodeUriComponent('%0D%0A%0D%0A'), 'Comments: ', decodeUriComponent('%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "ActionUrl_CompDecline", "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[COMP-DECLINE] JD Request #', body('Get_Request')?['ID'], ' - ', body('Get_Request')?['Title'])), '&body=', encodeUriComponent(concat('Why is this request being returned to HR? Your reasons go to HR.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "CompActionHelpText",    "raw": "no",  "v": "Approve opens an email with fields for Salary Grade, FLSA Classification and Comments. Fill them in and send, and the request goes to Operations. Return To HR opens an email for your reasons, and the request goes back to HR."}
```
The labels `Salary Grade:`, `FLSA Classification:` and `Comments:` and the marker line are what the future Comp reply flow will read. Don't reword them without updating that flow.

**Apply Tokens** and **Leftover Tokens** don't change. **Save**.

---

## Part 12: Send, respond, connections

### 12.1 Send Comp Email
Keep the settings copied from the HR flow:
- **Body:** in code view (**</>**), containing only the **Html** variable;
- **Attachments:** the entire array `variables('Attachments')`;
- **Run after:** is successful, has failed, is skipped.

### 12.2 Respond to the parent flow
1. Check that **Respond to a PowerApp or flow** is the **last** step, below **Send Comp Email**.
2. Output: **EmailSent** (Text) = `yes`.
3. **⋯** → **Settings** → **Run after** → for **Send Comp Email**, tick **only is successful**.

   If the email fails, the child then ends as **failed**, so the approval flow (Part 14) bounces to HR instead of marking the request With Compensation.

### 12.3 Save, connections, turn on
1. **Save**.
2. Go to the flow's **details page** → **Run only users** → **Edit** → for **SharePoint**, **Office 365 Users** and **Office 365 Outlook**, choose **Use this connection** → **Save**.
3. **Turn on** the flow.

---

## Part 13: Test the child flow on its own

1. Set **JD Compensation Mailbox** to your own address.
2. In **JD Update Requests**, find a test request **submitted after Part 5**, so it has RelatedTitles, for a job with other levels. Note its **ID**. Show the **ID** column via **+ Add column** → **Show/hide columns** if it's hidden.
3. **Send Comp Review Email** → **Edit** → **Test** → **Manually** → **Test**:
   - **RequestID:** that ID · **HRApprovedBy:** your name · **HRApprovalNotes:** `Please confirm grade against the Coordinator II level.`
4. **Run flow**.

| Check | Expect |
|---|---|
| The run | Every step green, except **Respond**, which shows **skipped** ("The client application is not waiting…"). That's normal for a manual test |
| **Related Title List** / **Get Related** | The related titles, then those jobs as full items with grades |
| **Related Rows** | One row per level, including the current job, each with its **own** code, grade and salary type |
| **Block List** → Related | `"keep": "yes"` |
| The email | Subject `Compensation Review: JD Update #NN · <title>` |
| HR Approval | "Approved By <you> On <today>", plus the notes card |
| Current Classification + Related Job Levels | Near the top. Levels are in title order, and only the current job is highlighted and marked **(This Job)** |
| Everything else | Matches the HR email for the same request |
| **Approve** button | `[COMP-APPROVE] JD Request #NN - …` with the three labelled lines |
| **Return To HR** button | `[COMP-DECLINE] JD Request #NN - …` with the reasons prompt |
| **Leftover Tokens** | **false** |

Run it again with:
- a request for a job with **no** related levels → no Related section, and no errors;
- **HRApprovalNotes** empty → no notes card.

---

## Part 14: Process HR Approval Reply

This flow runs when HR's `[APPROVE]` email lands in HRConnect. The account behind your **Office 365 Outlook** connection needs **Full Access** and **Send As** on HRConnect@citynational.com.

`[APPROVE]` doesn't match Comp's later `[COMP-APPROVE]`, because the filter needs the `[` right before the word.

### 14.1 Create the flow
1. **Solutions** → your solution → **+ New** → **Automation** → **Cloud flow** → **Automated**.
2. **Name:** `Process HR Approval Reply` · **Trigger:** **When a new email arrives in a shared mailbox (V2)** → **Create**.
3. Trigger:
   - **Original Mailbox Address:** `HRConnect@citynational.com`
   - **Folder:** Inbox
   - **Subject Filter:** `[APPROVE]`
   - **Include Attachments:** No
4. **⋯** on the trigger → **Settings** → **Concurrency control:** On → **Degree of parallelism:** `1` → **Done**.

You can **copy** steps 14.2–14.6 from **Process HR Decline Reply** (**⋯** → **Copy to my clipboard**, then **+** → **Paste an action**) instead of building them. Keep the same names, and apply the differences noted.

### 14.2 Basic values
Three **Compose** actions:

| Name | Inputs |
|---|---|
| `SenderEmail` | Expression: `toLower(triggerOutputs()?['body/from'])` |
| `AllowedDomain` | Expression: `concat('@', 'citynational.com')`. Don't type a leading `@` as plain text, because the designer treats it as an invalid expression |
| `RequestIdText` | Expression: `first(split(trim(last(split(coalesce(triggerOutputs()?['body/subject'], ''), 'JD Request #'))), ' '))` |

### 14.3 Guard: internal sender
1. **+** → **Condition** → rename `Internal sender` → left: **Expression** `endsWith(outputs('SenderEmail'), toLower(outputs('AllowedDomain')))` → **is equal to** → right: **Expression** `true`.
2. **False** → **Terminate** → **Succeeded**.
3. Leave **True** empty. The next steps go **below** the condition.

### 14.4 Guard: request number
1. **+** → **Condition** → rename `Has request number` → left: **Expression** `isInt(outputs('RequestIdText'))` → **is equal to** → right: **Expression** `true`.
2. **False**:
   - **Send an email from a shared mailbox (V2)** → rename `Bounce No Number` → **Original Mailbox Address:** `HRConnect@citynational.com` · **To:** `outputs('SenderEmail')` · **Subject:** `Approval not processed` · **Body:** `I couldn't find a request number in your email's subject, so nothing was sent to Compensation. Please click Approve in the HR review email again and keep the subject as it is.`
   - **Terminate** → **Succeeded**.

### 14.5 HR's notes (optional)
1. **+** → **Content Conversion** → **Html to text** (keep the name) → **Content:** **Expression** `triggerOutputs()?['body/body']`
2. **+** → **Compose** → rename `CommentsRaw` → **Expression**:
```
replace(replace(first(split(if(contains(body('Html_to_text'), 'Leave blank if none.'), split(body('Html_to_text'), 'Leave blank if none.')[1], body('Html_to_text')), '[Write above this line.')), decodeUriComponent('%0D'), ''), decodeUriComponent('%C2%A0'), ' ')
```
   *(If you copied this from the decline flow, this expression is the one difference. It looks for the Approve prompt.)*
3. **+** → **Data Operation** → **Select** → rename `Comment Lines` → **From:** `split(outputs('CommentsRaw'), decodeUriComponent('%0A'))` → **Map** in text mode → `trim(item())`
4. **+** → **Compose** → rename `Comments` → **Expression**:
```
trim(replace(replace(replace(replace(replace(replace(replace(replace(replace(join(body('Comment_Lines'), decodeUriComponent('%0A')), concat(decodeUriComponent('%0A'), decodeUriComponent('%0A')), '~~P~~'), concat(decodeUriComponent('%0A'), '-'), '~~B~~-'), concat(decodeUriComponent('%0A'), '*'), '~~B~~*'), concat(decodeUriComponent('%0A'), decodeUriComponent('%E2%80%A2')), concat('~~B~~', decodeUriComponent('%E2%80%A2'))), decodeUriComponent('%0A'), ' '), '~~B~~', decodeUriComponent('%0A')), '~~P~~', concat(decodeUriComponent('%0A'), decodeUriComponent('%0A'))), concat(decodeUriComponent('%0A'), ' '), decodeUriComponent('%0A')), '  ', ' '))
```
   This joins lines that Html to text wrapped, and keeps paragraphs and lines starting with `-`, `*` or `•`.

### 14.6 Get the request
1. **+** → **SharePoint** → **Get items** → rename `Get Request` → your site, **JD Update Requests** → **Filter Query:** type `ID eq ` then **Expression** `outputs('RequestIdText')` → **Top Count:** `1`.
2. **+** → **Compose** → rename `Request` → **Expression**: `first(body('Get_Request')?['value'])`

### 14.7 Guard: open for approval
1. **+** → **Compose** → rename `Problem` → **Expression**:
```
if(empty(outputs('Request')), concat('I couldn''t find request #', outputs('RequestIdText'), ', so nothing was sent.'), if(not(contains(createArray('Submitted', 'Returned to HR'), coalesce(outputs('Request')?['RequestStatus']?['Value'], ''))), concat('Request #', outputs('RequestIdText'), ' is ', coalesce(outputs('Request')?['RequestStatus']?['Value'], 'not open for approval'), ', so it can''t be approved from this email. Nothing was sent.'), ''))
```
   Notes are optional, so there's no length check.
2. **+** → **Condition** → rename `No problem` → left: **Expression** `outputs('Problem')` → **is equal to** → right: leave empty.
3. **False**:
   - **Send an email from a shared mailbox (V2)** → rename `Bounce Problem` → **Original Mailbox Address:** HRConnect · **To:** `outputs('SenderEmail')` · **Subject:** `concat('Approval not processed: JD Request #', outputs('RequestIdText'))` · **Body:** `outputs('Problem')`
   - **Terminate** → **Succeeded**.
4. Leave **True** empty. The rest goes **below** the condition.

### 14.8 Who approved
1. **+** → **Office 365 Users** → **Get user profile (V2)** → rename `Get Approver` → **User (UPN):** `outputs('SenderEmail')`
2. **+** → **Compose** → rename `ApprovedByName` → `coalesce(body('Get_Approver')?['displayName'], outputs('SenderEmail'))`
3. On **ApprovedByName**: **Run after** → for **Get Approver**, tick **is successful**, **has failed**, **is skipped** and **has timed out**.

### 14.9 Send the Comp email
1. **+** → **Flows** (Built-in) → **Run a Child Flow** → **Child Flow:** **Send Comp Review Email**. If it isn't listed, check it's in the same solution and turned **on**.
2. Inputs:

| Input | Expression |
|---|---|
| **RequestID** | `int(outputs('RequestIdText'))` |
| **HRApprovedBy** | `outputs('ApprovedByName')` |
| **HRApprovalNotes** | `outputs('Comments')` |

### 14.10 If the Comp email fails
1. Hover just below **Run a Child Flow** → **+** → **Add a parallel branch** → **Send an email from a shared mailbox (V2)** → rename `Bounce Comp Failed`:
   - **Original Mailbox Address:** HRConnect · **To:** `outputs('SenderEmail')`
   - **Subject:** `concat('Approval not processed: JD Request #', outputs('RequestIdText'))`
   - **Body:** `The Compensation email couldn't be sent, so this request is still waiting on HR. Please try Approve again, or contact the JD agent owner if it keeps failing.`
2. **Run after** → for **Run a Child Flow**, tick **only has failed** and **has timed out**.
3. Below it, in the same branch: **Terminate** → **Status:** Failed → **Message:** `Comp email failed`.

### 14.11 Mark With Compensation
In the **other** branch, directly under **Run a Child Flow**:
1. **SharePoint** → **Update item** → rename `Mark With Compensation` → your site, **JD Update Requests**:

| Field | Value |
|---|---|
| **Id** | `int(outputs('RequestIdText'))` |
| **Title** | `outputs('Request')?['Title']` |
| **RequestStatus Value** | `With Compensation` |
| **HRApprovedBy** | `outputs('ApprovedByName')` |
| **HRApprovedOn** | `utcNow()` |
| **HRApprovalNotes** | `outputs('Comments')` |
| **TitleReviewRequested** | `outputs('Request')?['TitleReviewRequested']` |

   **Map every Yes/No column to its current value.** Update item resets a Yes/No column that's left blank to the column's default, so TitleReviewRequested would flip. Do the same for any other Yes/No column on the list.

### 14.12 Confirm to HR
1. **+** → **Send an email from a shared mailbox (V2)** → rename `Confirm To HR`:
   - **Original Mailbox Address:** HRConnect · **To:** `outputs('SenderEmail')`
   - **Subject:** `concat('Sent to Compensation: JD Request #', outputs('RequestIdText'))`
   - **Body:**
```
concat('Request #', outputs('RequestIdText'), ' (', outputs('Request')?['Title'], ') was approved and sent to Compensation.', if(empty(outputs('Comments')), '', ' Your notes were included.'))
```
2. Optional: **Office 365 Outlook** → **Move email (V2)** → **Message Id:** `triggerOutputs()?['body/id']` → **Folder:** **Processed Approvals** (create it in HRConnect first) → **Original Mailbox Address:** HRConnect.
3. **Save**.

---

## Part 15: Tighten the decline flow's status check

The decline flow blocks only **Declined** and **Approved**, so HR could still decline a request that's already with Compensation. Switch it to an allowlist:

1. **Process HR Decline Reply** → **Edit** → **Problem** → replace its expression with:
```
if(empty(outputs('Request')), concat('I couldn''t find request #', outputs('RequestIdText'), ', so nothing was sent.'), if(not(contains(createArray('Submitted', 'Returned to HR'), coalesce(outputs('Request')?['RequestStatus']?['Value'], ''))), concat('Request #', outputs('RequestIdText'), ' is ', coalesce(outputs('Request')?['RequestStatus']?['Value'], 'not open for a decision'), ', so it can''t be declined from this email. Nothing was sent.'), if(less(length(outputs('Comments')), 15), 'Your comments were empty or too short, so nothing was sent to the manager. Please click Decline in the HR review email again and write, above the line, why the request is declined and what needs to change.', '')))
```
2. **Save**.

---

## Part 16: End-to-end tests

Keep **JD Compensation Mailbox** pointed at your own address. Submit **fresh** requests through the agent, because older ones lack the newer columns.

| # | Do | Expect |
|---|---|---|
| 1 | Submit for a job **with** other levels → HR **Approve** with no notes | The Comp email arrives with the Related Job Levels table and grades, and no notes card. Status **With Compensation**, HR Approved By/On filled. HR gets "Sent to Compensation" |
| 2 | Submit for a job **without** other levels → **Approve** with notes | The Comp email has no Related section, shows HR's notes, and has no errors. **HRApprovalNotes** is filled |
| 3 | **Approve** test 1's request again | Bounce: "Request #NN is With Compensation, so it can't be approved…" |
| 4 | **Decline** test 1's request | Bounce: "…can't be declined from this email" |
| 5 | **Approve** with the subject changed to `#99999` | Bounce: "I couldn't find request #99999…" |
| 6 | Rename the Comp template in JD Templates for a moment → **Approve** a new request | Bounce: "The Compensation email couldn't be sent…". The status stays **Submitted**. Rename the template back afterwards |
| 7 | **Decline** a **Submitted** request | Works as before |
| 8 | In the Comp email, click **Approve** and **Return To HR** | Correct subjects and pre-filled bodies. Nothing processes them yet, because that's the Comp reply flow |

**In each run, check:**
- **Process HR Approval Reply:** the trigger, then green through **Confirm To HR**. Bounce cases end at **Terminate (Succeeded)**.
- **Send Comp Review Email** (called runs): every step green, **Respond** included. **Leftover Tokens** is false.

Before go-live, in each environment: upload the template, set **JD Compensation Mailbox** to the real Comp address, check the HRConnect permissions, and publish the agent.

---

## Appendix A: New placeholders in the Comp template

| Placeholder / marker | Filled by |
|---|---|
| `{{HRApprovedBy}}`, `{{HRApprovedDate}}`, `{{HRApprovalNotes}}` | Token List (Part 11) |
| `BLOCK:HRApprovalNotes` | Block List: shown when notes exist |
| `BLOCK:Related` | Block List: shown when Related Rows has more than 1 row |
| `REPEAT:Related` with `{{RelTitle}}`, `{{RelNote}}`, `{{RelJobCode}}`, `{{RelGrade}}`, `{{RelSalaryType}}`, `{{RelRowBg}}` | Related Html (Part 10) |
| `{{ActionUrl_CompApprove}}`, `{{ActionUrl_CompDecline}}`, `{{CompActionHelpText}}` | Token List (Part 11) |

Removed from the Comp template (and its Token List): `{{ActionUrl_Approve}}`, `{{ActionUrl_Decline}}`, `{{ActionHelpText}}`. Every other placeholder is the same as in the HR email, and so are its Token List rows.

---

## Appendix B: Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "Invalid reference to X" when saving | A step was renamed after it was referenced, or it sits below or beside the step using it | Rename it exactly as in this guide. Make sure it's above, on the main line (not in a branch). Or delete the reference and re-insert it from **Dynamic content** |
| In a test, the top half of the flow is **Skipped** and the run "starts" at RequesterName | **Respond to a PowerApp or flow** moved near the top. It's always skipped in manual tests, and everything below it gets skipped too | Move Respond to the **end** (Part 12.2) |
| **Respond** skipped: "The client application is not waiting…" | A manual test, with no parent waiting | Normal. It runs when called from Process HR Approval Reply |
| Related section missing; Block List shows `"keep": "no"` | Related Rows has 1 or fewer rows, or the Related row in Block List still says `"no"` | Check the Block List row (Part 9). Check **RelatedTitles** on the request (Part 5) and **Get Related**'s output |
| **RelatedTitles** empty on new requests | The agent node isn't mapped, or the title key is wrong | Part 5.5 (map `Global.RelatedLevels`, publish). Part 5.3 (`TITLEKEY`, object wrapper) |
| Every related row says "(This Job)" with the same code and grade | A Related Rows value still uses `outputs('JD')` | Change it to `item()` (Part 8.6) |
| The current job appears twice | **Related Others** isn't removing it: `JOBCODE_COL` is wrong | Use the JD list's job code internal name (Part 8.4) |
| Get Related errors on the filter | The title column isn't `Title`, or Status isn't a text/choice column named `Status` | Fix the internal names in 8.2 and 8.3 |
| `sort` error: "did not find the named sort field" | Sorting raw SharePoint items | Sort **Related Rows** by `'title'` (Part 10.2), not Related All |
| Grade shows as `{"Value": "10"}` | A Choice column | Add `?['Value']` to that Related Rows value |
| A Compose with `@citynational.com` errors | A leading `@` is read as an expression | Use `concat('@', 'citynational.com')` |
| HR marked **With Compensation** but no Comp email | Respond is set to run after **failed** too | Respond: run after **is successful** only (Part 12.2) |
| "Run a Child Flow" can't find Send Comp Review Email | It isn't in the solution, or it's off | Part 6.1 step 3, then turn it on |
| A child flow call fails on connections | Run-only connections aren't set | Part 12.3 |
| The email goes to the old address after changing the env var | The **Current value** overrides the default, or the flow has the old value cached | Update the current value. Turn the flow off and on |

---

## Appendix C: Removing the script-based lookup (if you built it)

If you built the earlier version that copied the Related Levels flow's script:
1. In **Send Comp Review Email**, delete **Get Related** (the copy with `Status eq 'Active'`), **Select titles**, **Run a script** and **Filter Related**. Then build Part 8 in their place. **Related Others**, **Related All** and **Related Rows** are rebuilt as in Part 8.
2. Flow details page → **Run only users** → remove or ignore **Excel Online (Business)**. It's no longer needed.
3. Re-run the Part 13 tests.
