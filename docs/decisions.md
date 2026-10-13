# Fixer Decision Log

Answers to the open questions in the design doc (section 14) and the assumptions it listed. Each row says what was decided, who decided it, and what changed as a result. Later rows override earlier ones.

| Date | Ref | Question | Decision | Decided by | Consequence |
|---|---|---|---|---|---|
| 2026-10-13 | Q1 | Fixer's named owner | **Dave Christy** | Dave Christy | Receives P1 alerts and the Monday proof pack; signs the charter addition; sole editor of the audit list |
| 2026-10-13 | Q13 | Hybrid Runbook Worker host | **An existing on-prem domain-joined server, co-managed with Netrio.** Server name to be chosen. | Dave Christy | P0-1 becomes a Netrio change request (Arc agent, worker extension, outbound HTTPS). Run two workers so a 30-day silence never purges the group |
| 2026-10-13 | Q8 | Authentication method | **Password hash sync** | Dave Christy | Containment order confirmed: Entra block first, AD disable second, re-apply the Entra block every 5 minutes until AD disable succeeds. `accountExpires` has no cloud effect |
| 2026-10-13 | Q9 | OU layout | **Separate OUs already exist** for employees, contractors, and disabled users. Paths to be recorded in the Site Map and runbook config. | Dave Christy | P0-3 shrinks to delegating rights on the existing OUs. Worker type still drives OU placement and `employeeType` (see Q15) but nothing else |
| 2026-10-13 | Q10 | License assignment | **Direct per-user assignment** | Dave Christy | Runbooks use `assignLicense` and remove by the same call; no license groups in the Group Registry |
| 2026-10-13 | Q11 | Exchange hybrid | **An on-prem Exchange server existed once and is decommissioned** | Dave Christy | P0 check added: read `msExchRecipientTypeDetails` and `RemoteRecipientType` on a synced mailbox. If still AD-mastered, the shared-mailbox conversion and forwarding attributes must be written from AD on the hybrid worker, not from Exchange Online |
| 2026-10-13 | Q14 | ClickUp plan | **Business Plus** | Dave Christy | Workspace invite and removal are human subtasks (Enterprise-only API). 1,000 requests per minute per token. Conditional form logic is available |
| 2026-10-13 | Q4 | App owners | **A different IT person per app, with overlap; names to follow** | Dave Christy | App Owner table seeded with placeholders; tickets route to the IT list until names are entered |
| 2026-10-13 | Q5 | Contractor access to Claude, ChatGPT, Copilot, BlueCherry | **Same as employees** | Dave Christy | No contractor restriction anywhere. `ContractorAllowed` column removed from the App Owner table |
| 2026-10-13 | Q15 | Form changes | **Declined:** Worker Type, Contract End Date, Sponsor Email, Contracting Company, Additional Apps, Urgency, Confidential, Mailbox Retention. **Approved:** the six Fixer status fields; Dave owns all three forms and will add them when ready. Employee and contractor forms feed the same list and are not treated differently; only the question wording differs. | Dave Christy | See "Structural consequences" below |
| 2026-10-13 | Q16 | Meaning of "Date & Time of Termination (EST)" | **It is the access cutoff** | Dave Christy | Containment is scheduled at exactly that timestamp. Relabeling the field to say "cutoff" is recommended but optional (field ID unchanged) |
| 2026-10-13 | A8 | First credential delivery | **Temporary Access Pass sent to the supervisor** on the start morning | Dave Christy | `Fixer-Entra-IssueTap` is in scope; proposed lifetime 8 hours, single TAP per user; supervisor email comes from the form |
| 2026-10-13 | Q7 | Mailbox and account retention | **Convert to a shared mailbox only when the form asks for a forward or a delegate.** Otherwise remove the license (mailbox is deleted by Microsoft after 30 days) and delete the AD account at T plus 30 days under its own approval. | Dave Christy | A converted mailbox keeps its account disabled forever (no automatic deletion); the proof pack lists those accounts yearly |
| 2026-10-13 | A4 | Default license | **Microsoft 365 Business Premium** (SKU part number `SPB`) for every new account. Business Premium is capped at 300 users; Swim USA moves individual users to Microsoft 365 E3 only when they need more mailbox storage, and new users never start on E3. | Dave Christy | License SKU map has one row (SPB). Fixer never assigns E3. If SPB has no seats left, the license step holds and IT decides (buy, or free a seat). On offboarding, Fixer removes whichever license the user holds, SPB or E3. A leaver on E3 whose mailbox exceeds 50 GB cannot become an unlicensed shared mailbox, so when the retention rule says convert, Fixer keeps the license on that mailbox and flags it to IT |

## Structural consequences of Q15 and Q5

1. **Two paths, not four.** Onboarding and offboarding. Worker type (employee or contractor) is still detected from which form fields are populated, and it still decides the OU and the `employeeType` attribute, but it changes no approval, license, app, or timing rule.
2. **Contractor end-date enforcement is removed.** Design Phase 4 is withdrawn. Contractors leave through the same offboarding form as employees. The residual risk, a contractor who is never offboarded, is covered only by a governance item: the Monday proof pack lists accounts with `employeeType=Contractor` and no sign-in in 60 days.
3. **No Urgency field.** Every offboarding is scheduled at the cutoff timestamp. A cutoff at or before submission time is the immediate path: the containment card goes out at once and executes on approval.
4. **No Confidential field.** To avoid revealing a scheduled termination to eight app owners days early, **revoke tickets are created at the cutoff, not at submission**, for every offboarding. The approval card still goes only to the named approvers.
5. **No Mailbox Retention field.** The Q7 rule above decides: forward or delegate requested means convert to shared; otherwise license removal and account deletion at T plus 30.
6. **No Additional Apps field.** Grant tickets for Databricks, Claude, ChatGPT, and Copilot are created only when the approver accepts a suggestion extracted from "Any Additional Information". Revoke tickets for all eight apps still fire on every offboarding.
7. **Credential delivery contact** is not needed; the supervisor receives the TAP.

## Still open (not blocking Phase 0, blocking Phase 2)

| Ref | Question | Needed for |
|---|---|---|
| Q2 | HR approver (role or person) and backup | Stage 1 approval routing |
| Q3 | IT approver, backup chain, on-call approver group for immediate terminations, who carries the phone | Stage 2 routing and containment paging |
| Q6 | Pre-approve containment at submission for scheduled terminations (recommended) | Whether containment can fire unattended at the cutoff |
| Q4 names | The IT person per app | App Owner table |
| Q13 name | The server that hosts the hybrid worker | P0-1 |
| Q9 paths | The three OU distinguished names | Runbook config |
