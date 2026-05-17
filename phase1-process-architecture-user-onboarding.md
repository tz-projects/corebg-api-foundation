# Architecture - Verbose

Good moment to step into implementation. SharePoint + Power Automate is a sensible stack for this — it covers intake, state, automation, and audit without needing custom development. Let me lay out the implementation architecture, then flag what's design vs. what's tool-config (which we'll park).

---

**The architecture at a glance:**

Four layers:

1. **Form layer** — Power Apps customized SharePoint forms for intake.
2. **State layer** — One or more SharePoint lists as the system of record for requests.
3. **Automation layer** — Power Automate flows that run triage, validation, notifications, and (eventually) provisioning.
4. **Queue layer** — SharePoint list views + Teams for human triage and approval handling.

Each layer has a specific job; the cleanliness of the design comes from not mixing them.

---

**The form layer:**

Native SharePoint list forms are too limited for what you need — no conditional sections, weak validation, no clean way to handle the "request type drives everything" pattern. **Power Apps customized list forms** are the right choice. They give you conditional sections (show external-user fields only when "external" is selected, show bulk fields only when bulk is chosen), lookup validation (validate requester against AD via Microsoft Graph as they type), file attachment for bulk Excel uploads, and submission to the underlying SharePoint list.

There's a real design decision embedded here: **one form with strong branching vs. one form per request type**. My recommendation is one form with branching, anchored on the "Request type" dropdown at the top. Reasons: requesters only need to bookmark/find one form; the conditional logic is well-defined (you already mapped it); maintenance is in one place. The trade-off is that the Power App becomes more complex to build and test — but it's a one-time cost.

Each submission lands as a new item in the requests list.

---

**The state layer:**

The single most important implementation principle: **one place holds the canonical state of a request**, and everything else is a view or a downstream action. Today, state is fragmented across SharePoint and ServiceNow with humans copying between them. That's what we're undoing.

The state layer is a **SharePoint list — call it "SmartBear Access Requests"** — where each item is one request. Columns include:

- Request ID, request type, submission timestamp
- Requester identity (auto-captured from sign-in)
- All form fields (form binds directly to columns)
- **Status** — controlled vocabulary, drives the entire state machine: *Submitted, In Triage, Needs Clarification, Rejected, Flagged for Human Triage, Pending Approval, Approved, Pending Provisioning, Provisioned, Closed, On Hold*
- Triage outcome reasons (free text or structured codes)
- Audit fields — who/what changed status, when, with what comment

For **bulk requests**, the cleanest pattern is a parent-child list relationship: the bulk request is one item in a parent list ("Bulk Requests"), and each row in the uploaded Excel becomes a child item in the main Access Requests list, linked back to the parent. That way, each user in the bulk gets independent triage, independent state, and independent audit — but the requester sees them as one batch. A flow parses the Excel on submission and fans out the child items.

The Status column is the heartbeat of the whole system. Every state transition is governed by flow logic, not by humans freely editing the field. (You'd lock the column from manual edits except for designated roles.)

---

**The automation layer:**

This is where Power Automate does the work. The key flows:

**Flow 1: Triage flow.** Triggered when a new item is created in the Access Requests list. Runs the deterministic checks in order:

- Form completeness (mostly belt-and-suspenders — the form should enforce required fields, but the flow re-checks)
- Requester identity validation — Microsoft Graph lookup against your tenant's AD
- Duplicate detection — SharePoint list query for pending requests against the same target user
- Request-type-specific lookups:
  - SmartBear User Management API call to check if the target user already has a license, what type, etc. — via an HTTP action with an API key stored in Azure Key Vault (or a custom connector)
  - AD lookup for the target user if internal
  - Sponsor lookup if external

Each check writes its result to the request item. At the end, the flow sets Status to one of: *Needs Clarification*, *Rejected*, *Flagged*, or *Pending Approval*.

**Flow 2: Bulk parser flow.** Triggered when a bulk request item is created with an Excel attachment. Reads the file, validates the template structure, fans out child items into the main list. Each child item then triggers Flow 1 independently.

**Flow 3: Notification flow.** Triggered on any Status change. Sends appropriate emails/Teams messages based on the new status — to requester (always), to target user (on completion), to platform team (on flagged or pending actions).

**Flow 4: Provisioning flow.** Triggered when Status moves to "Pending Provisioning" after approval. Calls SmartBear's User Management API to add the user / change role / remove user. Updates Status to "Provisioned" on success, or "Provisioning Failed" with details on error.

The actual flow definitions, custom connectors, API auth handling, and Key Vault setup are tool-config — parked for that conversation. The *design* says: these four flows exist, with these responsibilities, with this triggering pattern.

---

**The queue layer:**

Two queues that the platform team operates:

- **Human Triage Queue** — a SharePoint list view filtered to Status = "Flagged for Human Triage." Platform team members work this view, open each request, make a decision, and use a custom button (Power Apps action or quick edit) to set the status to Pending Approval, Rejected, or Needs Clarification with a reason.

- **Approval Queue** — a list view filtered to Status = "Pending Approval." Same model. Approvers open the request, see all the triage results, make the call.

In the pilot, both can be operated from the SharePoint list views directly. As volume grows, you can layer **Teams adaptive cards** on top — when a request hits a queue, a card posts to a Teams channel with key details and inline approve/reject buttons. The platform team works from Teams instead of clicking into SharePoint each time. The underlying state is still the SharePoint list.

You can also use **Power Automate Approvals** (the built-in approval action) for the approval step — it generates assignable approval tasks with built-in audit trail and notifications. Whether to use built-in Approvals or roll your own with status fields depends on how much complexity you need in the approval logic. Built-in Approvals is simpler; custom status handling is more flexible. Either works.

---

**Audit and notifications:**

Audit is handled two ways:

- **Per-request audit** — each request item carries its full history in version history (SharePoint tracks every change) plus a structured audit log field (multi-line) where flows append timestamped entries on each state transition.
- **Cross-request audit** — a separate "Access Decisions" list that captures every approval/rejection/provisioning event with consistent fields, for reporting. Easier to query than version history.

Notifications via Power Automate — emails for status changes, Teams cards for queue arrivals, escalation if items sit in a queue too long.

---

**Pilot vs. scaled implementation:**

The same architecture supports both. In the pilot:

- Flows are simpler. Triage might just check completeness and duplicate; the SmartBear/AD/Ping lookups can be manual in a "Flagged for Human Triage" step.
- Provisioning is manual — the platform team reads the approved request and does the SmartBear API call manually (or uses the SmartBear UI).
- One platform team member operates everything from a single SharePoint view.

As volume grows:

- Triage checks get progressively automated by adding HTTP actions to Flow 1.
- Flow 4 (provisioning) goes from manual to fully automated when you're confident in the rules.
- Teams adaptive cards reduce click-through time for the team.
- Reporting dashboards (Power BI on the audit list) start mattering.

The architecture doesn't change. You're just turning on automations that the design always anticipated.

---

**A few specific things to be careful about:**

- **Form binding to list columns.** Power Apps customized forms are tightly coupled to the list schema. Adding fields later means form changes too. Worth investing in a clean schema upfront.
- **API key storage for SmartBear and any other system.** Never in the flow itself. Azure Key Vault, referenced from flows via secure variables. Tool-config detail, but flagging because it's a frequent mistake.
- **Flow versioning and testing.** Production flows handling provisioning need a non-prod environment to test changes in. SharePoint + Power Automate has reasonable patterns for this (separate environments, solution packaging), but it's not as mature as proper CI/CD. Worth planning.
- **Power Automate run history retention is limited** (28 days by default). For longer audit retention, the audit log fields on the request item are your source of truth, not flow run history.
- **License costs.** Power Automate has per-user vs. per-flow licensing modes, and some operations (HTTP connector, custom connectors) may require premium licenses. Worth confirming with whoever owns the M365 tenant before committing. Tool-config, but flagging because it can derail timelines.

---

**Decisions for you:**

1. Power Apps customized SharePoint forms as the form layer — agree, or do you want to look at alternatives (Microsoft Forms, standalone Power App, etc.)?
2. One form with conditional branching vs. one form per request type — I'm recommending one form. Push back if you'd rather have separate forms per type.
3. Parent/child list pattern for bulk requests — agree, or another model in mind?
4. Use built-in Power Automate Approvals for the approval step, or roll custom status-driven approval handling? (We can park this until we design the approval step itself.)
5. Anything in this architecture you'd want to challenge or reshape before we go deeper into any specific layer?

Once you settle these, the natural next step is to go back to the workflow design and pick up where we left off — the approval step after triage. The implementation conversation can resume layer-by-layer whenever a design decision needs implementation context.


# Implementation concise, less verbose

## 3. SmartBear APIs involved

**Triage stage (read-only):**
- `GET /user-management/v1/orgs/{org}/members` — list members, license types
- `GET /user-management/v1/orgs/{org}/members/{email}` — check specific user
- `GET /apis/{owner}` — list APIs owned by a user (for offboard content check)
- `GET /domains/{owner}` — same for domains
- `GET /projects/{owner}` — same for projects

**Provisioning stage (write):**
- `POST /user-management/v1/orgs/{org}/members` — add user with role
- `PUT /user-management/v1/orgs/{org}/members/{email}/role` — change license type
- `DELETE /user-management/v1/orgs/{org}/members?emails={email}` — remove user
- Resource transfer APIs — for offboard with content transfer (need to confirm exact endpoint; flag for SmartBear docs review)

All require API key with org owner privileges. Stored in Azure Key Vault, never in flows.

---

## 4. State machine

States for any request:
`Submitted → In Triage → [Needs Clarification | Rejected | Flagged for Human Triage | Pending Approval] → Approved → Pending Provisioning → Provisioned → Closed`

Side states: `On Hold`, `Provisioning Failed`.

---

## 5. SharePoint list schema (POC)

One list: **SmartBear Access Requests**
- Standard columns above (Request ID, type, requester, status, etc.)
- Type-specific fields stored together — leave blanks for non-applicable
- Multi-line text fields: Business justification, Triage notes, Audit log
- Status column: choice field, locked from manual edit

Locking the Status column is permission-based — flows run as a service account that can edit; humans can't. Tool-config detail, parked.

---

## 6. Power Automate flows (POC scope)

**Flow 1: Triage** (one flow, branches by request type internally)
- Trigger: item created
- Common checks → type-specific checks → set status
- Write triage results to item

**Flow 2: Notifications**
- Trigger: status changed
- Branch on new status → send appropriate email/Teams message

**Flow 3: Provisioning** (manual trigger initially for POC safety)
- Trigger: status = Pending Provisioning (or button click for POC)
- Branch by request type → call appropriate SmartBear API
- Update status to Provisioned or Provisioning Failed

Bulk parser flow — out of scope for this POC since you scoped to single-user cases.

---

## 7. Queue handling (POC)

Two SharePoint views:
- Human Triage Queue: Status = Flagged
- Approval Queue: Status = Pending Approval

Platform team works these views. Approval mechanism — for POC, simplest is to use built-in Power Automate Approvals action triggered when status hits Pending Approval. Approvers get a task; approval sets status to Approved.

---

## 8. Build sequence (recommended order)

1. SharePoint list schema
2. Power Apps customized form (one form, conditional sections)
3. Manual test: submit forms, verify items land correctly
4. Flow 1 (Triage) — start with common checks only, add SmartBear API calls incrementally
5. Flow 2 (Notifications)
6. Approval mechanism wired in
7. Flow 3 (Provisioning) — last, because it's the destructive one
8. End-to-end test for each of the three use cases

---

## 9. Open items I need from you

1. **Non-prod SmartBear environment confirmed?** Without this, provisioning testing risks real licenses.
2. **API key procured** with org owner privileges?
3. **Power Automate Premium licensing** confirmed in tenant?
4. **Service account / app registration** for flows authenticated to Graph + SmartBear?
5. **Approval step** — single approver (platform team lead) for POC, or you want to test multi-approver routing?
6. **Content transfer endpoint** for offboard — I'll verify in SmartBear docs once you confirm we're proceeding, but flagging the gap.

Once you have the open items resolved, the build sequence above is the path. Want me to drill into any specific stage before you start?


# Implementation

## Field review

**Common observations across all three forms:**

- **Requester info is auto-captured from sign-in.** Don't ask the submitter to retype their name, Windows account, work email. Only useful field on the requester side: relationship to the user.
- **"Requesting for yourself or on behalf"** is derivable. If target email = signed-in email, it's self; otherwise, on behalf. Don't ask.
- The "requester/relationship" group becomes just one field: "Your relationship to the user" (choice: self / manager / team lead / peer / other).

---

**Onboard new user — field review:**

| Field                                       | Keep? | Note                                     |
| ------------------------------------------- | ----- | ---------------------------------------- |
| User type (internal/external)               | Yes   | Drives provisioning path                 |
| Internal: Windows account, name, work email | Yes   | Required for AD lookup                   |
| External: name, work email                  | Yes   | Windows account N/A — drop for externals |
| License type                                | Yes   |                                          |
| Business justification                      | Yes   |                                          |
| Requesting for self or behalf               | Drop  | Derive from sign-in                      |
| Requester name/account/email                | Drop  | Auto-captured                            |
| Requester relationship                      | Keep  | Useful for triage                        |

**Missing for external:** sponsoring internal manager (name + email), external user's organization. These were in earlier drafts — should still be required.

---

**Offboard user — field review:**

| Field                                       | Keep?                 | Note                                                                                                                                     |
| ------------------------------------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| User type                                   | Yes                   |                                                                                                                                          |
| Internal: Windows account, name, work email | Yes                   |                                                                                                                                          |
| Reason (choice + other)                     | Yes                   |                                                                                                                                          |
| Has content been transferred? (yes/no)      | Reframe               | At submission, nothing's transferred yet. Reframe as: "If user owns content, transfer to:" (optional). Triage verifies actual ownership. |
| Transfer target details                     | Yes (if owns content) |                                                                                                                                          |
| Requester relationship                      | Keep                  |                                                                                                                                          |
| Requester info                              | Drop                  | Auto-captured                                                                                                                            |

License type fields don't belong here — correct, you didn't include them.

---

**License type change — field review:**

| Field                                       | Keep? | Note                                            |
| ------------------------------------------- | ----- | ----------------------------------------------- |
| User type                                   | Yes   |                                                 |
| Internal: Windows account, name, work email | Yes   |                                                 |
| Does user have a SmartBear license?         | Drop  | Derive in triage                                |
| Current license type                        | Drop  | Derive in triage                                |
| Desired license type                        | Yes   |                                                 |
| Reason for change                           | Merge | Combine with business justification — one field |
| Business justification                      | Yes   | Merged                                          |
| Requester relationship                      | Keep  |                                                 |
| Requester info                              | Drop  | Auto-captured                                   |

---

## Triage rules per request type

**Onboard new user:**

Common: form completeness, requester is active employee (Graph), no duplicate pending request.

Type-specific:
- Target user not already in SmartBear → Rejected (file change/access request instead)
- Internal: target's Windows account exists & active in AD → Needs Clarification on fail
- External: target identity exists in Ping → Needs Clarification on fail (open question, see below)
- External: sponsor is current active employee → Needs Clarification on fail
- External: sponsor seniority/relationship reasonable → Flagged (human judgment)
- Seat available for requested license type → Flagged (procurement path) on fail

**Offboard user:**

Common: completeness, requester active, no duplicate.

Type-specific:
- Target user exists in SmartBear → Rejected on fail
- Content ownership check (SmartBear API: GET user's APIs/domains/projects)
  - If owns content AND no transfer target on form → Flagged
  - If owns content AND transfer target specified → validate target is current Designer in SmartBear → Needs Clarification on fail
- No conflicting pending requests for same user → Flagged

**License type change:**

Common: completeness, requester active, no duplicate.

Type-specific:
- Target user has a license → Rejected on fail (file onboarding instead)
- Desired type != current type → Rejected on fail (no-op)
- Consumer → Designer: Designer seat available → Flagged on fail
- Designer → Consumer: target currently has edit/comment access on resources → Flagged (warn requester of impact, requires acknowledgement)

---

## External user identity validation — recommendation

If Ping exposes an admin/SCIM API and your flows can be granted access, the triage check mirrors the Graph pattern for internals: lookup the external user's email in Ping, confirm identity exists and is active.

If Ping API access isn't available for the POC:
- Every external user request gets routed to **Flagged for Human Triage**
- Platform team manually verifies in the Ping admin console
- Once Ping API integration is set up, switch the check to automated

This isn't a workflow change — it's a triage automation gap, handled the same way as any other deterministic-vs-judgment check. The design accommodates both.

**Action item:** Confirm with whoever owns Ping in the org whether the workflow can be granted API access for read-only identity lookups. Tool-config detail, but it determines POC scope for external users.

---

## Two small open items

1. **External user organization field** — not on your draft. Worth capturing for audit purposes ("which external party does this person belong to?"). Add to onboard form for externals?
2. **Sponsor field for external onboards** — was in earlier draft, not in your current list. Confirm whether you want it.

Once you confirm those, the form spec is locked and we can move to the SharePoint schema.





**Tooling and access you need for the POC:**

Microsoft side:
- M365 tenant with SharePoint, Power Apps, Power Automate
- **Power Automate Premium license** — required for the HTTP connector to call SmartBear API (the standard license won't cut it)
- A SharePoint site to host the list and form
- Entra ID/Graph API permissions for AD lookups (Directory.Read.All scope or equivalent)

SmartBear side:
- Swagger Studio admin access
- API key with org owner privileges (for User Management API)
- **A non-prod SmartBear org or tenant for testing** — critical, you don't want POC flows hitting real licenses. Check with SmartBear if you have a sandbox entitlement.

Secrets/security:
- Azure Key Vault (or equivalent) to hold the SmartBear API key — don't embed it in flows
- A service account or app registration in Entra for the flow to authenticate as

Optional but useful:
- Power Platform CLI (`pac`) — for exporting/importing solutions, source control
- PnP PowerShell — for scripting SharePoint list creation
- VS Code — for editing JSON/PowerShell
- Postman or similar — for poking at SmartBear API before wiring it into a flow

**Before installing anything, confirm:**

1. Do you have Power Automate Premium licensing in your tenant, or need to procure it?
2. Do you have a non-prod SmartBear environment for testing?
3. Who owns the Entra app registration / Graph permissions — can you get them, or is it a separate request?

Those three are the gating items. Tool installs are trivial; access is what blocks POCs.



# Phase 1 end - end design

Now I have the full picture. This changes the framing significantly — because the pipeline is where real enforcement happens, not API Hub alone. API Hub's governance and publishing gate is the prerequisite to the pipeline passing. The two are linked in sequence. Let me give you Phase 1 as a whole.

---

**What Phase 1 is actually trying to accomplish**

Phase 1 has one governing objective: establish the governance foundation and the pipeline enforcement mechanism without breaking existing deployments, while giving teams a defined and supported path to compliance. Everything in Phase 1 either enables that, or protects against doing it badly.

There are several workstreams that need to run in parallel, and a very deliberate sequencing of when enforcement actually bites.

---

**Workstream 1: Understand the blast radius before anything else**

Before any pipeline changes, before any team communication, you need to know what you're dealing with. API Hub's existing rules are already evaluating your 600+ APIs. The first order of business is determining whether API Hub gives you any aggregate view of violations — a reporting page, an export, an API you can query. If it does, you pull that data and understand the distribution: which rules are producing the most violations, which teams are cleanest, which are worst, and critically, how many APIs are in a state where they could realistically get to published within a reasonable timeframe.

If API Hub doesn't give you aggregate visibility at scale, that is itself a gap you need to solve before proceeding — because you cannot design a remediation plan without knowing what you're remediating.

---

**Workstream 2: Finalize the governance rule set and severity model**

Right now your active rules are all set to Error. That's a blunt instrument. Before teams engage with this, you need to make a deliberate decision about which rules are hard gates versus informational.

The way I'd think about this: Error severity means "you cannot publish without fixing this." Warning means "you can see the violation, it's recorded, but it doesn't block you." That distinction is your phased enforcement lever.

For Phase 1, I'd recommend a tiered approach. Rules that are foundational — operationId present, title present, summary present, one tag per operation — are reasonable hard gates from the start. These are the basics that any well-formed spec should have, and they're relatively low-lift to fix.

Rules that require significant effort on migration-era specs — all model properties must have examples, API contact present, API license present, local definitions only — should start at Warning in Phase 1. Teams are made aware, violations are visible, but they're not blocked. These graduate to Error in a later phase with a defined timeline.

The custom naming and versioning rules are a special case. Before those go on in any form, the standards they enforce need to be officially documented and ratified — not just drafted. If a team gets a governance violation for a naming convention they've never seen, you've lost credibility. These rules should be introduced in Warning mode initially, after the standards are published and communicated.

---

**Workstream 3: Resolve the spec sync question — this is the most important design decision in Phase 1**

Here's the workflow problem you're creating with the pipeline checks. Today, specs live in GitHub alongside backend code. Developers update specs in GitHub. The pipeline deploys. That's the current flow.

You want to introduce: pipeline checks whether the spec in API Hub is published, and whether the published spec matches what's in GitHub.

Those two checks together imply that API Hub needs to stay in sync with GitHub. The question is who or what is responsible for that sync — and this is a design decision you need to make explicitly, because it determines the developer workflow and the failure modes.

There are two basic models. The first is developer-managed sync — teams update their spec in GitHub, and they're also responsible for keeping API Hub updated. The pipeline then checks whether those two things match. The problem with this model is it creates a manual step that teams will forget or skip, and you'll get a lot of pipeline failures that are essentially sync failures rather than governance failures. It adds friction without adding a lot of value.

The second is pipeline-managed sync — the pipeline itself takes the spec from GitHub, pushes it to API Hub, governance runs, and if it passes the publishing gate, the pipeline continues deployment. If governance fails, the pipeline stops and reports the violations. In this model, API Hub becomes a governance checkpoint in the pipeline rather than a tool developers interact with directly for spec management. API Hub reflects what's in GitHub, not the other way around.

The second model is cleaner and more aligned with how these teams already work — they live in GitHub. It removes the "keep two things in sync manually" problem. It also means the pipeline is doing the heavy lifting, which is appropriate for Phase 1.

There's a longer-term question about whether API Hub eventually becomes the design tool and GitHub becomes downstream — but that's a later phase conversation. For now, GitHub is source of truth, and API Hub is the governance gate and catalog.

---

**Workstream 4: Pipeline integration design and phased enforcement**

The pipeline checks you described — is API onboarded, is it published, does published spec match GitHub — need to come in with a critical safety mechanism: a notify-only mode before a blocking mode.

On day one, the pipeline checks run, they log results, they notify the team if something fails — but they do not block the deployment. This gives you a period where teams can see what would have failed without their deployments being broken. This is your observation window.

After a defined period — probably four to eight weeks depending on team volume — you flip the switch and the pipeline actually blocks on failure. By that point teams have had visibility into their violations, they've had a remediation runway, and the enforcement is not a surprise.

The notification on failure needs to be specific and actionable. "Your API failed governance" is useless. "Your API swagger-petstore failed the following governance checks in API Hub: operationId missing on 3 operations, API contact field not populated. Fix these violations and republish before deployment will proceed" is what teams need.

There's also an important edge case to design for: what happens when a net-new API is being deployed for the first time and it's not in API Hub at all? That's the onboarding trigger. The pipeline should catch this and route it — either block with instructions to submit an onboarding request, or integrate with whatever onboarding process you've designed. This connects directly back to the user lifecycle and API onboarding work.

---

**Workstream 5: Team communication and remediation support**

Teams should not hear about this from a broken pipeline. They should hear about it from the platform team before the pipeline changes go in.

The communication needs to answer four questions. What is changing and why. What violations do you currently have — which means you need the blast radius assessment done first. What you need to do to remediate, with specific guidance per rule. And by when, with a clear timeline for when blocking enforcement starts.

The support model matters here. 85 teams is a lot of surface area. You need either self-service remediation documentation — per-rule guidance on how to fix the violation — or a triage channel where teams can ask questions, or both. Without that, the platform team gets flooded with questions it has to answer one by one.

---

**Workstream 6: The grandfathering decision**

You need to make an explicit decision about whether existing APIs get treated differently from net-new APIs. My recommendation is yes — existing APIs get a remediation window with a defined end date, and net-new APIs are held to full governance from day one. This prevents teams from feeling punished for a migration they did before governance existed, and it gives you a clean enforcement line going forward.

Defining what "net-new" means is important. Likely it's any API spec created in API Hub after a certain cutoff date, or any existing API that goes through a major version change after that date. Minor revisions to existing APIs during the remediation window probably get the same grace as the base spec.

---

**How Phase 1 sequences**

First: blast radius assessment and rule severity model finalization — internal, no team communication yet.

Second: document and ratify the naming and versioning standards, define the custom rules against them.

Third: finalize the pipeline integration design, build the pipeline checks in notify-only mode.

Fourth: communicate to teams — here are your current violations, here is what you need to fix, here is the timeline.

Fifth: enable pipeline checks in notify-only mode — teams start seeing real pipeline feedback.

Sixth: after the remediation window, flip to blocking mode.

Only after Phase 1 is stable do you move to expanding the rule set, tightening severity on the Warning-tier rules, or tackling the more complex governance scenarios.

---

That's the full Phase 1 picture. Where do you want to go deeper — the spec sync model, the pipeline enforcement design, the rule severity decisions, or the team communication and remediation approach?


about api spec source of truth

So the refined conclusion is: API-owning teams get Consumer access. The pipeline runs as a service account with write access. The platform team's access level needs a conscious decision on whether they're permitted to edit specs outside the pipeline in exceptional circumstances. And that exception policy, whatever it is, should be documented.
This also directly shapes the license type decisions in your user lifecycle work — Consumer is the default for anyone onboarding from an API-owning team, and the Designer license becomes largely a platform team concern. 