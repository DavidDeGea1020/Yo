# Send To Compensation (Side Review): Build Guide

**What you'll build:** a side trip from the HR Ops step. HR Ops clicks **Send To Compensation**, and the Compensation team gets a full review email with HR Ops's notes. Comp has **one** option: **Return To HR Ops**, with their review comments. HR Ops then gets a short **Compensation Reviewed** email with Comp's comments and their usual three buttons.

The request **stays With HR Ops the whole time.** Comp never approves or declines anything, so the main HRBP → HR Ops path is unaffected.

```
HR Ops email ──[Send To Compensation]──► [OPS-TO-COMP] email to HRConnect (optional notes)
                                              ▼
                                  Process Ops To Comp
                                   checks: With HR Ops · no Comp review already open
                                   → Send Comp Side Review Email (child) → Comp team
                                   → CompReviewStatus = Requested · confirm to HR Ops
Comp email ──[Return To HR Ops]──► [COMP-RETURN] email to HRConnect (review comments, required)
                                              ▼
                                  Process Comp Return Reply
                                   checks: With HR Ops · Comp review is Requested · comments given
                                   → Compensation Reviewed email to HR Ops (+ original HR Ops email .eml)
                                   → CompReviewStatus = Completed · confirm to Comp
HR Ops then uses the usual buttons: Approve · Return To HR · Send To Compensation (again, if needed)
```

| Part | What | Where |
|---|---|---|
| 1 | Columns | SharePoint |
| 2 | Comp mailbox setting and permissions | Solution, SharePoint |
| 3 | The two templates (design brief and HTML agent prompts) | JD Templates |
| 4 | The Comp email child flow | Send Comp Side Review Email (copy of Send HR Ops Review Email) |
| 5 | The Send To Compensation flow | Process Ops To Comp (new) |
| 6 | The Return To HR Ops flow | Process Comp Return Reply (new) |
| 7 | End-to-end tests | — |

### Before you start
- **HR Ops Return and HRBP Options** is built, especially:
  - Part 3 (the HR Ops **Send To Compensation** button, `[OPS-TO-COMP]`);
  - Part 5 (**JD Email Archive** and the archived **Request NN - HR Ops Review.html**).
- The conventions are the same as the other guides: **Expression** tab, exact step names, **Run after** under **⋯ → Settings**.

---

## Part 1: SharePoint columns

**JD Update Requests** → **+ Add column**:

| Name | Type | Settings | Holds |
|---|---|---|---|
| `CompReviewStatus` | Single line of text | — | `Requested` or `Completed` |
| `CompRequestedBy` | Single line of text | — | HR Ops person who sent it |
| `CompRequestedOn` | Date and time | Include time **Yes** | When |
| `CompRequestNotes` | Multiple lines of text | Rich text **Off** | HR Ops's notes for Comp |
| `CompReviewedBy` | Single line of text | — | The Comp reviewer |
| `CompReviewedOn` | Date and time | Include time **Yes** | When |
| `CompReviewComments` | Multiple lines of text | Rich text **Off** | Comp's review |

`CompReviewStatus` is plain text, not Yes/No, which avoids the Update item reset quirk.

---

## Part 2: Compensation mailbox setting and permissions

1. **Solutions** → your solution → **+ New** → **More** → **Environment variable**:
   - **Display name:** `JD Compensation Team Mailbox`
   - **Name (schema):** let it generate, but make sure it's **different** from the old Compensation mailbox variable (now JD HR Ops Mailbox)
   - **Data type:** Text
   - **Default value:** the Comp team's address (in DEV and UAT, your own)
   - **Save**
2. **Documents library** (where the redlined docs live) → **Library settings** → **Permissions** → **Grant Permissions** → the **Compensation** group (security group or M365 group) → **Read**. This lets Open In Word and Download work for Comp.

Comp doesn't need access to **JD Email Archive**. The original HR Ops email is attached for HR Ops, not sent to Comp.

---

## Part 3: The two templates

The email designs are still to come. These are the agreed placeholders, so the flows in Parts 4–6 fill them correctly. Give each prompt to your HTML agent, then upload the results to **JD Templates** with the exact names.

### 3.1 JD Comp Side Review Email.html (to the Comp team)
A full review, built from the **HR Ops review email** template.

**Prompt for your HTML agent:**
```
Create "JD Comp Side Review Email.html" from the attached HR Ops review email (JD Compensation Review Email.html). Keep the design and every section; change only what is listed.

1. Header pill: "Confidential: HR Ops &amp; Compensation". Eyebrow: "COMPENSATION REVIEW REQUEST". H1: "HR Ops Requested A Compensation Review Of {{JobTitle}}". Sub-line: "Review The Classification, Related Levels And Changes Below, Then Return Your Review To HR Ops."
2. Add a section directly under the hero, before HR Approval: label "HR OPS REQUEST", subtitle "Requested By {{CompRequestedBy}} On {{CompRequestedDate}}.", followed by <!--BLOCK:CompRequestNotes--> a quote card labelled "HR OPS&#8217; NOTES" containing {{CompRequestNotes}} <!--/BLOCK:CompRequestNotes-->.
3. Replace the whole decision section (Approve / Return To HR / Send To Compensation and its guidance box) with "COMPENSATION REVIEW", subtitle "Send Your Review Back To HR Ops.", ONE pill button "Return To HR Ops" ({{ActionUrl_CompReturn}}, #1A1A1A fill, white text), and the help line {{CompReturnHelpText}} under it.
4. Remove {{ActionUrl_CompApprove}}, {{ActionUrl_CompDecline}}, {{ActionUrl_SendToComp}} and {{CompActionHelpText}} entirely.

Rules: do not rename, remove or duplicate any other {{placeholder}}, BLOCK or REPEAT marker; each marker appears once. New placeholders allowed: {{CompRequestedBy}}, {{CompRequestedDate}}, {{CompRequestNotes}}, {{ActionUrl_CompReturn}}, {{CompReturnHelpText}}, plus the block CompRequestNotes. Table layout, inline CSS, same MSO wrapper and mobile classes, ASCII-only.
Deliver the full file and a list of placeholders added and removed.
```

### 3.2 JD Comp Review Returned Email.html (to HR Ops)
A short email in the same style as **JD Returned To HRBP Email.html** and **JD Resent To HR Ops Email.html**.

**Prompt for your HTML agent:**
```
Create "JD Comp Review Returned Email.html", a short consolidated email in exactly the same style as the attached "JD Returned To HRBP Email.html" (same header, hero, quote cards, grey reference strip, pill buttons, footer).

Purpose: Compensation finished a review that HR Ops asked for, and sent it back to HR Ops. The original HR Ops review email is attached as a file.

Layout:
1. Hidden preheader {{Preheader}}.
2. Header pill "Compensation Reviewed".
3. Hero: eyebrow "JOB DESCRIPTION UPDATE REQUEST"; H1 "Compensation Reviewed {{JobTitle}}"; line "Compensation Sent Their Review Back To HR Ops. The Request Is Still With HR Ops."; meta "Request #{{RequestID}} &middot; Reviewed {{ReviewedDate}}".
4. Compensation's review (most prominent): label "COMPENSATION REVIEW", quote card "COMPENSATION&#8217;S WORDS" with {{CompReviewComments}}, "&mdash; {{CompReviewedBy}}" under it.
5. Your request (grey strip): label "YOUR REQUEST", subtitle "What HR Ops Asked Compensation.", quote card (italic) with {{CompRequestNotes}}, "&mdash; {{CompRequestedBy}}".
6. Original request (white): Requested By {{RequesterName}}, Sections Changed {{SectionsChanged}}; line "The original HR Ops review email is attached to this message."; link "Open The Redlined Document" to {{DocUrl}}.
7. HR Ops decision: three pill buttons Approve ({{ActionUrl_CompApprove}}, green #00843D), Return To HR ({{ActionUrl_CompDecline}}, white with red #B3261E border), Send To Compensation ({{ActionUrl_SendToComp}}, white with #1A1A1A border); help line {{CompActionHelpText}}.
8. Footer: "Request #{{RequestID}} &middot; Reviewed {{ReviewedDate}}" and the usual confidentiality line.

Rules: use only these placeholders; no BLOCK or REPEAT markers; no grade, job code, salary type or other classification data; table layout, inline CSS, same MSO wrapper and mobile classes, ASCII-only. Deliver the full file and the placeholder list.
```

---

## Part 4: Send Comp Side Review Email (child flow)

This is a copy of **Send HR Ops Review Email**, so it keeps the full data set Comp needs: classification, related levels, flags, the document and so on.

### 4.1 Make the copy
1. **Solutions** → your solution → **Cloud flows** → **Send HR Ops Review Email** → details page → **Save As** → `Send Comp Side Review Email` → **Save**.
2. **Solutions** → your solution → **Add existing** → **Automation** → **Cloud flow** → **Outside Dataverse** → tick it → **Add**.
3. Open **Send Comp Side Review Email** → **Edit**. Make every change below in the **copy** only.

### 4.2 HR approval details come from the request now
In the copy, the HRBP approval details are read from SharePoint rather than from trigger inputs:

| Compose | New expression |
|---|---|
| `HRApprovedBy` | `coalesce(body('Get_Request')?['HRApprovedBy'], '')` |
| `HRApprovalNotes` | `coalesce(body('Get_Request')?['HRApprovalNotes'], '')` |
| `HRApprovedDate` | `if(empty(body('Get_Request')?['HRApprovedOn']), '', convertFromUtc(body('Get_Request')?['HRApprovedOn'], 'Eastern Standard Time', 'MMMM d, yyyy'))` |

### 4.3 Inputs
1. Click the trigger → **+ Add an input** → **Text** → `CompRequestedBy`.
2. **+ Add an input** → **Text** → `CompRequestNotes` → **⋯** → **Make the field optional**.
3. Delete the old **HRApprovedBy** and **HRApprovalNotes** inputs (**⋯** → **Delete**). They're no longer referenced after 4.2. Keep **RequestID**.
4. Below the HRApprovedDate Compose, add three **Compose** actions:

| Name | Inputs |
|---|---|
| `CompRequestedBy` | **Dynamic content** → the trigger's **CompRequestedBy** |
| `CompRequestNotes` | **Expression:** `coalesce(`, insert the trigger's **CompRequestNotes**, `, '')` |
| `CompRequestedDate` | **Expression:** `convertFromUtc(utcNow(), 'Eastern Standard Time', 'MMMM d, yyyy')` |

### 4.4 Remove the HR Ops-only steps
These came over with the copy, but Comp emails don't use them:
1. **IsResend** → change its expression to `false`.
2. Delete the **Attach Original Ops** scope. **Send Comp Email**'s run after then points at **Attach Doc** again. Keep **is successful**, **has failed** and **is skipped**.
3. Delete the **Archive If First Send** condition. Otherwise Comp emails would overwrite the archived HR Ops email.
4. **Respond to a PowerApp or flow** stays **last**. **⋯** → **Settings** → **Run after** → for **Send Comp Email**, tick only **is successful**.

### 4.5 Recipient, template, subject, preheader
1. **CompMailbox** Compose → **Dynamic content** → **Environment variables** → **JD Compensation Team Mailbox**.
2. **Get Template** → **File Path** → **Expression** `'/JD Templates/JD Comp Side Review Email.html'`. Use the same folder spelling as before.
3. **Subject** →
```
concat('Compensation Review Requested: JD Update #', body('Get_Request')?['ID'], ' · ', body('Get_Request')?['Title'])
```
4. **Preheader** →
```
concat('HR Ops (', outputs('CompRequestedBy'), ') asked for a compensation review of ', body('Get_Request')?['Title'], '.')
```

### 4.6 Blocks
**Block List** → add the row (mind the commas):
```
  {"name": "CompRequestNotes", "keep": "@{if(empty(outputs('CompRequestNotes')), 'no', 'yes')}"}
```

### 4.7 Placeholders
**Token List**:
- **Delete** the rows `ActionUrl_CompApprove`, `ActionUrl_CompDecline`, `ActionUrl_SendToComp` and `CompActionHelpText`.
- **Add**:
```
  {"k": "CompRequestedBy",      "raw": "no",  "v": "@{outputs('CompRequestedBy')}"},
  {"k": "CompRequestedDate",    "raw": "no",  "v": "@{outputs('CompRequestedDate')}"},
  {"k": "CompRequestNotes",     "raw": "no",  "v": "@{outputs('CompRequestNotes')}"},
  {"k": "ActionUrl_CompReturn", "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[COMP-RETURN] JD Request #', body('Get_Request')?['ID'], ' - ', body('Get_Request')?['Title'])), '&body=', encodeUriComponent(concat('Your review for HR Ops (required). This goes to HR Ops.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "CompReturnHelpText",   "raw": "no",  "v": "Return To HR Ops opens an email for your review. Write it above the line and send; HR Ops receives it and decides the next step. The request stays with HR Ops."}
```
The sentence `This goes to HR Ops.` and the marker line are what Part 6 looks for.

### 4.8 Save, connections, test
1. **Save** → details page → **Run only users** → **Edit** → SharePoint, Office 365 Users and Office 365 Outlook → **Use this connection** → **Save** → **Turn on**.
2. With your own address in **JD Compensation Team Mailbox**: **Test** → **Manually** → RequestID (a request that's **With HR Ops**), CompRequestedBy `Your Name`, CompRequestNotes `Please confirm grade vs. Coordinator III.` → **Run flow**.
3. **Expect:**
   - the email has **HR Ops Request** with the notes, all the review content, and **one** Return To HR Ops button (`[COMP-RETURN] JD Request #NN - …`);
   - **Leftover Tokens** is false;
   - **Respond** is skipped, which is normal for a manual test.

---

## Part 5: Process Ops To Comp (the Send To Compensation flow)

### 5.1 Create the flow
1. **Solutions** → **+ New** → **Automation** → **Cloud flow** → **Automated** → **Name:** `Process Ops To Comp` → **Trigger:** **When a new email arrives in a shared mailbox (V2)** → **Create**.
2. **Original Mailbox Address:** `HRConnect@citynational.com` · **Folder:** Inbox · **Subject Filter:** `[OPS-TO-COMP]` · **Include Attachments:** No.
3. **Settings** → **Concurrency control** On → **Degree** `1`.
4. Use the same Outlook connection as your other reply flows.

### 5.2 Front half: copy from Process HR Approval Reply
Copy these from **Process HR Approval Reply**, unchanged (**⋯** → **Copy to my clipboard** → **+** → **Paste an action**), keeping their names:
**SenderEmail**, **AllowedDomain**, **RequestIdText**, **Internal sender**, **Has request number**, **Html to text**, **CommentsRaw**, **Comment Lines**, **Comments**, **Get Request**, **Request**.

- **CommentsRaw** works as is, because the Send To Compensation prompt also ends with `Leave blank if none.`
- In **Bounce No Number**, change the body to say "…nothing was sent to Compensation. Please click Send To Compensation in the HR Ops email again…".

### 5.3 Can it go to Comp?
1. **Compose** → rename `Problem`:
```
if(empty(outputs('Request')), concat('I couldn''t find request #', outputs('RequestIdText'), ', so nothing was sent.'), if(not(equals(coalesce(outputs('Request')?['RequestStatus']?['Value'], ''), 'With HR Ops')), concat('Request #', outputs('RequestIdText'), ' is ', coalesce(outputs('Request')?['RequestStatus']?['Value'], 'not with HR Ops'), ', so it can''t be sent to Compensation. Nothing was sent.'), if(equals(coalesce(outputs('Request')?['CompReviewStatus'], ''), 'Requested'), concat('Request #', outputs('RequestIdText'), ' is already with Compensation (sent by ', coalesce(outputs('Request')?['CompRequestedBy'], 'HR Ops'), '). Nothing was sent.'), '')))
```
2. **Condition** → `No problem` → `outputs('Problem')` **is equal to** (empty).
   - **False:** **Bounce Problem** (**Subject** `concat('Not sent to Compensation: JD Request #', outputs('RequestIdText'))`, **Body** `outputs('Problem')`) → **Terminate** (Succeeded).
   - **True:** leave empty.

### 5.4 Who sent it
1. **Get user profile (V2)** → `Get Ops Sender` → `outputs('SenderEmail')`
2. **Compose** → `OpsSenderName` → `coalesce(body('Get_Ops_Sender')?['displayName'], outputs('SenderEmail'))`. **Run after**: successful, failed, skipped, timed out.

### 5.5 Send the Comp email
1. **Flows** → **Run a Child Flow** → **Send Comp Side Review Email**:
   - **RequestID:** `int(outputs('RequestIdText'))`
   - **CompRequestedBy:** `outputs('OpsSenderName')`
   - **CompRequestNotes:** `outputs('Comments')`
2. Under it: **+** → **Add an action** → **SharePoint** → **Update item** → rename `Mark Comp Requested` (this is the left branch):

| Field | Value |
|---|---|
| Id | `int(outputs('RequestIdText'))` |
| Title | `outputs('Request')?['Title']` |
| RequestStatus Value | `outputs('Request')?['RequestStatus']?['Value']` (unchanged, still **With HR Ops**) |
| CompReviewStatus | `Requested` |
| CompRequestedBy | `outputs('OpsSenderName')` |
| CompRequestedOn | `utcNow()` |
| CompRequestNotes | `outputs('Comments')` |
| TitleReviewRequested | `outputs('Request')?['TitleReviewRequested']` (and every other Yes/No column at its current value) |

   Setting **RequestStatus** to its own current value stops the column from being reset to its default.
3. Hover between **Run a Child Flow** and **Mark Comp Requested** → **+** → **Add a parallel branch** → **Send an email from a shared mailbox (V2)** → `Bounce Comp Failed`:
   - **To:** `outputs('SenderEmail')`
   - **Subject:** `concat('Not sent to Compensation: JD Request #', outputs('RequestIdText'))`
   - **Body:** `The Compensation email couldn't be sent. Please try Send To Compensation again, or contact the JD agent owner if it keeps failing.`
   - **Run after** for Run a Child Flow: only **has failed** and **has timed out**.
   - Below it, in the same branch: **Terminate** (Failed).
4. Under **Mark Comp Requested** (left branch): **Send an email from a shared mailbox (V2)** → `Confirm To HR Ops`:
   - **To:** `outputs('SenderEmail')`
   - **Subject:** `concat('Sent to Compensation: JD Request #', outputs('RequestIdText'))`
   - **Body:** `concat('Request #', outputs('RequestIdText'), ' (', outputs('Request')?['Title'], ') was sent to Compensation for review. It stays with HR Ops; you will get their review by email.')`
5. Optional: **Move email (V2)** → **Processed Comp Requests**.
6. **Save** → turn it **on**.

---

## Part 6: Process Comp Return Reply (the Return To HR Ops flow)

### 6.1 Create the flow
**Automated cloud flow** → **Name:** `Process Comp Return Reply` → **Trigger:** **When a new email arrives in a shared mailbox (V2)** → HRConnect · Inbox · **Subject Filter:** `[COMP-RETURN]` → **Concurrency** 1 → the same Outlook connection.

### 6.2 Front half: copy from Process HR Ops Return Reply
Copy **SenderEmail**, **AllowedDomain**, **RequestIdText**, **Internal sender**, **Has request number**, **Html to text**, **CommentsRaw**, **Comment Lines**, **Comments**, **Get Request** and **Request**. Then:
- **CommentsRaw** → replace it with:
```
replace(replace(first(split(if(contains(body('Html_to_text'), 'This goes to HR Ops.'), split(body('Html_to_text'), 'This goes to HR Ops.')[1], body('Html_to_text')), '[Write above this line.')), decodeUriComponent('%0D'), ''), decodeUriComponent('%C2%A0'), ' ')
```
- **Bounce No Number** body → "…nothing was sent to HR Ops. Please click Return To HR Ops in the Compensation email again…".

### 6.3 Is a Comp review open?
1. **Compose** → `Problem`:
```
if(empty(outputs('Request')), concat('I couldn''t find request #', outputs('RequestIdText'), ', so nothing was sent.'), if(not(equals(coalesce(outputs('Request')?['RequestStatus']?['Value'], ''), 'With HR Ops')), concat('Request #', outputs('RequestIdText'), ' is ', coalesce(outputs('Request')?['RequestStatus']?['Value'], 'no longer with HR Ops'), ', so your review can''t be sent. Please contact HR Ops directly.'), if(not(equals(coalesce(outputs('Request')?['CompReviewStatus'], ''), 'Requested')), concat('There is no open compensation review for request #', outputs('RequestIdText'), '. Nothing was sent.'), if(less(length(outputs('Comments')), 15), 'Your review was empty or too short, so nothing was sent to HR Ops. Please click Return To HR Ops again and write your review above the line.', ''))))
```
   The comments requirement is the last check. To make comments optional, delete the `if(less(…), '…', '')` part and keep `''`, matching the closing brackets.
2. **Condition** `No problem` → **False:** **Bounce Problem** (**Subject** `concat('Review not sent: JD Request #', outputs('RequestIdText'))`, **Body** `outputs('Problem')`) → **Terminate** (Succeeded).

### 6.4 Details for the email
| # | Step | Settings |
|---|---|---|
| 1 | **Get user profile (V2)** → `Get Reviewer` | `outputs('SenderEmail')` |
| 2 | **Compose** → `ReviewerName` | `coalesce(body('Get_Reviewer')?['displayName'], outputs('SenderEmail'))`. Run after: the four boxes |
| 3 | **Get user profile (V2)** → `Get Requester` | `outputs('Request')?['RequestedBy']` |
| 4 | **Compose** → `RequesterName` | `coalesce(body('Get_Requester')?['displayName'], outputs('Request')?['RequestedBy'])`. Run after: the four boxes |
| 5 | **Compose** → `SiteUrl` | Your site URL (plain text) |
| 6 | **Get file metadata** → `Get Doc Metadata` | `outputs('Request')?['HRDocFileId']` |
| 7 | **Compose** → `DocUrl` | `if(empty(body('Get_Doc_Metadata')?['Path']), outputs('Request')?['{Link}'], concat(outputs('SiteUrl'), replace(encodeUriComponent(body('Get_Doc_Metadata')?['Path']), '%2F', '/'), '?web=1'))`. Run after: successful, failed, skipped |
| 8 | **Compose** → `ReviewedDate` | `convertFromUtc(utcNow(), 'Eastern Standard Time', 'MMMM d, yyyy')` |

### 6.5 Build the email
1. **Initialize variable** `Html` (String) and **Initialize variable** `Attachments` (Array, `[]`).
2. **Get file content using path** → `Get Template` → **JD Templates** → **JD Comp Review Returned Email.html**
3. **Set variable** **Html** → `if(contains(string(body('Get_Template')), '$content'), base64ToString(body('Get_Template')?['$content']), string(body('Get_Template')))`
4. **Compose** → `Token List`:
```json
[
  {"k": "Preheader",             "raw": "no",  "v": "@{concat('Compensation reviewed request #', outputs('RequestIdText'), ' (', outputs('Request')?['Title'], '). It is still with HR Ops.')}"},
  {"k": "JobTitle",              "raw": "no",  "v": "@{outputs('Request')?['Title']}"},
  {"k": "RequestID",             "raw": "no",  "v": "@{outputs('RequestIdText')}"},
  {"k": "ReviewedDate",          "raw": "no",  "v": "@{outputs('ReviewedDate')}"},
  {"k": "CompReviewComments",    "raw": "no",  "v": "@{outputs('Comments')}"},
  {"k": "CompReviewedBy",        "raw": "no",  "v": "@{outputs('ReviewerName')}"},
  {"k": "CompRequestNotes",      "raw": "no",  "v": "@{coalesce(outputs('Request')?['CompRequestNotes'], 'No notes were added.')}"},
  {"k": "CompRequestedBy",       "raw": "no",  "v": "@{coalesce(outputs('Request')?['CompRequestedBy'], '')}"},
  {"k": "RequesterName",         "raw": "no",  "v": "@{outputs('RequesterName')}"},
  {"k": "SectionsChanged",       "raw": "no",  "v": "@{coalesce(outputs('Request')?['SectionsChanged'], '')}"},
  {"k": "DocUrl",                "raw": "yes", "v": "@{outputs('DocUrl')}"},
  {"k": "ActionUrl_CompApprove", "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[OPS-APPROVE] JD Request #', outputs('RequestIdText'), ' - ', outputs('Request')?['Title'])), '&body=', encodeUriComponent(concat('Notes (optional). Leave blank if none.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "ActionUrl_CompDecline", "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[OPS-DECLINE] JD Request #', outputs('RequestIdText'), ' - ', outputs('Request')?['Title'])), '&body=', encodeUriComponent(concat('Why is this request being returned to HR? Your reasons go to HR.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "ActionUrl_SendToComp",  "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[OPS-TO-COMP] JD Request #', outputs('RequestIdText'), ' - ', outputs('Request')?['Title'])), '&body=', encodeUriComponent(concat('What should Compensation review? Your notes go to the Compensation team. Leave blank if none.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "CompActionHelpText",    "raw": "no",  "v": "Approve opens an email for optional notes and completes HR Ops review. Return To HR opens an email for your reasons, and the request goes back to HR. Send To Compensation asks Compensation to review again; the request stays with HR Ops."}
]
```
5. **Apply to each** `Apply Tokens` (concurrency **Off**) with **Token Step** and **Set variable Html** inside, exactly as in the HR Ops Return guide (6.8 step 6). Then **Leftover Tokens** below the loop.

### 6.6 Attach the original HR Ops email
**Scope** `Attach Original Ops` → inside it:
1. **Get file content using path** → `Get Original Ops` → `concat('/JD Email Archive/Request ', outputs('RequestIdText'), ' - HR Ops Review.html')`
2. **Compose** → `OriginalOpsHtml` → `if(contains(string(body('Get_Original_Ops')), '$content'), base64ToString(body('Get_Original_Ops')?['$content']), string(body('Get_Original_Ops')))`
3. **Compose** → `OpsEml` →
```
concat('From: HRConnect@citynational.com', decodeUriComponent('%0D%0A'), 'Subject: Original HR Ops Review - JD Request #', outputs('RequestIdText'), ' - ', outputs('Request')?['Title'], decodeUriComponent('%0D%0A'), 'MIME-Version: 1.0', decodeUriComponent('%0D%0A'), 'Content-Type: text/html; charset="utf-8"', decodeUriComponent('%0D%0A'), decodeUriComponent('%0D%0A'), outputs('OriginalOpsHtml'))
```
4. **Append to array variable** → **Attachments** → `{"Name": "Original HR Ops Review - Request @{outputs('RequestIdText')}.eml", "ContentBytes": "@{base64(outputs('OpsEml'))}"}`

### 6.7 Send, mark, confirm
1. Below the scope: **Send an email from a shared mailbox (V2)** → `Send Comp Review Returned`:
   - **Original Mailbox Address:** HRConnect · **To:** **JD HR Ops Mailbox** (env var)
   - **Subject:** `concat('Compensation reviewed: JD Request #', outputs('RequestIdText'), ' · ', outputs('Request')?['Title'])`
   - **Body:** **</>** → only **Html**
   - **Attachments:** entire array → `variables('Attachments')`
   - **Run after** for Attach Original Ops: **is successful**, **has failed** and **is skipped**
2. **Update item** → `Mark Comp Reviewed`:

| Field | Value |
|---|---|
| Id | `int(outputs('RequestIdText'))` |
| Title | `outputs('Request')?['Title']` |
| RequestStatus Value | `outputs('Request')?['RequestStatus']?['Value']` (unchanged) |
| CompReviewStatus | `Completed` |
| CompReviewedBy | `outputs('ReviewerName')` |
| CompReviewedOn | `utcNow()` |
| CompReviewComments | `outputs('Comments')` |
| TitleReviewRequested | `outputs('Request')?['TitleReviewRequested']` (and every other Yes/No column) |

3. **Send an email from a shared mailbox (V2)** → `Confirm To Comp`:
   - **To:** `outputs('SenderEmail')`
   - **Subject:** `concat('Review sent to HR Ops: JD Request #', outputs('RequestIdText'))`
   - **Body:** `concat('Your review of request #', outputs('RequestIdText'), ' (', outputs('Request')?['Title'], ') was sent to HR Ops.')`
4. Optional: **Move email (V2)** → **Processed Comp Reviews**.
5. **Save** → turn it **on**.

---

## Part 7: End-to-end tests

Point **JD Compensation Team Mailbox** and **JD HR Ops Mailbox** at addresses you can watch. Use a fresh request that has reached **With HR Ops** (submitted after the archive parts, so the HR Ops email is archived).

| # | Do | Expect |
|---|---|---|
| 1 | HR Ops email → **Send To Compensation** (with notes) | The Comp email arrives with HR Ops's notes, the full review content, and one **Return To HR Ops** button. **CompReviewStatus = Requested**, with CompRequested… filled. HR Ops gets "Sent to Compensation". Status stays **With HR Ops** |
| 2 | **Send To Compensation** again for the same request | Bounce: "…is already with Compensation…" |
| 3 | Comp email → **Return To HR Ops** with no text | Bounce: "Your review was empty or too short…" |
| 4 | Comp email → **Return To HR Ops** with a review | **Compensation Reviewed** reaches HR Ops: Comp's review, HR Ops's original notes, the details, the **Original HR Ops Review .eml** and the 3 buttons. **CompReviewStatus = Completed**, with CompReviewed… filled. Comp gets a confirmation |
| 5 | Comp clicks **Return To HR Ops** again on the same email | Bounce: "There is no open compensation review…" |
| 6 | From the Compensation Reviewed email → **Send To Compensation** | A new Comp email goes out (allowed again because the last review is Completed) |
| 7 | While Comp has it (Requested), HR Ops clicks **Return To HR** | The request returns to the HRBPs. When Comp later replies → bounce: "…is Returned to HR, so your review can't be sent…" |
| 8 | **Send To Compensation** for a request that's **Submitted** | Bounce: "…can't be sent to Compensation…" |
| 9 | HR Ops **Approve** from any email after a Comp review | Not processed yet (that's the JDXpert step, still to come) |

**In each run:** **Leftover Tokens** is false, bounces end at **Terminate (Succeeded)**, child runs end with **Respond** green, and **RequestStatus** never changes in this side flow.
