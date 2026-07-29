# VARStreet QA Engineering Take-Home Assignment
**Candidate:** Pramodini Girgaonkar
**Role:** Junior QA Engineer (2–3 Years Experience)
**CRMs evaluated:** Zoho CRM (Free/Trial) and HubSpot CRM (Free/Starter)

---

# Part 1 — CRM Workflow Analysis

## Task 1.1 — Lead-to-Deal Lifecycle

**Zoho CRM flow:**
1. **Create Lead** — Leads module → New Lead → enter Company, Last Name, Email, Phone, Lead Source, Lead Status.
2. **Convert Lead** — "Convert" button opens a modal with three checkboxes: create Contact, create Account (pre-filled from Company field), create Deal (optional — can be skipped or deferred).
3. **Deal creation** — On conversion, a Deal record is created and linked to the new Contact/Account. Pipeline stage defaults to the first stage of the selected Pipeline.
4. **Pipeline movement** — Deal is dragged across Kanban stages or edited via the Stage dropdown; each stage change can trigger workflow rules.
5. **Closed Won/Lost** — Moving to a "Closed" stage type prompts for Closing Date and (if configured) a Loss Reason for Closed Lost.

**HubSpot CRM flow:**
1. **Create Lead/Contact** — HubSpot doesn't have a hard "Lead" object by default in Free/Starter; a Contact is created directly, with a Lifecycle Stage property (Subscriber → Lead → MQL → SQL → Opportunity → Customer) used to represent lead status instead of a separate record type.
2. **Create Deal** — Deals are created manually or via a "Create deal" action associated with the Contact/Company. There's no forced "conversion" step — Contact and Deal exist independently and are linked by association.
3. **Pipeline movement** — Deals move across a Kanban board; each pipeline can have custom stages with associated probability %.
4. **Closed Won/Lost** — Dragging to Closed Won/Lost stage; Closed Lost prompts for a "Closed lost reason" dropdown if configured.

**Auto-populated vs. manual (Zoho conversion):**
| Field | Behavior |
|---|---|
| Account Name | Auto-filled from Lead's Company field |
| Contact Name/Email/Phone | Auto-copied from Lead |
| Deal Name | Auto-suggested as "[Company]-Deal" but editable |
| Deal Owner | Defaults to Lead Owner, not always the actual salesperson |
| Custom Lead fields | Only copied over if explicitly mapped in Lead-to-Deal/Contact field mapping — unmapped custom fields are silently dropped |

**Behavioral differences (Zoho vs HubSpot):**
1. **Duplicate detection on conversion:** Zoho actively checks for duplicate Leads/Contacts by matching on Email during conversion and blocks/warns before creating a duplicate Contact. HubSpot does **not** block duplicate Contact creation by default — it will happily create two Contact records with the same email unless deduplication automation is separately configured (a paid feature in higher tiers).
2. **Lead as a distinct object:** Zoho has a hard separation between Lead and Contact/Deal — data physically moves to new records on conversion. HubSpot treats "lead" as a status (Lifecycle Stage) on the same Contact record, so there's no conversion event, no separate ID, and no risk of the Lead-to-Contact field-mapping gaps that Zoho has.
3. **Deal creation optionality:** Zoho nudges/forces a Deal to be created as part of conversion (checkbox is checked by default). HubSpot deal creation is a fully separate, optional action — a Contact can exist indefinitely with zero deals, which affects pipeline reporting completeness.

**Where data could be silently lost or overwritten:**
- **Zoho:** Unmapped custom Lead fields disappear silently on conversion — nothing in the UI flags this loss.
- **Zoho:** If "Create Deal" is unchecked during conversion, that decision is not logged anywhere visible later — a QA engineer inspecting only the Contact record later has no way to tell a Deal was intentionally skipped vs. never considered.
- **HubSpot:** Because Lifecycle Stage is just one property on the Contact, if a workflow or manual edit overwrites Lifecycle Stage (e.g., marketing bulk-imports and resets it to "Subscriber"), a Contact can silently regress from "Opportunity" back to "Lead" with no audit trail visible in the main UI (only in property history, which most users never check).

---

## Task 1.2 — Contact & Account Management Workflow

**Creating a Contact under existing vs. new Account:**
- **Zoho:** Contact record has an "Account Name" lookup field. Selecting an existing Account links it directly; leaving it blank creates an orphaned Contact (no Account). No validation forces an Account link.
- **HubSpot:** Contacts and Companies (Zoho's "Account" equivalent) are associated via a separate association action, not a required field. A Contact can be created with zero Company association, and HubSpot's auto-association (matching email domain to Company domain) can silently attach a Contact to the *wrong* Company if domains overlap (e.g., shared gmail.com domains, or a generic company domain used by multiple client contacts).

**Duplicate email across two different contacts:**
- **Zoho:** By default, Zoho enforces email uniqueness on Contacts at the module level (configurable) — attempting to save a second Contact with an identical email triggers a duplicate warning and blocks save unless the admin has disabled this validation rule.
- **HubSpot:** Email is meant to be the primary unique identifier for Contacts, and HubSpot generally **blocks** creating a second Contact with an identical primary email — but this only applies to the primary email property. If the duplicate email is entered as a secondary/additional email, or via API import, HubSpot can silently create a second record, leading to fragmented history.

**Account deletion/merge — downstream effects:**
- **Zoho:** Deleting an Account moves associated Contacts and Deals to "Recycle Bin" logic in some configs, but by default associated Contacts are **not** deleted — they become orphaned (Account field goes blank) rather than being deleted with the Account. Activities logged against the Account can become inaccessible if not also linked to the Contact/Deal.
- **HubSpot:** Deleting a Company does not delete associated Contacts or Deals — they remain but lose the Company association. Merging two Companies consolidates associated Contacts/Deals under the surviving record, but activity timeline entries tied to custom properties on the deleted Company can be lost if property mapping conflicts exist during merge.

**"Primary Contact" for an Account:**
- **Zoho:** No native "Primary Contact" field on Account out of the box — this must be added as a custom lookup field, meaning many orgs never track it and rely on "first Contact created" by convention.
- **HubSpot:** Has a built-in "Primary Contact" designation on Company records that's explicitly selectable, making it a first-class concept vs. Zoho's workaround approach.

---

## Task 1.3 — Email & Activity Logging Workflow (depth on **Zoho CRM**)

- **Where a logged email appears:** An email sent from within a Deal record's activity panel appears on both the Deal's timeline and the linked Contact's timeline (Zoho auto-propagates it up the Contact→Deal relationship). An email sent from the Contact record only, with no Deal open, appears only on the Contact — it does **not** retroactively appear on any of that Contact's associated Deals.
- **Orphaned activities:** Yes — a Call or Note logged via the general Activities module without selecting a "Related To" record saves successfully as a standalone activity with no parent. It's visible only by searching the Activities module directly; it won't surface on any Contact/Deal timeline, making it effectively invisible to a rep working from a Deal view.
- **BCC-based email capture:** Zoho provides a unique BCC-drop-box email address per user. Emails BCC'd to it are logged, but matching happens by email address against existing Contacts — if the recipient's email isn't in Zoho as a Contact, the email is either dropped or logged to an "Unrecognized" bucket. If two different Contacts across different Accounts share an email domain alias, misattribution is possible during matching.
- **Activity timeline UX gaps:** The timeline shows email subject, timestamp, and a snippet, but does **not** clearly show whether an email actually delivered/bounced or was opened, unless email tracking is separately enabled and the user clicks into each entry. A QA engineer relying on "an email appears in the timeline" as proof of successful communication would miss silent send failures (bounce, blocked, or spam-filtered emails still show as "sent" in the timeline).

---

## Task 1.4 — Automation & Workflow Rules (Both CRMs)

**Setup — "When Deal moves to Negotiation, send email + create follow-up task":**

*Zoho:*
1. Setup → Automation → Workflow Rules → New Rule, Module = Deals.
2. Trigger: "On a record action" → Stage changes to "Negotiation."
3. Condition: Stage = Negotiation.
4. Actions: "Email Notification" (select template) + "Create Task" (assign owner, due date offset).

*HubSpot:*
1. Automation → Workflows → Deal-based workflow.
2. Enrollment trigger: Deal stage = Negotiation.
3. Actions: "Send internal email" or "Send automated email" + "Create task" action.

**Intentionally breaking it:**
| Break attempt | Zoho behavior | HubSpot behavior |
|---|---|---|
| Delete associated Contact | Workflow still fires (Deal-level trigger), but email action referencing Contact's email field fails silently — no error surfaced to the user, task is still created | Workflow re-evaluates enrollment criteria; if a personalization token references the deleted Contact, the email step fails and the workflow shows a "Failed" status **only inside workflow history**, not to the deal owner |
| Remove/rename the Deal stage used as trigger | Rule becomes inactive-by-omission — it simply never fires again, with no warning that the trigger condition is now unreachable | HubSpot flags the workflow with a visible warning icon in the Workflows list ("uses a stage that no longer exists") — better surfaced than Zoho |
| Disable the email template | Task creation still succeeds; the email action fails with the error visible only in Workflow → **Automation Logs**, not visible to end users on the Deal record itself | Email step shows as "Not sent" in workflow history; task step still completes |

**Does it fail silently?**
Both platforms fail *partially* silently — task/action steps that don't depend on the broken piece still succeed, giving a false impression that "the automation worked" if the rep only checks for the task and not the email.

**Where a QA engineer would look:**
- Zoho: Setup → Automation → **Workflow Rules → [Rule] → Recent Actions/Logs** (admin-only visibility).
- HubSpot: Automation → Workflows → **[Workflow] → History tab**, filterable by enrolled record and per-action status (admin/marketing hub access).

**Audit trail gaps:**
- Neither CRM surfaces automation failures to the *record owner* by default — a rep working the Deal has no in-context indicator that "this automation ran and partially failed" unless they're an admin who checks logs proactively. This is a real production risk: reps trust that stage-based notifications went out.

---

## Task 1.5 — Reporting & Dashboard Workflow (depth on **HubSpot**)

- **New deal → report sync delay:** Standard HubSpot reports (deals by stage, revenue) update near-real-time (seconds) for native properties, since reports query live object data rather than a batch-processed warehouse for most standard report types. No manual refresh needed.
- **Changing close date/amount → does the number update immediately?** Yes for single-object reports built on the Deal properties directly. However, reports built with **custom report builder** joins across Deal + Company + Contact can lag by a few minutes due to association-index recalculation, especially right after a bulk edit.
- **Custom property + filtering accuracy:** Creating a custom Deal property (e.g., "Renewal Type") and filtering a report by it generally returns accurate results — **but** if the property was created *after* existing deals were entered, those older deals show as blank/null and get silently excluded from filtered views rather than flagged as "missing data," which can understate pipeline totals.
- **Misleading data scenario:** A sales manager builds a "Revenue Closing This Month" report filtered by Close Date. A rep pushes a deal's close date out (common in real slippage) — the deal instantly disappears from this month's report and appears in next month's, with **no visual flag** that forecast dropped due to a date change vs. an actual loss. The manager sees a falling number and may misread it as pipeline attrition. **Fix:** add a secondary report or alert tracking "close date changes in the last 7 days" so date-push activity is visible alongside the headline revenue number.

---

## Task 1.6 — Spot the Logical Bugs

**Bug 1 — Deal can exist without an Owner (both CRMs)**
- **Steps to reproduce:** Create a Deal via bulk import/API where Owner field is left blank; save.
- **Actual behavior:** Deal saves successfully with no Owner assigned.
- **Expected behavior:** System should either enforce a default Owner (e.g., importing user) or block save with a required-field validation.
- **Business impact:** Ownerless deals vanish from every "My Deals" / rep-filtered pipeline report, understating team pipeline and creating deals no one is accountable for — directly affects forecasting accuracy.

**Bug 2 — Deal can be marked Closed Won with required custom fields empty**
- **Steps to reproduce:** Open a Deal, drag to Closed Won stage without validation rules configured on custom mandatory fields (e.g., "Contract Value," "Renewal Date").
- **Actual behavior:** Stage change succeeds regardless of empty custom fields unless a separate validation rule/required-field-per-stage config is explicitly built.
- **Expected behavior:** Business-critical fields should be enforced as required before a Won state is allowed, since this data feeds finance/forecasting.
- **Business impact:** Revenue recognized in reports without contract data backing it — creates reconciliation issues between CRM and finance systems.

**Bug 3 — Re-opening a Closed Lost deal does not reset/flag the original close date consistently**
- **Steps to reproduce:** Close a Deal as Lost with Close Date = July 1. Reopen it to an earlier pipeline stage. Re-close it as Won weeks later.
- **Actual behavior:** In Zoho, the original Close Date field is not automatically cleared on reopen — it can persist as a stale value unless manually updated, meaning historical/point-in-time reports run against "as of July 1" could show it as Lost, while current reports show it Won, with no reconciling note.
- **Expected behavior:** Reopening a Closed deal should clear/flag the Close Date as pending re-close, and ideally log a "stage history" entry visible on the record.
- **Business impact:** Historical pipeline reports become internally inconsistent depending on when they're run — a serious data-integrity issue for any audit or trend analysis.

**Bug 4 — Automation logs are admin-only, invisible to standard users**
- **Steps to reproduce:** As a standard (non-admin) user, attempt to view Workflow/Automation history for a Deal you own.
- **Actual behavior:** Both Zoho and HubSpot restrict automation execution logs to admin-level roles by default.
- **Expected behavior:** Record owners should at least see a lightweight "automation activity" indicator on their own records (e.g., "notification sent," "notification failed") without needing admin access.
- **Business impact:** Reps can't self-diagnose why an expected email/task didn't happen, increasing support burden on admins and creating trust gaps in automation reliability.

---

## Task 1.7 — Improvement Suggestions (**HubSpot**)

**1. Automation failure visibility on the record itself**
- *Current friction:* When a workflow step fails (e.g., broken personalization token), the failure is buried in admin-only Workflow History — the deal owner has no idea their notification never sent.
- *Proposed change:* Add a small "Automation activity" panel directly on the Deal/Contact record, visible to the owner, showing pass/fail status of the last N automation runs touching that record.
- *Risk/trade-off:* Surfacing failure details to non-admins could expose internal workflow logic (e.g., condition names) that some orgs consider sensitive; would need a simplified, non-technical status view.

**2. Enforce required fields per pipeline stage natively**
- *Current friction:* Deals can be dragged to Closed Won with critical custom fields empty, because per-stage validation isn't a native feature — it requires custom workflow logic to replicate.
- *Proposed change:* Native UI option in pipeline settings to mark specific properties "required to enter this stage," blocking the stage-change with an inline prompt if empty.
- *Risk/trade-off:* Adds friction for legitimate fast-moving deals; needs an admin-configurable override/exception path to avoid blocking valid edge-case closes.

**3. Flag close-date changes distinctly from stage changes in reporting**
- *Current friction:* A pushed-out close date and a genuinely lost deal look identical in month-over-month revenue reports, misleading managers about real pipeline health.
- *Proposed change:* Auto-tag any Close Date edit as a distinct "Date Change" event type in the deal's property history, and offer a report widget summarizing "deals slipped this period" alongside revenue.
- *Risk/trade-off:* Additional property history entries increase data volume and could clutter the activity timeline if not visually separated from substantive updates.

---

# Part 2 — Deep Product Understanding

## Task 2.1 — Mental Model of the Product (**Zoho CRM**, for a new sales rep)

Think of Zoho CRM as four connected buckets that represent where a prospect is in your sales journey.

A **Lead** is a raw, unqualified inquiry — someone who filled a form, got scanned at an event, or was cold-outreached. It's a temporary holding record; nothing about it is "real" yet in terms of pipeline value.

When a Lead is qualified as a genuine prospect, you **convert** it — this splits it into three permanent records: a **Contact** (the person), an **Account** (the company they work for), and optionally a **Deal** (the actual sales opportunity with a dollar value and close date you're tracking). This split matters because a Contact and Account can have *multiple* Deals over time (renewals, upsells), but a Lead is one-shot — once converted, the Lead record is essentially archived and the action moves to these three living records.

Before conversion, the CRM "lives" at the very top of funnel — marketing hand-off, initial qualification calls, nothing forecasted yet. After conversion and through Closed Won, it lives in active pipeline management — stage progression, activities, quote/proposal tracking. Post-sale (after Closed Won), the CRM's role shifts again: the Deal is done, but the Account and Contact persist as the system of record for renewals, support history, and future opportunities — this is why Account is the durable anchor and Lead/Deal are more transactional layers around it.

---

## Task 2.2 — Feature Impact Assessment: "Mandatory manager approval before Closed Won"

| Area | Impact |
|---|---|
| **Deals module** | New "Pending Approval" sub-status needed between Negotiation and Closed Won; stage-change UI must intercept the Closed Won transition |
| **Automations** | Any existing workflow triggered "on Stage = Closed Won" now fires prematurely if not re-scoped to fire post-approval instead |
| **Reports/Dashboards** | Forecast reports counting "Closed Won" revenue must now decide whether Pending-Approval deals count as committed or excluded — ambiguity here directly misstates forecast |
| **Notifications** | New notification needed to alert the approver; existing "deal closed" congratulation emails to the rep must be re-timed to fire post-approval, not on the initial stage drag |
| **Integrations** | Any downstream system (billing, contract e-signature, finance ERP) triggered off "Closed Won" status must be re-pointed to the *approved* state, not the raw stage change, or invoices could fire before approval |
| **Permissions** | New role/field needed for "Approver," with logic to prevent the deal owner from self-approving |

**Silent breakage risks:**
- Any automation still watching raw stage = Closed Won will fire early, sending premature customer-facing emails or provisioning access before approval is granted.
- Bulk import tools that set Stage = Closed Won directly (bypassing the UI stage-drag) could skip the approval gate entirely if the check is only enforced client-side.

**Edge cases:**
- **Approver on leave:** needs a fallback/delegate approver or auto-escalation after X days, or deals get stuck indefinitely in Pending.
- **Approver = Deal Owner:** must be explicitly blocked, or self-approval defeats the control's purpose.
- **Bulk import/update:** approval logic must be enforced at the database/API level, not just the UI stage-drag, or a CSV import can bypass it entirely.

**6+ test areas to prioritize:**
1. Stage-transition UI blocks direct jump to Closed Won without approval.
2. Approver notification fires correctly and to the right person.
3. Self-approval is blocked when Approver = Owner.
4. Bulk/API-based stage updates are also gated (not just UI).
5. Existing "Closed Won" automations/integrations re-tested for premature firing.
6. Forecast/report totals correctly exclude Pending-Approval deals from "Won" revenue.
7. Approver-on-leave / no-approver-assigned fallback behavior.
8. Rejection flow — deal reverts to prior stage cleanly with reason captured.

---

## Task 2.3 — Cross-Feature Dependency Reasoning (Top 3 highest-risk features)

**1. Deal stage automation (workflow rules triggered on pipeline movement)**
- *Why high risk:* Every downstream system — notifications, tasks, forecasting, sometimes billing — hangs off stage changes. If this breaks, the failure is invisible until someone notices a customer never got an invoice or a rep never got a follow-up task.
- *Testing type:* Integration testing (does the trigger fire across all dependent actions) + negative/edge-case testing (deleted dependencies, disabled templates).
- *Automation candidate:* Yes, high ROI — this is a repeatable regression check that should run before every release since it silently breaks with schema/config changes.

**2. Lead conversion / deduplication logic**
- *Why high risk:* Blast radius is data integrity across the entire database — duplicate or fragmented Contact/Account records corrupt every downstream report and campaign for the life of the CRM.
- *Testing type:* Data/functional testing with deliberate duplicate-input scenarios; boundary testing on matching logic (exact match, case sensitivity, alias domains).
- *Automation candidate:* Yes — scriptable via API test suite feeding known duplicate patterns and asserting record counts.

**3. Reporting/forecast accuracy (report sync & filter correctness)**
- *Why high risk:* Directly feeds executive decision-making; a misleading number here has business impact far beyond the CRM itself (wrong hiring/budget decisions off inflated forecasts).
- *Testing type:* Data consistency testing — cross-check report totals against raw record queries after each change type (add, edit, custom property).
- *Automation candidate:* Partially — the sync-timing checks are good automation candidates, but "is this number *meaningful* to a manager" requires human judgment, so full automation ROI is moderate, not high.

---

## Task 2.4 — CRM Comparison: QA Perspective

**Zoho CRM is harder to test thoroughly**, in my assessment, and here's why:

Zoho's configuration surface is deeper — it exposes granular Workflow Rules, Blueprint (process enforcement), Validation Rules, and Assignment Rules as separate overlapping systems that can all touch the same record simultaneously, creating far more permutations of "what actually determines this field's final value" than HubSpot's more unified Workflows engine.

Observability is also weaker in Zoho by default — automation logs are admin-gated and less granular in surfacing per-step failure reasons compared to HubSpot's Workflow History tab, which shows a cleaner per-enrollment execution trace. This means a QA engineer testing Zoho automations often has to infer failure from downstream symptoms (an email that didn't send) rather than reading a clear log.

That said, HubSpot's integration/marketplace surface area is larger for orgs using its full Marketing + Sales + Service hub combination, which introduces its own cross-hub testing complexity (e.g., a Marketing email triggering a Sales lifecycle stage change) — so for a heavily-integrated HubSpot instance, the calculus could flip. But for a like-for-like CRM-only comparison, Zoho's overlapping automation systems and thinner audit trail make it the harder platform to test with confidence.

---

# Part 3 — Test Planning & Organisation

## Task 3.1 — Test Plan: "Approval Before Closed Won" Feature

**Scope:** Deal stage-transition gating logic, approver notification/assignment, approval/rejection UI, forecast report exclusion of pending-approval deals, and re-testing of existing Closed-Won-triggered automations.

**Out of scope:** Underlying e-signature/billing third-party integrations' internal logic (only the trigger hand-off point is in scope); historical data migration of already-closed deals.

**Assumptions & dependencies:** Approver role/permission set already exists or will be created by Dev; test environment has at least 2 user roles (rep + manager) with distinct logins; existing stage-based automations are documented before testing begins so regressions can be identified.

**Top 3 risk areas:**
1. Bypass via bulk import/API — approval gate enforced only in UI, not at data layer.
2. Self-approval when Approver = Deal Owner.
3. Premature firing of existing Closed-Won automations/integrations before this change re-scopes their trigger.

**Entry criteria:** Feature deployed to staging; approver role configured; at least one existing Closed-Won automation present in test org to validate against regression.

**Exit criteria:** All 6+ test areas from Task 2.2 pass; zero critical/high bugs open; regression suite for existing automations passes unchanged (or documented/approved changes only).

**Environment & data setup:** Staging org with 2 test users (rep, manager) with correct roles; 3-5 seeded Deals at Negotiation stage in varying states (normal, missing custom fields, no owner) to cover edge cases.

---

## Task 3.2 — Test Suite Structure: Deal/Opportunity Module

```
Deal Module Test Suite/
├── Smoke/
│   ├── Create Deal (manual)
│   ├── Create Deal (via Lead conversion)
│   ├── Move Deal through pipeline stages
│   └── Close Deal Won / Lost
├── Regression/
│   ├── Field validation (required fields, data types)
│   ├── Duplicate detection on Deal name/Account combo
│   ├── Deal-Contact-Account association integrity
│   ├── Owner assignment & reassignment
│   └── Workflow automation triggers per stage
├── Edge Cases/
│   ├── Deal with no Owner
│   ├── Deal with empty required custom fields at Closed Won
│   ├── Reopen Closed Lost deal — Close Date behavior
│   ├── Bulk import bypassing UI validation
│   └── Concurrent edits (two users editing same Deal)
└── Integration/
    ├── Deal stage change → email notification
    ├── Deal stage change → task creation
    ├── Deal Closed Won → downstream report/forecast update
    └── Deal Closed Won → third-party integration trigger (if applicable)
```

**Automation candidates:** Smoke suite (high-frequency regression, stable UI) and Integration suite (API-testable, deterministic) — high ROI, run on every build.
**Manual-only:** Edge cases involving concurrent-user timing and exploratory UX judgment calls (e.g., "does this failure feel silent to a user") — low automation ROI, requires human judgment.

**Full example test case (step-table format):**

| Field | Value |
|---|---|
| **Test ID** | DEAL-EDGE-04 |
| **Title** | Deal cannot be marked Closed Won with mandatory custom field empty |
| **Preconditions** | Validation rule configured: "Contract Value" required before Closed Won |
| **Steps** | 1. Open a Deal at Negotiation stage with Contract Value field empty. 2. Drag/edit Stage to Closed Won. 3. Attempt to save. |
| **Expected Result** | Save is blocked; inline error highlights "Contract Value" as required; Deal remains at Negotiation stage |
| **Actual Result** | *(fill in after execution)* |
| **Priority** | High |
| **Type** | Edge case / Functional |

---

## Task 3.3 — Prioritisation Under Time Pressure (4 hours before release)

| Time | Focus |
|---|---|
| **Hour 1** | Smoke test: create Deal → move through pipeline → Closed Won/Lost, on both a fresh Deal and a Lead-converted Deal. Confirm core happy path isn't broken. |
| **Hour 2** | Approval-gate specific tests: block-on-empty-fields, self-approval block, approver notification firing. This is the actual feature being shipped — highest priority. |
| **Hour 3** | Regression on existing Closed-Won-triggered automations/integrations — confirm no premature or duplicate firing post-change. |
| **Hour 4** | Edge cases as time allows: bulk import bypass check (highest-risk untested item if time runs out), reopened-deal close date behavior, forecast report exclusion accuracy. |

**Explicitly deferring:** Full regression of unrelated modules (Contacts, Accounts, Reports outside the forecast view), performance/load testing, cross-browser UI testing, and exhaustive concurrent-edit scenarios.

**Risks I'd communicate as untested:** Bulk/API-based approval bypass is the single biggest known-unknown — if there's no time to verify it, I'd flag explicitly that data-layer enforcement is unverified and recommend either a fast manual spot-check by a dev or a documented risk acceptance before release.

---

## Task 3.4 — Defect Communication (Bug Report)

**Title:** Deal can be marked Closed Won via bulk import while bypassing the mandatory manager-approval gate

**Severity:** Critical
**Priority:** P1
**Justification:** This defeats the entire purpose of the approval-gate feature being released — it's not a cosmetic gap but a full control bypass with direct financial/compliance impact (revenue recognized and downstream billing/provisioning triggered without required sign-off).

**Environment:** Staging, Deal module, bulk import via CSV upload feature; test org with Approval-Gate feature enabled.

**Steps to Reproduce:**
1. Prepare a CSV of Deal records with Stage column set directly to "Closed Won" (skipping intermediate stages).
2. Use the standard bulk import tool to upload the CSV.
3. Review the imported Deal records.

**Actual Result:** Deals import successfully with Stage = Closed Won, with no approval record created and no approver notification sent — the record behaves identically to a properly-approved Closed Won deal in all reports and downstream automations.

**Expected Result:** Bulk import should either reject direct writes to "Closed Won" status (forcing deals to enter via the approval workflow) or automatically route imported "Closed Won" deals into the Pending-Approval state rather than the final state.

**Business Impact:** Any user with import access can silently circumvent the manager-approval control entirely, exposing the business to unauthorized revenue recognition, premature customer provisioning/billing, and compliance/audit risk — this is a release blocker for the feature as currently implemented.

---

*(End of assignment writeup)*
