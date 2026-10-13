# Fixer Phase 0 Build Specification
**Surf Crew / Fixer** | Draft v0.1 | October 13, 2026 | Owner: Dave Christy
**Parent:** [Onboarding and Offboarding Workflow Design](fixer-onboarding-offboarding-workflow-design.md) (v0.1, merged)
**Status:** build-ready for every item not marked **[BLOCKED ON Q#]**. The Q numbers refer to section 14 of the design doc.

This document turns the design into the concrete artifacts Phase 0 needs: exact ClickUp fields and IDs, the new fields and forms, the state and audit list schemas, the reference tables, the change set contract, and the catalog of runbooks and flows with their permissions. It is written so a builder can start without re-reading the design doc, and so that nothing is built that the design did not call for.

**Confidence key.** **[OBS]** read directly from the ClickUp workspace on October 13, 2026 (field IDs are in `config/clickup-fields.json`). **[DESIGN]** carried from the design doc unchanged. **[NEW]** a decision made in this spec that the design doc left open; each one says why.

---

## 0. Corrections to the design doc from a direct field read [OBS]

| Design doc said | What the workspace shows | Effect on build |
|---|---|---|
| Requester identity for offboarding is not captured | Offboarding list has a **Requestor Email** field (email type) | Separation-of-duties check on offboarding uses this field; no form change needed for it |
| Employee Company Email is the primary key | It is a **short_text** field, not an email-typed field | Fixer must validate the value (RFC-style format, `@swimusa.com` or an allowed-domain list) before any directory lookup; a value that fails validation goes to Needs Info |
| One Location field | Two overlapping dropdowns: **Location** (9 options) and **Office Locations** (10 options, different names for the same sites) | Fixer reads both; the Site Map table (section 5.4) maps each option ID to one site code. Form owners should retire one of the two (Q15) |
| Division(s) Access is the division signal | There is also a free-text **Divisions/Brands Needed** | Labels field is authoritative; free text is LLM-normalized to suggestions only |
| YPLM Access Needed is a checkbox | Both a checkbox and a legacy short-text twin exist, plus a **PLM Request Types** label and an **Accumark Access Needed?** dropdown | Checkbox is authoritative for YPLM; AccuMark stays a human subtask |
| ClickBot automations create app subtasks | The ClickUp API exposes no way to list native Automations | The audit of existing automations is a manual step in the ClickUp UI (section 8, item P0-11) |

None of the 44 onboarding fields or 20 offboarding fields is marked required at the list level [OBS]. Required-ness lives in the form definitions, which the API does not expose.

---

## 1. Fixer custom fields to add to both lists [DESIGN]

Create these on both lists. Record the generated field IDs in `config/clickup-fields.json` under `fields_to_add`.

| Field | Type | Options or format | Written by |
|---|---|---|---|
| Fixer Request ID | short_text | `FX-YYYY-NNNN` | Fixer only |
| Fixer State | drop_down | Received, Needs Info, Awaiting HR, Awaiting IT, Approved: Scheduled, Executing, Partially Complete, Complete, Rejected, Cancelled, Failed: Needs Human | Fixer only |
| Worker Type (resolved) | drop_down | Employee, Contractor, Unknown | Fixer only |
| Approval Status | short_text | Free text, for example `HR approved 10/14 3:10 PM ET; IT pending` | Fixer only |
| Scheduled Execution | date | Date and time, Eastern shown in UI, stored UTC | Fixer only |
| Fixer Summary | text | Rollup of executed steps and app ticket status | Fixer only |

Field permissions: hide these fields from the forms so submitters cannot set them. Fixer treats any non-empty Fixer Request ID on a newly created task as a duplicate signal and investigates (section 8.1 of the design doc).

---

## 2. Form field changes [DESIGN, with concrete definitions]

### 2.1 Onboarding list (serves both the New Hire form and the New Contractor form)

| Field | Type | Options | Required on | Why |
|---|---|---|---|---|
| Worker Type | drop_down | Employee, Contractor | Both forms, hidden and pre-set per form if ClickUp forms allow a default; otherwise the deterministic rule in design 3.3 stays as fallback | Removes the inference |
| Contract End Date | date | | Contractor form | Enforcement needs a date |
| Sponsor Email | email | | Contractor form | Approval routing; the Swim USA employee accountable for the contractor |
| Contracting Company | short_text | | Contractor form | Record only |
| Additional Apps | labels | Databricks, Claude, ChatGPT, Microsoft 365 Copilot, BlueCherry | Neither (optional) | The in-scope apps that have no field today. BlueCherry also stays selectable in the legacy apps label; Fixer treats either as a request |
| Credential Delivery Contact | short_text | Personal email or mobile number | Both forms **[BLOCKED ON Q: credential policy, design assumption 8]** | Only add if the policy is to deliver to the person; if delivery goes to the supervisor, skip this field |

**One person per submission.** The New Contractor form should collect one contractor per submission. Until the form is changed, Fixer keeps the split logic (one request per name in **New Contractor(s) Name(s)**) so that existing submissions still work. **[NEW]** Fixer rejects a submission naming more than 5 people as an anomaly and routes it to a human, because a list that long is more likely a bulk project than a form error.

### 2.2 Offboarding list

| Field | Type | Options | Required | Why |
|---|---|---|---|---|
| Worker Type | drop_down | Employee, Contractor | Yes | Routing |
| Urgency | drop_down | Scheduled, Immediate | Yes | Fast-track path |
| Confidential | checkbox | | No (default off) | Suppresses tickets and broad notices until cutoff |
| Mailbox Retention | drop_down | Convert to shared mailbox, No retention needed | Yes | Decides the order of license removal and mailbox conversion |
| Rename: "Date & Time of Termination (EST)" to "Access cutoff date and time (Eastern)" | date (existing field `dbb44014-...`) | | Yes | Removes the last-day vs cutoff ambiguity. Renaming keeps the field ID, so no code change |

### 2.3 Form-level settings to verify in the ClickUp UI (not visible by API)

- Which questions are required on each of the three forms.
- Whether a form can set a hidden default value for Worker Type.
- Whether the forms create exactly one task per submission.

---

## 3. ClickUp webhook registration [DESIGN]

| Setting | Value |
|---|---|
| Endpoint | Power Automate "When an HTTP request is received" trigger URL of the Fixer Intake flow |
| Scope | Two webhooks, one per list: `list_id` 901100618258 and 901100622311 |
| Events | `taskCreated`, `taskUpdated`, `taskStatusUpdated`, `taskCommentPosted` (comment events feed the Needs Info loop; status events on subtasks feed the app ticket rollup) |
| Secret handling | The secret returned at creation is stored in the Automation account's credential store or Azure Key Vault, never in the flow definition. The Intake flow reads it through a secured input |
| Signature check | HMAC-SHA256 of the raw body, hex, compared to `X-Signature` with a constant-time compare. Compute over the raw request body bytes, never over a re-serialized JSON object. Any mismatch: respond 401, log an `intake.signature_rejected` audit event, do nothing else |
| Reply | ClickUp counts a delivery as failed if the endpoint returns a non-success status **or takes longer than 7 seconds** [V]. The Intake flow therefore does only three things synchronously: verify the signature, write a `webhook_id:history_item_id` receipt row, respond 200. Everything else runs after the response, so the trigger must use the "Respond" action before the downstream work, not at the end |
| Idempotency | ClickUp sends **both** `taskCreated` and `taskStatusUpdated` for every new task [V], and retries each failed event up to five times [V]. Dedupe on `{webhook_id}:{history_item_id}` (the key ClickUp documents) before any processing |
| Health | A webhook's `health.fail_count` rises on each dropped event; at **100** the webhook is **suspended and ClickUp stops sending without notifying anyone** [V]; a 410 response suspends it immediately. The reconciliation sweep (F2) therefore also calls Get Webhooks on each run, alerts the owner when `health.status` is not `active`, and re-activates a suspended webhook with Update Webhook. Never return 410 from the endpoint |
| Rate limits | The API returns 429 with `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` (Unix timestamp); there is **no** `Retry-After` header [V]. Retry logic waits until `X-RateLimit-Reset`, not on a fixed backoff. Per-token limits are 100 requests per minute on Business and below, 1,000 on Business Plus, 10,000 on Enterprise [V] |

[V] items above were checked against developer.clickup.com and help.clickup.com on October 13, 2026; the page list is in the design doc's verification appendix (v0.2).

**ClickUp plan dependencies surfaced by verification [V]:**

| Capability | Plan needed | Fixer impact |
|---|---|---|
| API invite, edit, and remove users; invite and remove guests | Enterprise only | On any lower plan, design steps 19 (onboard) and 14 (offboard) become urgent human subtasks. **[BLOCKED ON Q14]** |
| Native Automation "Call webhook" | Business and above; requests are unsigned, custom headers only | Confirms the API webhook as the primary trigger; native automations are not a secure fallback for a privileged workflow |
| Form conditional logic | Business Plus and Enterprise | Needed if the forms are to show contractor-only questions conditionally; otherwise keep two separate forms as today |
| Required form questions | All plans; required Custom Fields at list level are Business and above | Required-ness can be enforced on the form questions themselves |
| Automations API | Does not exist in the public API | Audit of existing ClickBot automations is manual (P0-11) |

**Reading task data [V]:** Get Task with `include_subtasks=true` returns `custom_fields[]` with only set values populated; dropdown values are option IDs (names come from `type_config.options`), dates are Unix milliseconds (date-only fields return 4:00 AM in the token owner's time zone), and unset checkboxes may come back as null rather than false. The field map in `config/clickup-fields.json` keys on option IDs for this reason.

---

## 4. State and audit lists (SharePoint) [DESIGN]

Site: the same SharePoint site that holds the email triage audit list. Two lists, both with item-level permissions set so that only the Fixer service connection can add items; everyone else reads; the Fixer owner can edit with version history on.

### 4.1 Fixer Requests (one row per request)

| Column | Type | Notes |
|---|---|---|
| Title | Text | The Fixer Request ID (`FX-2026-0142`) |
| RequestKey | Text, indexed, enforce unique | `ONB\|...` or `OFF\|...` per design 8.1 |
| Path | Choice | EmployeeOnboard, ContractorOnboard, EmployeeOffboard, ContractorOffboard |
| State | Choice | Same values as the Fixer State ClickUp field |
| ClickUpTaskId | Text, indexed | Originating task |
| ClickUpListId | Text | 901100618258 or 901100622311 |
| LinkedTaskIds | Multi-line text (JSON array) | Duplicate or resubmitted tasks linked to this request |
| SubjectDisplayName | Text | |
| SubjectUPN | Text | Empty until generated (onboarding) or resolved (offboarding) |
| SubjectAdObjectGuid | Text | |
| SubjectEntraObjectId | Text, indexed | |
| WorkerType | Choice | Employee, Contractor, Unknown |
| WorkerTypeSource | Choice | Form, History, AdEmployeeType, AdOu, Default |
| RequesterEmail | Text | |
| ApproverStage1 | Text | UPN of the HR or sponsor approver |
| ApproverStage2 | Text | UPN of the IT approver |
| ChangeSetJson | Multi-line text | Current change set (section 6) |
| ChangeSetHash | Text | SHA-256 of the canonical change set |
| ContainmentChangeSetJson / ContainmentChangeSetHash | Multi-line text / Text | Offboarding only |
| ApprovalsJson | Multi-line text | Array of `{stage, approver, decision, comment, at, hash}` |
| ExecuteAt | Date and time | Containment or account-creation time, UTC |
| EnableAt | Date and time | Onboarding: start-morning enable time, UTC |
| ContractEndDate | Date | Contractor only |
| DeleteAt | Date and time | Offboarding: T plus retention days |
| LockedUntil | Date and time | Lease for concurrency (design 8.1); null when free |
| LockedBy | Text | Flow run ID holding the lease |
| LastError | Multi-line text | |
| AppTicketsJson | Multi-line text | Array of `{app, taskId, status, owner, updatedAt}` |
| Created / Modified | Built in | |

### 4.2 Fixer Audit Log (append only)

| Column | Type | Notes |
|---|---|---|
| Title | Text | `{RequestId} {EventType}` |
| At | Date and time | UTC, millisecond precision kept in the Title suffix if needed |
| RequestId | Text, indexed | |
| ClickUpTaskId | Text | |
| Path | Choice | As above |
| EventType | Choice | See section 9 |
| System | Choice | ClickUp, PowerAutomate, AzureOpenAI, AD, Entra, ExchangeOnline, OneDrive, M365Licensing, Teams, SharePoint |
| StepKey | Text, indexed | `{RequestKey}\|{stepName}` |
| Actor | Text | `fixer` or a human UPN |
| ChangeSetHash | Text | Hash the event was executed under |
| PaRunId | Text | Power Automate run ID |
| AutomationJobId | Text | Azure Automation job ID, when a runbook ran |
| Result | Choice | Started, Succeeded, AlreadyDone, Failed, Skipped, Alerted |
| Detail | Multi-line text | Structured JSON; never contains passwords, TAP values, or tokens |

Retention: never delete; archive yearly by the owner if the list approaches SharePoint's view threshold.

---

## 5. Reference tables (SharePoint lists, owner-maintained)

### 5.1 Group Registry

| Column | Type | Notes |
|---|---|---|
| GroupName | Text | Display name |
| Source | Choice | AD, Cloud |
| AdDistinguishedName | Text | AD groups only |
| EntraObjectId | Text | Both (synced groups have one too) |
| Kind | Choice | Security, Distribution, M365, Teams, LicenseGroup |
| FixerManaged | Yes/No | Only Yes groups can appear in a change set |
| Sensitive | Yes/No | Sentinel list; never added by Fixer, flagged on offboarding |
| Aliases | Multi-line text | Names people type in the free-text distro fields; used by the normalizer |

Seed: export with `Get-MgGroup -All -Property displayName,id,onPremisesSyncEnabled,groupTypes,mailEnabled,securityEnabled` and mark FixerManaged and Sensitive by hand. **[BLOCKED ON Q10]** for which groups are license groups.

### 5.2 Division-to-Group map

| Column | Type |
|---|---|
| DivisionOptionId | Text (the ClickUp label option ID from `config/clickup-fields.json`) |
| DivisionLabel | Text |
| GroupNames | Multi-line text (one Group Registry name per line) |

Ten rows, one per division label. Owner fills the group names. **[BLOCKED ON group inventory]**

### 5.3 App Owner table

| Column | Type | Notes |
|---|---|---|
| App | Choice | Databricks, BlueCherry, NuOrder, YPLM, Qlik, Claude, ChatGPT, Copilot |
| OwnerClickUpUserId | Number | Ticket assignee |
| BackupClickUpUserId | Number | |
| GrantSlaHours | Number | Default 24 business hours before the start date |
| RevokeSlaHours | Number | Default 4 business hours |
| ContractorAllowed | Choice | Yes, No, SponsorJustification |

**[BLOCKED ON Q4 and Q5]** for owners and contractor policy.

### 5.4 Site Map

| Column | Type | Notes |
|---|---|---|
| ClickUpOptionId | Text | From either Location field |
| SiteCode | Text | One code per physical site |
| PhysicalDeliveryOfficeName | Text | AD attribute value |
| UsageLocation | Text | ISO 3166 two-letter code |
| AdOu | Text | Only if OU is per site **[BLOCKED ON Q9]** |

### 5.5 License SKU map

| Column | Type |
|---|---|
| WorkerType | Choice (Employee, Contractor) |
| SkuPartNumber | Text (for example `SPE_E3`) |
| SkuId | Text (GUID) |
| AssignmentMethod | Choice (Direct, Group) |
| LicenseGroupName | Text (when Group) |

**[BLOCKED ON Q10 and design assumption 4]**

---

## 6. Change set contract

The change set is the single object an approver approves and a runbook executes. Schema: `schemas/change-set.schema.json`.

Rules:
- **Canonical form:** keys sorted, no whitespace, UTF-8, timestamps in ISO 8601 UTC. The hash is SHA-256 of that string.
- **Approval binds to the hash.** Any rebuild that yields a different hash voids existing approvals (design 7.5).
- **Items are typed.** Each item has `type`, `target`, `source` (AD or Cloud), `reason` (which form field or table produced it), and `requiresApprovalStage` (1 or 2). Approvers see `reason` on the card.
- **Suggestions are not items.** Copy-from diffs and LLM-extracted app hints go in `suggestions[]`, which is not hashed into the executable set until an approver accepts one through Request changes.
- **Secrets never appear.** Passwords and TAP values are generated inside the runbook at execution and are never written to the change set, the audit log, or ClickUp.

---

## 7. Runbook catalog (Azure Automation)

All runbooks are PowerShell 7, accept a `RequestId`, `StepKey`, and `ChangeSetHash`, write an audit row at start and end, check live state before acting, and return a structured result object. Hybrid runbooks run on the Hybrid Runbook Worker as the Fixer AD service account; cloud runbooks run with the Automation account's managed identity.

| Runbook | Worker | Action | Idempotency check | Rights needed |
|---|---|---|---|---|
| `Fixer-AD-ReadUser` | Hybrid | Resolve a user by UPN, mail, or name; return objectGUID, OU, groups, employeeType, enabled, accountExpires | Read only | Read on the domain |
| `Fixer-AD-CreateUser` | Hybrid | Create disabled user in target OU with attributes, manager, employeeType, accountExpires | `Get-ADUser -Filter` on sAMAccountName and UPN; return AlreadyDone if present with matching employeeType | Create user in Employees and Contractors OUs |
| `Fixer-AD-SetGroupMembership` | Hybrid | Add or remove listed groups | Compare current membership first | Modify members on Fixer-managed groups only |
| `Fixer-AD-EnableUser` | Hybrid | Enable account, set `pwdLastSet` per policy | Skip if already enabled | Write userAccountControl in the two OUs |
| `Fixer-AD-DisableUser` | Hybrid | Disable, reset password twice to random values, stamp description with RequestId, move to Disabled OU | Skip each sub-step already done | Disable, reset password, move between the three OUs |
| `Fixer-AD-DeleteUser` | Hybrid | Delete (T plus retention), only under its own approval | Skip if absent | Delete in Disabled OU only; AD Recycle Bin must be enabled |
| `Fixer-Sync-StartDelta` | Hybrid (Connect server) or remote invoke | `Start-ADSyncSyncCycle -PolicyType Delta` | Skip if a cycle is already running | ADSyncOperators membership on the Connect server |
| `Fixer-Entra-WaitForUser` | Cloud | Poll Graph for the UPN up to 90 minutes | Read only | `User.Read.All` |
| `Fixer-Entra-SetUserProps` | Cloud | usageLocation, preferredLanguage | Compare first | `User.ReadUpdate.All` (least privileged for PATCH /users [V]); `User.ReadWrite.All` also works |
| `Fixer-Entra-SetGroupMembership` | Cloud | Add or remove cloud group members | Compare first | `GroupMember.ReadWrite.All` [V]. Removal must call the `/$ref` form; an app holding both this and a user-write permission can delete the user object by mistake if `/$ref` is omitted [V] |
| `Fixer-Entra-AssignLicense` | Cloud | Direct assignment or license-group membership | Check the user's `assignedLicenses` property first (the `licenseDetails` endpoint is delegated-only and unusable app-only [V]); available = `prepaidUnits.enabled` minus `consumedUnits` from `subscribedSkus` | `LicenseAssignment.ReadWrite.All` [V] and `LicenseAssignment.Read.All` for `subscribedSkus` [V] |
| `Fixer-Entra-RemoveLicenses` | Cloud | Remove all licenses | Skip if none | Same |
| `Fixer-Entra-BlockSignIn` | Cloud | `accountEnabled=false` then `revokeSignInSessions` | Skip if already disabled; always revoke | `User.EnableDisableAccount.All` plus `User.Read.All` (least privileged [V]) and `User.RevokeSessions.All` [V]. A target holding an admin role needs the app to hold a privileged directory role too [V] |
| `Fixer-Entra-Inventory` | Cloud | Snapshot groups, licenses, roles, devices, mailbox type for the audit and the Sentinel check | Read only | `User.Read.All`, `GroupMember.Read.All`, `RoleManagement.Read.Directory`, `Device.Read.All` |
| `Fixer-EXO-SetMailboxOffboarding` | Cloud | Auto-reply, forwarding (internal only), full-access delegate, convert to shared if retaining | Compare each setting | Exchange Online app-only with a custom role scoped to mailbox settings. In an Exchange hybrid the conversion must use on-premises `Set-RemoteMailbox -Type Shared` or sync reverts it [V] **[BLOCKED ON Q11]** |
| `Fixer-OneDrive-GrantAccess` | Cloud | Add the named person as site collection admin on the leaver's OneDrive | Skip if present | SharePoint app-only `Sites.FullControl.All` or the SharePoint Online admin role, scoped by policy |
| `Fixer-Entra-IssueTap` | Cloud | Create a Temporary Access Pass for first sign-in and deliver per policy | Skip if a live TAP exists (one per user [V]) | `UserAuthMethod-TAP.ReadWrite.All` (narrower) or `UserAuthenticationMethod.ReadWrite.All` [V]; the user must be in scope of the TAP authentication method policy [V] **[BLOCKED ON design assumption 8]** |
| `Fixer-Canary` | Both | Weekly synthetic onboard and offboard of a test account in a test OU | n/a | Same as above, test OU only |

**Do not grant** `Directory.ReadWrite.All`; every action above has a narrower permission [V].

### 7.1 Execution constraints confirmed by verification [V]

These are documented Microsoft behaviors that the runbooks and the orchestrator must respect. Sources are listed in the design doc's verification appendix.

1. **AD disable is the authoritative containment step.** A cloud-side `accountEnabled=false` on a synced user is undone by the next sync cycle if the AD account is still enabled. Containment therefore runs the Entra block first for speed, then the AD disable, and if the AD disable fails the Scheduler re-applies the Entra block every 5 minutes (and re-revokes sessions) until the AD disable succeeds, while the P1 alert stands. The design doc's statement that a failed AD disable leaves the cloud "already closed" is withdrawn.
2. **`accountExpires` is not honored under password hash sync.** An expired AD account stays active in Entra ID. The contractor end date is enforced by Fixer's scheduled offboarding, never by `accountExpires`. Under pass-through authentication or federation the docs are silent, so the same rule applies.
3. **Convert to shared before removing the license, and keep the account.** The convert option only exists while the mailbox is licensed; after license removal the data is held 30 days and then deleted (an optional tenant feature, Delicensing Resiliency, adds a further 30 days). A shared mailbox stays anchored to the user object, so an account whose mailbox is retained as shared is **never deleted** at T plus 30; it stays disabled in the Disabled OU and is reviewed at the owner's retention cadence. Shared mailboxes under 50 GB with no archive and no hold need no license.
4. **Approvals cannot outlive a 30-day flow run.** Any approval that could stay open longer than 30 days (a contractor extension requested early, an onboarding submitted far ahead) uses "Create an approval" plus a separate flow triggered by the approval response in Dataverse, not "Start and wait".
5. **Sync timing.** Default delta cycle every 30 minutes; `Start-ADSyncSyncCycle -PolicyType Delta` forces one. Revoked sessions take a few minutes to apply; access tokens already issued last up to one hour, which is the worst-case window after containment for applications that do not support continuous access evaluation.
6. **usageLocation is cloud-set on synced users** and must be set before license assignment. Confirm no sync rule on the Connect server maps usageLocation, because a mapped attribute is overwritten by the next cycle.
7. **employeeType syncs from AD** only on Connect 2.2.1.0 or later with the default "In from AD - User Common" rule intact; a cloned or customized rule may not include it. Check the rule before relying on it for worker type resolution.
8. **Hybrid Runbook Worker group is purged after 30 days without a ping.** Keep two workers in the group and alert on the worker ping metric; a cold standby left off needs re-registration.
9. **No gateway connector writes to AD.** Confirms the Hybrid Runbook Worker choice; the only first-party alternative is Power Automate for desktop on a domain-joined machine, which the design rejects.

---

## 8. Flow catalog (Power Automate)

| ID | Flow | Trigger | Does | Writes |
|---|---|---|---|---|
| F1 | Fixer Intake | HTTP request (ClickUp webhook) | Verify signature, respond 200, read full task after a 60-second delay, compute request key, create or link the Fixer Requests row, start F3 | Fixer Requests, Audit Log |
| F2 | Reconciliation Sweep | Recurrence, every 15 minutes | List tasks created or updated in the last 24 hours in both lists; any without a Fixer Request ID goes through the same path as F1 | Same |
| F3 | Orchestrator | Called by F1, F2, F4 | Validation, extraction (Azure OpenAI), resolution, change set build, approvals, step execution by starting runbook jobs, status loop | Everything |
| F4 | Scheduler | Recurrence, every 5 minutes | Fire rows with `ExecuteAt <= now` and state Approved: Scheduled; take the lease; write a heartbeat row; raise the late-fire alarm | Fixer Requests, Audit Log |
| F5 | App Ticket Rollup | ClickUp webhook `taskStatusUpdated` on subtasks tagged `fixer-app` | Refresh the parent's Fixer Summary and AppTicketsJson | ClickUp, Fixer Requests |
| F6 | Escalations | Recurrence, every 15 minutes | Approval reminders, T minus 24h and T minus 4h pages, cutoff-passed P1 alerts, Needs Info reminders and stalls | Teams, Audit Log |
| F7 | Weekly Proof Pack | Recurrence, Monday 8:00 AM ET | Compose the nine-part pack from the Audit Log; post to the Fixer Governance task and to the owner in Teams | ClickUp, Teams |
| F8 | Scheduler Heartbeat Check | Recurrence, hourly | Alert if F4's last heartbeat is older than 20 minutes | Teams |
| F9 | Canary | Recurrence, Sunday 11:00 PM ET | Run `Fixer-Canary` and record the result for F7 | Audit Log |

Connection rules: F1 through F9 run under a dedicated Fixer service account's connections. The only Azure connection is "Create job" on the Automation account, with the Automation Job Operator role scoped to the account. No Graph connection and no AD connection exists in Power Automate.

---

## 9. Audit event taxonomy

`intake.received`, `intake.signature_rejected`, `intake.duplicate_linked`, `validate.passed`, `validate.needs_info`, `extract.completed`, `extract.low_confidence`, `resolve.completed`, `resolve.ambiguous`, `changeset.built`, `changeset.voided`, `approval.requested`, `approval.responded`, `approval.reminder`, `approval.escalated`, `schedule.set`, `schedule.fired`, `schedule.late`, `step.started`, `step.succeeded`, `step.already_done`, `step.failed`, `step.skipped`, `circuit.opened`, `circuit.closed`, `ticket.created`, `ticket.released`, `ticket.cancelled`, `ticket.status_changed`, `status.written`, `alert.raised`, `verify.passed`, `verify.failed`, `canary.result`, `proofpack.posted`.

Every `step.*` event carries StepKey, ChangeSetHash, PaRunId, and AutomationJobId when a runbook ran.

---

## 10. Phase 0 checklist

| # | Item | Owner | Depends on |
|---|---|---|---|
| P0-1 | Choose and Arc-enable the domain-joined server for the Hybrid Runbook Worker; confirm with Netrio if co-managed | IT | Q13 |
| P0-2 | Create the Automation account, enable managed identity, grant the Graph application permissions in section 7 with admin consent, create the Exchange Online app-only role | IT | Q8, Q11 |
| P0-3 | Create AD OUs (Employees, Contractors, Disabled, Fixer-managed groups) or map existing ones; create the Fixer AD service account with delegated rights per section 7; enable AD Recycle Bin if off | IT | Q9 |
| P0-4 | Confirm Entra Connect sync scope covers the OUs; grant `Fixer-Sync-StartDelta` rights on the Connect server | IT | Q8, Q9 |
| P0-5 | Create the SharePoint lists in sections 4 and 5 with the stated permissions | Fixer owner | Q1 |
| P0-6 | Seed Group Registry, Site Map, License SKU map | IT | Q10, group inventory |
| P0-7 | Fill Division-to-Group map and App Owner table | Business owners | Q4, Q5 |
| P0-8 | Add the Fixer custom fields (section 1) and the form fields (section 2) in ClickUp; record IDs in `config/clickup-fields.json` | Form owner | Q15, Q16 |
| P0-9 | Register the two webhooks (section 3) against the Intake flow | Fixer owner | F1 deployed |
| P0-10 | Create the Fixer service account for Power Automate and its ClickUp API token (scoped user) | IT | |
| P0-11 | Audit existing ClickBot automations on both lists in the ClickUp UI; list each one and mark it keep or disable | Form owner | |
| P0-12 | Deploy runbooks in section 7 to a test OU and run `Fixer-AD-CreateUser` then `Fixer-AD-DisableUser` on a test account | IT | P0-1 to P0-4 |
| P0-13 | Deploy F1, F2, F3 in shadow mode (no runbook jobs, approval cards in preview) and F4, F6 with the late-fire alarm live | Fixer owner | P0-5, P0-8, P0-9 |
| P0-14 | Add this workflow to Fixer's charter with the owner's sign-off | Fixer owner | Q1 |

**Exit criterion for Phase 0 [DESIGN]:** P0-12 passes, and F1 through F4 have processed one real submission from each form in shadow mode with a change set that an IT tech agrees matches what they would have done.
