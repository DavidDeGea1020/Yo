# HR Connect (Lily): Terms of Use Acknowledgement – Process Report

Nov 2, 2026 · @Eric

## Summary

HR Connect – Lily now requires every user to accept a Terms of Use (ToU) before she answers anything, and re-prompts each user once every 365 days or whenever the ToU version changes. The build is complete in DEV and runs in both Microsoft Teams and Microsoft 365 Copilot.

The solution has five parts, all packaged in Lily's existing Power Platform solution:

| Component | Type | Purpose |
| --- | --- | --- |
| `LilyTouAcknowledgements` | SharePoint list | Audit log of every Accept and Decline |
| Lily ToU Site / List / Current Version | Environment variables (3) | Point the flows at the right list per environment; control the ToU version |
| Lily ToU - Check Acknowledgement | Power Automate cloud flow | Looks up a valid acceptance for the user |
| Lily ToU - Record Acknowledgement | Power Automate cloud flow | Writes the user's Accept or Decline |
| ToU Gate | Copilot Studio topic (trigger: A message is received) | Runs the check, shows the ToU card, records the response |

No custom code is involved. The flows use the standard SharePoint connector through a solution connection reference.

## Architecture overview

&#91;embedded content: ToU gate flow · 2 flows, 1 list, 1 decision\]

The gate calls Flow 1 on a conversation's first message; users with a valid acceptance go straight to Lily's answer, everyone else gets the card, and Flow 2 logs their choice.

## Data store: SharePoint list

Every response is a new row in `LilyTouAcknowledgements`; rows are never updated, so HR and Legal keep a full audit trail.

| Column (internal name) | Type | Contents |
| --- | --- | --- |
| UserUPN (`Title`) | Single line of text | User's UPN, lower-cased |
| `UserObjectId` | Single line of text | Entra ID object ID |
| `DisplayName` | Single line of text | User's display name |
| `TouVersion` | Single line of text | ToU version accepted, e.g. 2026.1 |
| `Decision` | Single line of text | Accepted or Declined |
| `RespondedOn` | Date and time | UTC time of the response |
| `ExpiresOn` | Date and time | RespondedOn + 365 days |
| `Channel` | Single line of text | Teams or M365 Copilot channel ID |
| `ConversationId` | Single line of text | Copilot Studio conversation ID |

**Indexes:** UserUPN (Title) and ExpiresOn, so lookups stay fast past the 5,000-item list view threshold.

**Permissions:** inheritance is broken on the list. Broad Members and Visitors groups are removed; access is limited to HR admins who need audit access and the account behind the flows' SharePoint connection. End users never touch the list directly.

## Environment variables and solution components

Three environment variables keep the flows portable across DEV, UAT and QA, and let HR force everyone to re-accept by changing one value.

| Display name | Data type | DEV value | Used by |
| --- | --- | --- | --- |
| Lily ToU Site | Data source (SharePoint site) | HR Connect SharePoint site | Both flows |
| Lily ToU List | Data source (SharePoint list, parent: Lily ToU Site) | LilyTouAcknowledgements | Both flows |
| Lily ToU Current Version | Text | 2026.1 | Both flows |

All of the following live in Lily's solution and travel together on export:

- Topic: ToU Gate
- Cloud flows: Lily ToU - Check Acknowledgement, Lily ToU - Record Acknowledgement
- Environment variables: the three above
- Connection reference: SharePoint

The SharePoint list itself is not a solution component. It must exist in, or be reachable from, each target environment.

## Power Automate flows

Both flows use the **When Copilot Studio calls a flow** trigger and return values with **Respond to Copilot**, so they run only when Lily calls them.

### Flow 1: Lily ToU - Check Acknowledgement

Finds the user's most recent valid acceptance for the current ToU version.

| Step | Action | Configuration |
| --- | --- | --- |
| Input | Trigger | `UserUPN` (text) |
| 1 | Compose (UPNClean) | `replace(toLower(triggerBody()?['text']), '''', '''''')` lower-cases the UPN and escapes apostrophes |
| 2 | SharePoint Get items | Site and list from environment variables; Top Count 1; Order By `ExpiresOn desc` |
| 3 | Respond to Copilot | Returns `HasValidAck` (Yes/No), `ExpiresOn` (text), `TouVersion` (text) |

Get items filter query (tokens in brackets are dynamic content):

```
Title eq '[UPNClean]' and TouVersion eq '[Lily ToU Current Version]' and Decision eq 'Accepted' and ExpiresOn gt '[utcNow formatted yyyy-MM-ddTHH:mm:ssZ]'
```

`HasValidAck` is true when Get items returns at least one row; `ExpiresOn` is that row's expiry date, or blank.

### Flow 2: Lily ToU - Record Acknowledgement

Writes one row per response.

| Step | Action | Configuration |
| --- | --- | --- |
| Input | Trigger | `UserUPN`, `UserObjectId`, `DisplayName`, `Decision`, `Channel`, `ConversationId` (all text) |
| 1 | Compose (ExpiresOnCalc) | `addDays(utcNow(), 365)` for Accepted |
| 2 | SharePoint Create item | Maps every input to its column; UPN lower-cased; `TouVersion` from the environment variable; `RespondedOn` = `utcNow()` |
| 3 | Respond to Copilot | Returns `ExpiresOn` (text) |

Declined rows are logged for audit but never count as acceptance, because Flow 1 filters on `Decision eq 'Accepted'`.

## ToU Gate topic (Copilot Studio)

The gate is a custom topic in Lily with the trigger **A message is received**, so it can intercept any message before the orchestrator acts on it. Lily uses generative orchestration and **Authenticate with Microsoft**, which supplies the user's UPN, object ID and display name.

**Trigger condition:**

```
!IsBlank(System.Activity.Text) && (IsBlank(Global.TouExpiresOn) || Global.TouExpiresOn < Now())
```

- The gate ignores Adaptive Card clicks, which carry no text, so a card submit cannot re-fire it.
- It fires only when no acceptance is cached for this conversation, or the cached one has expired. `Global.TouExpiresOn` stores a date rather than a true/false flag, so a long-running Teams conversation cannot outlive the 365 days.

**Topic logic:**

1. Call Flow 1 with `System.User.PrincipalName`.
2. **If `HasValidAck` is true:** set `Global.TouExpiresOn`, then answer the user's message with a Generative answers node over Lily's knowledge sources, and end the topic.
3. **Otherwise:** show the ToU Adaptive Card. Interruptions are off and the card reprompts up to 2 times, so the user cannot reach Lily without choosing.
   1. **Accept with the box checked:** call Flow 2 with Decision = Accepted, set `Global.TouExpiresOn`, confirm the next prompt date, answer the original question with Generative answers, end the topic.
   2. **Accept without the box checked:** ask the user to check the box and show the card again.
   3. **Decline:** call Flow 2 with Decision = Declined, show an alternate HR contact, and end the conversation.

**ToU Adaptive Card (schema 1.5):** title, short intro, a Legal-approved summary, a link to the full ToU document, an "I have read and agree" toggle (`agree`), and Accept / Decline submit buttons (`accept`, `decline`). The toggle is not marked required, because that would also block Decline; the topic validates it instead.

## Runtime behavior

A returning user who accepted within the last year triggers one silent flow call per conversation and never sees the card.

| Scenario | Gate fires? | What the user sees |
| --- | --- | --- |
| First message of a conversation | Yes; calls Flow 1 | Valid acceptance: Lily answers. None: the ToU card |
| Later messages in the same conversation | No; acceptance is cached | Nothing |
| Clicking Accept or Decline on the card | No; card clicks carry no text | Confirmation or decline message |
| New conversation (start over, session timeout, new M365 Copilot chat) | Yes, once; globals reset | Nothing, if acceptance is still valid |
| Acceptance expires mid-conversation | Yes; cached date is past | The card |
| ToU version changed in the environment variable | Yes, at next conversation | The card |
| After declining | Yes, on next message | The card again |

## Testing performed

All tests below passed in DEV, in both Teams and M365 Copilot, using a cleared list between runs.

| # | Test | Expected and observed result |
| --- | --- | --- |
| 1 | First message ever | Card appears |
| 2 | Accept without checking the box | Prompt to check the box; card shown again |
| 3 | Check the box, then Accept | Confirmation; row with Decision = Accepted, ExpiresOn about one year out; original question answered |
| 4 | Send another message | No card; Flow 1 run history shows no new run |
| 5 | New conversation | No card; Flow 1 runs once and returns true |
| 6 | Row's ExpiresOn edited to yesterday, then new conversation | Card appears again |
| 7 | Lily ToU Current Version set to 2026.2, then new conversation | Card appears again; value reset afterward |
| 8 | Decline | Row logged as Declined; message shown; conversation ends; next message shows the card |
| 9 | Type a message while the card is waiting | Reprompt; no access to Lily without a choice |

Both flows were also unit-tested manually in Power Automate before being connected to the topic.

## Deployment: DEV to UAT to QA

The feature ships with Lily's existing solution through the standard managed-solution pipeline.

1. Confirm all components are in Lily's solution: the topic, both flows, the three environment variables and the SharePoint connection reference.
2. Export from DEV as **Managed**.
3. Import into UAT, then QA. The import wizard asks for:
   - **Connection reference:** the SharePoint connection for that environment
   - **Lily ToU Site / List:** that environment's site and list (a matching list per environment, or one shared list)
   - **Lily ToU Current Version:** 2026.1
4. After import, turn on both flows if they are off, then open Lily in that environment and **Publish**.

## Operations and known limitations

**Routine maintenance**

- **New ToU from Legal:** update the card text and document link, then change Lily ToU Current Version (for example to 2026.2) in each environment. Every user is prompted again at their next conversation; no other change is needed.
- **Audit requests:** HR admins can filter or export `LilyTouAcknowledgements` directly from SharePoint.

**Known limitations**

- **First-message routing.** Because the gate owns the first turn of each conversation, it answers that message from Lily's knowledge sources. A first message meant for a topic, such as "I need to open an HR ticket", does not reach Open HR Ticket until the user's next message.
- **Flow dependency.** The gate depends on Flow 1 and the SharePoint connection. If either fails, users cannot get past the gate.
- **Clickwrap acceptance.** Acceptance is a checkbox plus Accept button, not a typed e-signature. Legal should confirm this meets their requirement.

**Asks for IT**

- [ ] Confirm the account that owns the SharePoint connection in each environment, ideally a service account rather than an individual.
- [ ] Review list permissions and apply the appropriate retention policy to `LilyTouAcknowledgements`.
- [ ] Set up failure alerts on both flows so a broken connection is caught before users report it.
