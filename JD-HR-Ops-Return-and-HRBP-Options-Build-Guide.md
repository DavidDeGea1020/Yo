# HR Ops Return and HRBP Options: Build Guide

**What you'll build:** HR Ops can **return** a request to the HRBPs. The HRBPs get a short **Returned To HRBP** email, with HR Ops's comments and the original HRBP review email attached. From that email, the HRBPs choose one of two options:

- **Resend To HR Ops:** HR Ops gets a short **Resent To HR Ops** email, with the HRBP's notes, their own return comments and the original HR Ops email attached, plus their usual three buttons.
- **Decline To Manager:** the manager gets the usual decline email, and HR Ops gets a short **Declined By HRBP** notice that the request is closed.

**Who's who in this guide:**
- **HRBPs:** the first reviewers. Mailbox setting **JD HR Review Mailbox**.
- **HR Ops:** the second reviewers. Mailbox setting **JD HR Ops Mailbox** (formerly JD Compensation Mailbox).
- Every reply goes to **HRConnect@citynational.com**, where the flows pick it up by subject tag.

```
HRBP review email ──[Approve]──► Process HR Approval Reply ──► HR Ops review email (status With HR Ops)
                                                                     │
                                                         [Return To HR]  ([OPS-DECLINE])
                                                                     ▼
                                        Process HR Ops Return Reply ──► Returned To HRBP email (+ original HRBP email .eml)
                                                                     │   status Returned to HR
                         ┌───────────────────────────────────────────┴───────────────────────────┐
              [Resend To HR Ops] ([APPROVE])                                         [Decline To Manager] ([DECLINE])
                         ▼                                                                        ▼
        Process HR Approval Reply (existing)                                     Process HR Decline Reply (existing)
        → Send HR Ops Review Email in "resend" mode                              → decline email to the manager
        → Resent To HR Ops email (+ original HR Ops email .eml)                  → Declined By HRBP notice to HR Ops
          status With HR Ops                                                       status Declined
```

| Part | What | Where |
|---|---|---|
| 1 | Columns, statuses, the archive library | SharePoint |
| 2 | Upload the three new templates | JD Templates |
| 3 | HR Ops email buttons and help text | Send HR Ops Review Email |
| 4 | Archive every HRBP review email | Send HR Review Email |
| 5 | Archive every HR Ops email, plus "resend" mode | Send HR Ops Review Email |
| 6 | The return flow | Process HR Ops Return Reply (new) |
| 7 | HRBP option A: Resend To HR Ops | Process HR Approval Reply |
| 8 | HRBP option B: Decline To Manager (+ notice to HR Ops) | Process HR Decline Reply |
| 9 | End-to-end tests | — |

### Conventions
- **Expression:** click the field → **Expression** tab (**fx**) → paste → **Add**.
- **Rename** a step by clicking its title. Use the names **exactly** as written, because expressions refer to them by name, with spaces turned into underscores (`Get Request` → `body('Get_Request')`).
- **Run after:** on a step → **⋯** → **Settings** (or **Configure run after**).
- **The newline helper** `decodeUriComponent('%0A')` is a line break, and `decodeUriComponent('%0D%0A')` is a Windows line break.

### Before you start
- These flows exist and work: **Send HR Review Email** (HRBP child), **Process HR Decline Reply**, **Process HR Approval Reply** and **Send HR Ops Review Email** (the former Comp child).
- The account behind your **Office 365 Outlook** connection has **Full Access** and **Send As** on HRConnect.
- **JD HR Review Mailbox** (HRBPs) and **JD HR Ops Mailbox** (HR Ops) are set. In DEV and UAT, point them at addresses you can watch.

---

## Part 1: SharePoint

### 1.1 Columns on JD Update Requests
**+ Add column** for each:

| Name | Type | Settings |
|---|---|---|
| `OpsReturnedBy` | Single line of text | — |
| `OpsReturnedOn` | Date and time | **Include time:** Yes |
| `OpsReturnComments` | Multiple lines of text | **Use enhanced rich text:** Off |

### 1.2 Statuses
**RequestStatus** → **Column settings** → **Edit** → under **Choices**, make sure these exist: `Submitted`, `With HR Ops`, `Returned to HR`, `Declined` → **Save**.

### 1.3 The archive library (HR-only)
The archived emails contain restricted classification data, so they get their own locked-down library.
1. Site → **+ New** → **Document library** → **Name:** `JD Email Archive` → **Create**.
2. **Gear** → **Library settings** → **Permissions for this document library** → **Stop Inheriting Permissions** → **OK**.
3. Remove groups that include non-HR staff (for example, the site Visitors or Members groups, if they're broad).
4. **Grant Permissions** → the **HRBP** and **HR Ops** groups (security groups or M365 groups, not plain distribution lists) → **Read** → **Share**.
5. Check that the flow's SharePoint connection account keeps **Edit** (as owner, it does).

---

## Part 2: Upload the templates

Upload each of these to **JD Templates** (**Upload** → **Files**), keeping the names exactly:

| File | Sent by | To |
|---|---|---|
| **JD Returned To HRBP Email.html** | Process HR Ops Return Reply | HRBPs |
| **JD Resent To HR Ops Email.html** | Send HR Ops Review Email (resend mode) | HR Ops |
| **JD Declined By HRBP Notice.html** | Process HR Decline Reply | HR Ops |

Don't open or save them in SharePoint or Word. Repeat in each environment.

---

## Part 3: HR Ops email buttons and help text

The HR Ops email's three buttons each send a tagged email to HRConnect.

1. **Send HR Ops Review Email** → **Edit** → **Token List**.
2. Replace the `ActionUrl_CompApprove`, `ActionUrl_CompDecline` and `CompActionHelpText` rows, and add `ActionUrl_SendToComp`, so these four rows read:
```
  {"k": "ActionUrl_CompApprove", "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[OPS-APPROVE] JD Request #', body('Get_Request')?['ID'], ' - ', body('Get_Request')?['Title'])), '&body=', encodeUriComponent(concat('Notes (optional). Leave blank if none.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "ActionUrl_CompDecline", "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[OPS-DECLINE] JD Request #', body('Get_Request')?['ID'], ' - ', body('Get_Request')?['Title'])), '&body=', encodeUriComponent(concat('Why is this request being returned to HR? Your reasons go to HR.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "ActionUrl_SendToComp",  "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[OPS-TO-COMP] JD Request #', body('Get_Request')?['ID'], ' - ', body('Get_Request')?['Title'])), '&body=', encodeUriComponent(concat('What should Compensation review? Your notes go to the Compensation team. Leave blank if none.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "CompActionHelpText",    "raw": "no",  "v": "Approve opens an email for optional notes and completes HR Ops review. Return To HR opens an email for your reasons, and the request goes back to HR. Send To Compensation opens an email for your notes; Compensation reviews the request and replies to HR Ops, and the request stays with HR Ops."}
```
   Put commas between rows, with none after the last row.
3. **Save**.

The sentences `Your reasons go to HR.` and `Leave blank if none.`, and the line `[Write above this line.`, are what the reply flows look for. Keep them exactly as written.

---

## Part 4: Archive every HRBP review email

The Returned To HRBP email attaches the HRBP review email exactly as it was sent. For that to work, each one is saved when it goes out.

1. **Send HR Review Email** → **Edit** → click the **+** directly **below Send HR Email** → **SharePoint** → **Create file** → rename `Archive HRBP Email`:
   - **Site Address:** your site · **Folder Path:** folder icon → **JD Email Archive**
   - **File Name** → **Expression**:
   ```
   concat('Request ', body('Get_Request')?['ID'], ' - HRBP Review.html')
   ```
   - **File Content** → **Dynamic content** → **Variables** → **Html**
2. **Archive HRBP Email** → **⋯** → **Settings** → **Run after** → for **Send HR Email**, tick only **is successful**.
3. **+** below it → **Compose** → rename `Archive Done` → **Inputs:** `done` → **⋯** → **Settings** → **Run after** → for **Archive HRBP Email**, tick **is successful**, **has failed** and **is skipped**.

   This keeps the run **Succeeded** whenever the email went out, even if saving the copy fails.
4. **Save**.

**Check:** submit a fresh request through the agent. **JD Email Archive** should now have **Request NN - HRBP Review.html**, and it opens in the browser looking like the email.

---

## Part 5: Archive every HR Ops email, plus "resend" mode

The same child flow sends both the first HR Ops email and the **Resent To HR Ops** email. It tells them apart by the request's status at the moment it runs: **Returned to HR** means this is a resend from the HRBPs.

### 5.1 Is this a resend?
1. **Send HR Ops Review Email** → **Edit** → click the **+** directly **below Get Request** → **Compose** → rename `IsResend` → **Expression**:
```
equals(coalesce(body('Get_Request')?['RequestStatus']?['Value'], ''), 'Returned to HR')
```

### 5.2 HR Ops's last return, for the Resent email
Below **IsResend**: **+** → **Compose** → rename `OpsReturnedDate` → **Expression**:
```
if(empty(body('Get_Request')?['OpsReturnedOn']), '', convertFromUtc(body('Get_Request')?['OpsReturnedOn'], 'Eastern Standard Time', 'MMMM d, yyyy'))
```

### 5.3 Pick the template
1. Open **Get Template** and note the current **File Path**, e.g. `/JD Templates/JD Compensation Review Email.html`.
2. Click into **File Path** → delete it → **Expression**:
```
if(outputs('IsResend'), '/JD Templates/JD Resent To HR Ops Email.html', '/JD Templates/JD Compensation Review Email.html')
```
   Use the exact folder spelling from step 1. If your HR Ops template has a different file name, use it in the second path.

### 5.4 Subject and preheader
1. **Subject** → wrap the current expression like this (the part after the comma is your existing subject):
```
if(outputs('IsResend'), concat('Resent by HRBP: JD Update #', body('Get_Request')?['ID'], ' · ', body('Get_Request')?['Title']), concat(if(greater(length(outputs('FlagChips')), 0), '[Flagged] ', ''), 'HR Ops Review: JD Update #', body('Get_Request')?['ID'], ' · ', body('Get_Request')?['Title']))
```
2. **Preheader** →
```
if(outputs('IsResend'), concat('HRBP resent request #', body('Get_Request')?['ID'], ' (', body('Get_Request')?['Title'], ') after your return.'), concat('HR approved by ', outputs('HRApprovedBy'), ' · ', body('Get_Request')?['Title'], ' · ', if(greater(length(outputs('FlagChips')), 0), concat(length(outputs('FlagChips')), ' items flagged'), 'No flags raised')))
```

### 5.5 New placeholders
In **Token List**, add these three rows before the closing `]`:
```
  {"k": "OpsReturnComments", "raw": "no", "v": "@{coalesce(body('Get_Request')?['OpsReturnComments'], 'No reasons were given.')}"},
  {"k": "OpsReturnedBy",     "raw": "no", "v": "@{coalesce(body('Get_Request')?['OpsReturnedBy'], '')}"},
  {"k": "OpsReturnedDate",   "raw": "no", "v": "@{outputs('OpsReturnedDate')}"}
```
The Resent template also uses **HRApprovedBy**, **HRApprovedDate**, **HRApprovalNotes** (and its block), **RequesterName**, the **Section** chips, **DocViewUrl** (in the DocOk block) and the three HR Ops buttons. Those already exist in this flow, so nothing else is needed. In the normal HR Ops email, these three new rows find nothing to replace, which is harmless.

### 5.6 Attach the original HR Ops email on a resend
1. Click the **+** directly **below the Attach Doc scope** → **Control** → **Scope** → rename `Attach Original Ops`.
2. Inside the scope: **+** → **Condition** → left: **Expression** `outputs('IsResend')` → **is equal to** → right: **Expression** `true`.
3. In **True**, add in order:
   1. **SharePoint** → **Get file content using path** → rename `Get Original Ops` → **Site Address:** your site → **File Path** → **Expression**:
   ```
   concat('/JD Email Archive/Request ', body('Get_Request')?['ID'], ' - HR Ops Review.html')
   ```
   2. **Compose** → rename `OriginalOpsHtml`:
   ```
   if(contains(string(body('Get_Original_Ops')), '$content'), base64ToString(body('Get_Original_Ops')?['$content']), string(body('Get_Original_Ops')))
   ```
   3. **Compose** → rename `OpsEml`:
   ```
   concat('From: HRConnect@citynational.com', decodeUriComponent('%0D%0A'), 'Subject: Original HR Ops Review - JD Request #', body('Get_Request')?['ID'], ' - ', body('Get_Request')?['Title'], decodeUriComponent('%0D%0A'), 'MIME-Version: 1.0', decodeUriComponent('%0D%0A'), 'Content-Type: text/html; charset="utf-8"', decodeUriComponent('%0D%0A'), decodeUriComponent('%0D%0A'), outputs('OriginalOpsHtml'))
   ```
   4. **Append to array variable** → **Attachments** → **Value:**
   ```json
   {
     "Name": "Original HR Ops Review - Request @{body('Get_Request')?['ID']}.eml",
     "ContentBytes": "@{base64(outputs('OpsEml'))}"
   }
   ```
4. **Send Comp Email**, the step now directly after the scope → **⋯** → **Settings** → **Run after** → for **Attach Original Ops**, tick **is successful**, **has failed** and **is skipped**. If there's no archived copy (older requests), the email still goes out.

### 5.7 Archive the first HR Ops email
1. Click the **+** directly **below Send Comp Email** (the send step) → **Condition** → rename `Archive If First Send` → left: **Expression** `outputs('IsResend')` → **is equal to** → right: **Expression** `false`.
2. In **True**: **SharePoint** → **Create file** → rename `Archive HR Ops Email`:
   - **Folder Path:** **JD Email Archive**
   - **File Name** → **Expression** `concat('Request ', body('Get_Request')?['ID'], ' - HR Ops Review.html')`
   - **File Content** → **Html** variable
3. **Respond to a PowerApp or flow** must stay the **last** step, below this condition. **⋯** → **Settings** → **Run after** → for **Archive If First Send**, tick **is successful** and **has failed**.

   If the email itself fails, the condition is skipped, so Respond doesn't run and the child ends as failed. The parent then bounces to the HRBP.
4. **Save**.

Only the **first** HR Ops email is archived, so the attachment on every resend is always the original review.

---

## Part 6: Process HR Ops Return Reply

### 6.1 Create the flow
1. **Solutions** → your solution → **+ New** → **Automation** → **Cloud flow** → **Automated**.
2. **Name:** `Process HR Ops Return Reply` · **Trigger:** **When a new email arrives in a shared mailbox (V2)** → **Create**.
3. Trigger:
   - **Original Mailbox Address:** `HRConnect@citynational.com`
   - **Folder:** Inbox
   - **Subject Filter:** `[OPS-DECLINE]`
   - **Include Attachments:** No
4. **⋯** → **Settings** → **Concurrency control:** On → **Degree:** `1` → **Done**.
5. Make sure the trigger uses the **same Outlook connection** as Process HR Approval Reply. A different connection without access to HRConnect causes a trigger error.

You can copy steps 6.2–6.5 from **Process HR Decline Reply** (**⋯** → **Copy to my clipboard** → **+** → **Paste an action**). Keep the names, and apply the changes noted.

### 6.2 Basic values
Three **Compose** actions:

| Name | Expression |
|---|---|
| `SenderEmail` | `toLower(triggerOutputs()?['body/from'])` |
| `AllowedDomain` | `concat('@', 'citynational.com')` |
| `RequestIdText` | `first(split(trim(last(split(coalesce(triggerOutputs()?['body/subject'], ''), 'JD Request #'))), ' '))` |

### 6.3 Guards
1. **Condition** → rename `Internal sender` → `endsWith(outputs('SenderEmail'), toLower(outputs('AllowedDomain')))` **is equal to** `true`. In **False**: **Terminate** (Succeeded).
2. **Condition** → rename `Has request number` → `isInt(outputs('RequestIdText'))` **is equal to** `true`. In **False**:
   - **Send an email from a shared mailbox (V2)** → rename `Bounce No Number` → **Original Mailbox Address:** HRConnect · **To:** `outputs('SenderEmail')` · **Subject:** `Return not processed` · **Body:** `I couldn't find a request number in your email's subject, so nothing was sent to the HRBPs. Please click Return To HR in the HR Ops email again and keep the subject as it is.`
   - **Terminate** (Succeeded).

### 6.4 HR Ops's comments (optional)
1. **Content Conversion** → **Html to text** → **Content:** `triggerOutputs()?['body/body']`
2. **Compose** → rename `CommentsRaw`:
```
replace(replace(first(split(if(contains(body('Html_to_text'), 'Your reasons go to HR.'), split(body('Html_to_text'), 'Your reasons go to HR.')[1], body('Html_to_text')), '[Write above this line.')), decodeUriComponent('%0D'), ''), decodeUriComponent('%C2%A0'), ' ')
```
3. **Select** → rename `Comment Lines` → **From:** `split(outputs('CommentsRaw'), decodeUriComponent('%0A'))` → **Map** in text mode → `trim(item())`
4. **Compose** → rename `Comments`:
```
trim(replace(replace(replace(replace(replace(replace(replace(replace(replace(join(body('Comment_Lines'), decodeUriComponent('%0A')), concat(decodeUriComponent('%0A'), decodeUriComponent('%0A')), '~~P~~'), concat(decodeUriComponent('%0A'), '-'), '~~B~~-'), concat(decodeUriComponent('%0A'), '*'), '~~B~~*'), concat(decodeUriComponent('%0A'), decodeUriComponent('%E2%80%A2')), concat('~~B~~', decodeUriComponent('%E2%80%A2'))), decodeUriComponent('%0A'), ' '), '~~B~~', decodeUriComponent('%0A')), '~~P~~', concat(decodeUriComponent('%0A'), decodeUriComponent('%0A'))), concat(decodeUriComponent('%0A'), ' '), decodeUriComponent('%0A')), '  ', ' '))
```

### 6.5 The request
1. **SharePoint** → **Get items** → rename `Get Request` → **JD Update Requests** → **Filter Query:** `ID eq ` then **Expression** `outputs('RequestIdText')` → **Top Count:** `1`
2. **Compose** → rename `Request` → `first(body('Get_Request')?['value'])`

### 6.6 Is it returnable?
1. **Compose** → rename `Problem`:
```
if(empty(outputs('Request')), concat('I couldn''t find request #', outputs('RequestIdText'), ', so nothing was sent.'), if(not(contains(createArray('With HR Ops', 'With Compensation'), coalesce(outputs('Request')?['RequestStatus']?['Value'], ''))), concat('Request #', outputs('RequestIdText'), ' is ', coalesce(outputs('Request')?['RequestStatus']?['Value'], 'not with HR Ops'), ', so it can''t be returned from this email. Nothing was sent.'), ''))
```
   Problem is empty when everything checks out. Otherwise it holds the reason. "With Compensation" is there only for older test requests, and comments are optional.
2. **Condition** → rename `No problem` → `outputs('Problem')` **is equal to** (leave the right side empty).
3. **False**:
   - **Send an email from a shared mailbox (V2)** → rename `Bounce Problem` → **Original Mailbox Address:** HRConnect · **To:** `outputs('SenderEmail')` · **Subject:** `concat('Return not processed: JD Request #', outputs('RequestIdText'))` · **Body:** `outputs('Problem')`
   - **Terminate** (Succeeded).
4. Leave **True** empty. Everything after this goes **below** the condition.

### 6.7 Details for the email
| # | Step | Settings |
|---|---|---|
| 1 | **Get user profile (V2)** → `Get Returner` | **User (UPN):** `outputs('SenderEmail')` |
| 2 | **Compose** → `ReturnedByName` | `coalesce(body('Get_Returner')?['displayName'], outputs('SenderEmail'))`. **Run after** Get Returner: successful, failed, skipped, timed out |
| 3 | **Compose** → `ReturnComments` | `if(empty(outputs('Comments')), 'No reasons were given.', outputs('Comments'))` |
| 4 | **Get user profile (V2)** → `Get Requester` | **User (UPN):** `outputs('Request')?['RequestedBy']` |
| 5 | **Compose** → `RequesterName` | `coalesce(body('Get_Requester')?['displayName'], outputs('Request')?['RequestedBy'])`. **Run after** Get Requester: the same four |
| 6 | **Compose** → `SiteUrl` | Your site URL as plain text, with no trailing slash |
| 7 | **SharePoint → Get file metadata** → `Get Doc Metadata` | **File Identifier:** `outputs('Request')?['HRDocFileId']` |
| 8 | **Compose** → `DocUrl` | `if(empty(body('Get_Doc_Metadata')?['Path']), outputs('Request')?['{Link}'], concat(outputs('SiteUrl'), replace(encodeUriComponent(body('Get_Doc_Metadata')?['Path']), '%2F', '/'), '?web=1'))`. **Run after** Get Doc Metadata: successful, failed, skipped |
| 9 | **Compose** → `ReturnedDate` | `convertFromUtc(utcNow(), 'Eastern Standard Time', 'MMMM d, yyyy')` |
| 10 | **Compose** → `HRApprovedDate` | `if(empty(outputs('Request')?['HRApprovedOn']), '', convertFromUtc(outputs('Request')?['HRApprovedOn'], 'Eastern Standard Time', 'MMMM d, yyyy'))` |

### 6.8 Build the email
1. **Initialize variable** → `Html` (String, empty)
2. **Initialize variable** → `Attachments` (Array, `[]`)
3. **Get file content using path** → rename `Get Template` → **JD Templates** → **JD Returned To HRBP Email.html**
4. **Set variable** → **Html** →
```
if(contains(string(body('Get_Template')), '$content'), base64ToString(body('Get_Template')?['$content']), string(body('Get_Template')))
```
5. **Compose** → rename `Token List`:
```json
[
  {"k": "Preheader",         "raw": "no",  "v": "@{concat('HR Ops returned request #', outputs('RequestIdText'), ' (', outputs('Request')?['Title'], '). Decline it to the manager or resend it to HR Ops.')}"},
  {"k": "JobTitle",          "raw": "no",  "v": "@{outputs('Request')?['Title']}"},
  {"k": "RequestID",         "raw": "no",  "v": "@{outputs('RequestIdText')}"},
  {"k": "ReturnedDate",      "raw": "no",  "v": "@{outputs('ReturnedDate')}"},
  {"k": "CompComments",      "raw": "no",  "v": "@{outputs('ReturnComments')}"},
  {"k": "ReturnedByName",    "raw": "no",  "v": "@{outputs('ReturnedByName')}"},
  {"k": "RequesterName",     "raw": "no",  "v": "@{outputs('RequesterName')}"},
  {"k": "SectionsChanged",   "raw": "no",  "v": "@{coalesce(outputs('Request')?['SectionsChanged'], '')}"},
  {"k": "HRApprovedBy",      "raw": "no",  "v": "@{coalesce(outputs('Request')?['HRApprovedBy'], '')}"},
  {"k": "HRApprovedDate",    "raw": "no",  "v": "@{outputs('HRApprovedDate')}"},
  {"k": "DocUrl",            "raw": "yes", "v": "@{outputs('DocUrl')}"},
  {"k": "ActionUrl_Decline", "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[DECLINE] JD Request #', outputs('RequestIdText'), ' - ', outputs('Request')?['Title'])), '&body=', encodeUriComponent(concat('Why is this request being declined, and what needs to change? This goes to the manager word for word.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"},
  {"k": "ActionUrl_Resend",  "raw": "yes", "v": "@{concat('mailto:HRConnect@citynational.com?subject=', encodeUriComponent(concat('[APPROVE] JD Request #', outputs('RequestIdText'), ' - ', outputs('Request')?['Title'])), '&body=', encodeUriComponent(concat('Notes for HR Ops (optional). Leave blank if none.', decodeUriComponent('%0D%0A%0D%0A%0D%0A%0D%0A'), '[Write above this line. Anything below is ignored.]')))}"}
]
```
   The two button links are exactly the HRBP email's Decline and Approve mailtos, so the existing HRBP flows handle them.
6. **Apply to each** → rename `Apply Tokens` → **From:** `outputs('Token_List')` → **Settings** → **Concurrency Control: Off**. Inside it:
   - **Compose** → `Token Step`:
   ```
   replace(variables('Html'), concat('{{', item()?['k'], '}}'), if(equals(item()?['raw'], 'yes'), string(item()?['v']), replace(replace(replace(replace(replace(string(item()?['v']), '&', '&amp;'), '<', '&lt;'), '>', '&gt;'), decodeUriComponent('%0D'), ''), decodeUriComponent('%0A'), '<br>')))
   ```
   - **Set variable** → **Html** → `outputs('Token_Step')`
7. Below the loop: **Compose** → rename `Leftover Tokens` → `contains(variables('Html'), '{{')`

### 6.9 Attach the original HRBP email
1. **Scope** → rename `Attach Original`. Add these inside it:
   1. **Get file content using path** → rename `Get Original` → **File Path** →
   ```
   concat('/JD Email Archive/Request ', outputs('RequestIdText'), ' - HRBP Review.html')
   ```
   2. **Compose** → rename `OriginalHtml` →
   ```
   if(contains(string(body('Get_Original')), '$content'), base64ToString(body('Get_Original')?['$content']), string(body('Get_Original')))
   ```
   3. **Compose** → rename `Eml` →
   ```
   concat('From: HRConnect@citynational.com', decodeUriComponent('%0D%0A'), 'Subject: Original HRBP Review - JD Request #', outputs('RequestIdText'), ' - ', outputs('Request')?['Title'], decodeUriComponent('%0D%0A'), 'MIME-Version: 1.0', decodeUriComponent('%0D%0A'), 'Content-Type: text/html; charset="utf-8"', decodeUriComponent('%0D%0A'), decodeUriComponent('%0D%0A'), outputs('OriginalHtml'))
   ```
   4. **Append to array variable** → **Attachments** →
   ```json
   {
     "Name": "Original HRBP Review - Request @{outputs('RequestIdText')}.eml",
     "ContentBytes": "@{base64(outputs('Eml'))}"
   }
   ```

### 6.10 Send, mark, confirm
1. **Below the scope:** **Send an email from a shared mailbox (V2)** → rename `Send Returned Email`:
   - **Original Mailbox Address:** HRConnect · **To:** **JD HR Review Mailbox** (env var)
   - **Subject:** `concat('Returned by HR Ops: JD Request #', outputs('RequestIdText'), ' · ', outputs('Request')?['Title'])`
   - **Body:** **</>** code view → only the **Html** variable
   - **Attachments** → **Switch to input entire array** → `variables('Attachments')`
   - **Run after** for **Attach Original**: **is successful**, **has failed** and **is skipped**.
2. **Update item** → rename `Mark Returned To HR`:

| Field | Value |
|---|---|
| Id | `int(outputs('RequestIdText'))` |
| Title | `outputs('Request')?['Title']` |
| RequestStatus Value | `Returned to HR` |
| OpsReturnedBy | `outputs('ReturnedByName')` |
| OpsReturnedOn | `utcNow()` |
| OpsReturnComments | `outputs('ReturnComments')` |
| TitleReviewRequested | `outputs('Request')?['TitleReviewRequested']` (and every other Yes/No column at its current value) |

3. **Send an email from a shared mailbox (V2)** → rename `Confirm To HR Ops` → **To:** `outputs('SenderEmail')` · **Subject:** `concat('Returned to HRBPs: JD Request #', outputs('RequestIdText'))` · **Body:** `concat('Request #', outputs('RequestIdText'), ' (', outputs('Request')?['Title'], ') was returned to the HRBPs. They will decline it to the manager or resend it to HR Ops.')`
4. Optional: **Move email (V2)** → `triggerOutputs()?['body/id']` → **Processed Returns** folder in HRConnect.
5. **Save** → turn the flow **on**.

---

## Part 7: HRBP option A, Resend To HR Ops

**Resend To HR Ops** in the Returned email sends the same `[APPROVE]` email as the HRBP review email's Approve button. That means **Process HR Approval Reply** handles it, and it already accepts **Returned to HR**. It calls **Send HR Ops Review Email**, which sees status **Returned to HR**, switches to **resend mode** (Part 5), and sends the **Resent To HR Ops** email with the original HR Ops email attached. Then the approval flow marks the request **With HR Ops**.

The only change is the HRBP's confirmation wording:
1. **Process HR Approval Reply** → **Confirm To HR** → **Body** →
```
concat('Request #', outputs('RequestIdText'), ' (', outputs('Request')?['Title'], ') was ', if(equals(outputs('Request')?['RequestStatus']?['Value'], 'Returned to HR'), 'resent', 'approved and sent'), ' to HR Ops.', if(empty(outputs('Comments')), '', ' Your notes were included.'))
```
2. Check that its **Problem** allowlist reads `createArray('Submitted', 'Returned to HR')`.
3. **Save**.

`outputs('Request')` still holds the status from **before** the update, so "Returned to HR" correctly identifies a resend.

---

## Part 8: HRBP option B, Decline To Manager (+ notice to HR Ops)

**Decline To Manager** sends the usual `[DECLINE]` email, so **Process HR Decline Reply** sends the manager the normal decline email. Add the allowlist check and the HR Ops notice.

### 8.1 Allowlist
**Process HR Decline Reply** → **Problem** → its status check must be `createArray('Submitted', 'Returned to HR')`. If it isn't, replace it with:
```
if(empty(outputs('Request')), concat('I couldn''t find request #', outputs('RequestIdText'), ', so nothing was sent.'), if(not(contains(createArray('Submitted', 'Returned to HR'), coalesce(outputs('Request')?['RequestStatus']?['Value'], ''))), concat('Request #', outputs('RequestIdText'), ' is ', coalesce(outputs('Request')?['RequestStatus']?['Value'], 'not open for a decision'), ', so it can''t be declined from this email. Nothing was sent.'), if(less(length(outputs('Comments')), 15), 'Your comments were empty or too short, so nothing was sent to the manager. Please click Decline in the HR review email again and write, above the line, why the request is declined and what needs to change.', '')))
```

### 8.2 A second HTML variable
Variables must be created at the top level of a flow.
1. Find **Initialize variable (Html)** near the top of **Process HR Decline Reply**.
2. Click the **+** directly below it → **Initialize variable** → **Name:** `NoticeHtml` · **Type:** String · **Value:** empty.

### 8.3 The notice (only when the request came back from HR Ops)
1. At the **end** of the flow, below **Confirm To HR** (and Move email, if you have it): **+** → **Condition** → rename `Was Returned By HR Ops` → left: **Expression** `equals(outputs('Request')?['RequestStatus']?['Value'], 'Returned to HR')` → **is equal to** → right: **Expression** `true`.
2. In **True**, add in order:
   1. **Get file content using path** → rename `Get Notice Template` → **JD Templates** → **JD Declined By HRBP Notice.html**
   2. **Set variable** → **NoticeHtml** →
   ```
   if(contains(string(body('Get_Notice_Template')), '$content'), base64ToString(body('Get_Notice_Template')?['$content']), string(body('Get_Notice_Template')))
   ```
   3. **Compose** → rename `Notice Tokens`:
   ```json
   [
     {"k": "Preheader",         "v": "@{concat('HRBPs declined request #', outputs('RequestIdText'), ' (', outputs('Request')?['Title'], ') after your return. No action needed.')}"},
     {"k": "JobTitle",          "v": "@{outputs('Request')?['Title']}"},
     {"k": "RequestID",         "v": "@{outputs('RequestIdText')}"},
     {"k": "DeclinedDate",      "v": "@{outputs('DeclinedDate')}"},
     {"k": "HRComments",        "v": "@{outputs('Comments')}"},
     {"k": "DeclinedByName",    "v": "@{outputs('DeclinedByName')}"},
     {"k": "OpsReturnComments", "v": "@{coalesce(outputs('Request')?['OpsReturnComments'], 'No reasons were given.')}"},
     {"k": "OpsReturnedBy",     "v": "@{coalesce(outputs('Request')?['OpsReturnedBy'], '')}"},
     {"k": "RequesterName",     "v": "@{outputs('RequesterName')}"},
     {"k": "SectionsChanged",   "v": "@{coalesce(outputs('Request')?['SectionsChanged'], '')}"}
   ]
   ```
   4. **Apply to each** → rename `Apply Notice Tokens` → **From:** `outputs('Notice_Tokens')` → **Settings** → **Concurrency Control: Off**. Inside it:
      - **Compose** → `Notice Step`:
      ```
      replace(variables('NoticeHtml'), concat('{{', item()?['k'], '}}'), replace(replace(replace(replace(replace(string(item()?['v']), '&', '&amp;'), '<', '&lt;'), '>', '&gt;'), decodeUriComponent('%0D'), ''), decodeUriComponent('%0A'), '<br>'))
      ```
      - **Set variable** → **NoticeHtml** → `outputs('Notice_Step')`
   5. **Send an email from a shared mailbox (V2)** → rename `Send Declined Notice` → **Original Mailbox Address:** HRConnect · **To:** **JD HR Ops Mailbox** (env var) · **Subject:** `concat('Declined by HRBP: JD Request #', outputs('RequestIdText'), ' · ', outputs('Request')?['Title'])` · **Body:** **</>** → only **NoticeHtml**
3. Leave **False** empty.
4. **Save**.

`DeclinedDate`, `DeclinedByName` and `RequesterName` are the Composes you already built for the decline email. If yours have different names, use those.

---

## Part 9: End-to-end tests

Use **fresh** requests submitted after Parts 4 and 5.

### 9.1 The main loop (one request, #A)
| # | Do | Expect |
|---|---|---|
| 1 | Submit #A | The HRBP email arrives. **JD Email Archive** has **Request A - HRBP Review.html** |
| 2 | HRBP email → **Approve** | The HR Ops email arrives. **Request A - HR Ops Review.html** is archived. Status **With HR Ops** |
| 3 | HR Ops email → **Return To HR** (2 sentences) | The **Returned To HRBP** email reaches the HRBPs, with comments, the details and the **Original HRBP Review .eml**. Status **Returned to HR**, with the Ops columns filled. HR Ops gets a confirmation |
| 4 | Open the .eml | It opens as the full original HRBP email |
| 5 | Returned email → **Resend To HR Ops** (with a note) | The **Resent To HR Ops** email reaches HR Ops: the HRBP note, "Your Return" with step 3's comments, the section chips, the **Original HR Ops Review .eml** and the 3 buttons. Status **With HR Ops**. HRBP gets "…was resent to HR Ops" |
| 6 | Resent email → **Return To HR** (no comments) | The Returned email shows "No reasons were given." Status **Returned to HR** |
| 7 | Returned email → **Decline To Manager** (with reasons) | The manager gets the decline email. HR Ops gets the **Declined By HRBP** notice. Status **Declined** |

### 9.2 Guards
| # | Do | Expect |
|---|---|---|
| 8 | Old HR Ops email for #A → **Return To HR** | Bounce: "…is Declined, so it can't be returned…" |
| 9 | Old Returned email for #A → **Resend To HR Ops** | Bounce: "…can't be approved…" |
| 10 | A **Submitted** request, with a `[OPS-DECLINE]` subject | Bounce: "…is Submitted…" |
| 11 | Subject changed to `#99999` | Bounce: "I couldn't find request #99999…" |
| 12 | While a request is **With HR Ops**, its HRBP email → **Decline** | Bounce: "…can't be declined from this email" |
| 13 | A first-time HRBP **Decline** (never with HR Ops) | The manager gets the decline email, and **no** notice goes to HR Ops |

### 9.3 Edge cases
| # | Do | Expect |
|---|---|---|
| 14 | A request from before Parts 4–5 → return / resend | The emails still send, just without the .eml attachments |
| 15 | Comments with bullets and a blank line | Kept on separate lines, with the paragraph gap |

**In every run:** **Leftover Tokens** is false, bounces end at **Terminate (Succeeded)**, and child runs end with **Respond** green.

---

## Appendix: Troubleshooting
| Symptom | Fix |
|---|---|
| Trigger error, or the flow never runs | The trigger's Outlook connection must be the one with access to HRConnect. Check the solution's **Connection references** |
| "Invalid reference" when saving | A step name doesn't match. Rename it exactly, or re-insert the reference from Dynamic content |
| The resend sends the **normal** HR Ops email | **IsResend** is false. The request's status wasn't "Returned to HR" when the child ran. Check the spelling of the choice |
| The resend attachment is missing | **JD Email Archive** has no **Request NN - HR Ops Review.html** (an older request, or Part 5.7 not in place) |
| The .eml opens as code | Check **Eml / OpsEml** has a blank line between the headers and `<!DOCTYPE`. Send me its first 10 lines |
| "No access" on Open In Word | Give the HRBP and HR Ops groups **Read** on the documents library |
| The notice goes to HR Ops on a first-time decline | **Was Returned By HR Ops** must check `outputs('Request')` (the status **before** the update), not a fresh lookup |
| `{{…}}` left in an email | Its Token List row is missing or misspelled. **Leftover Tokens** shows true |
