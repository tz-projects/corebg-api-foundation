# API Governance — Communication Strategy

Outward-facing plan to introduce the API governance program to the wider organization and move the app-dev teams toward enforcement. Delivered as a sequence of emails, gated on evidence rather than a fixed calendar.

## Audience

Backend / app-dev teams are the primary remediators — the teams whose APIs are governed. Infrastructure teams are informed, since the gate lives in the deploy pipeline they don't own. Stakeholders and leadership consume the executive figures for progress and outcomes. Managers are cc'd on all team-facing waves.

## Standing principles

- **Channel is email** across every phase.
- **All APIs are treated the same.** One standard, one path, one timeline — no separation between new and existing APIs.
- **The non-conformance report is the primary way teams see their gap.** App-dev teams hold Consumer roles, so they get no self-service governance feedback in the Studio UI. The report is the source of truth. Pipeline-pushed violations are a supporting aid only — teams should never need to run the pipeline to know where they stand.
- **The enforcement date is an output, not an input** — derived from the remediation curve, not fixed upfront.
- **Enablement means understand the standard and remediate specs.** Pipeline integration is platform-owned; app teams handle only minimal configuration. No Jenkins-onboarding content in these comms.
- **Every message links the Remediation Toolkit** (Confluence — rules catalog) and points to a single support channel.

## Sequence

Phases run in order. Don't advance until the prior phase's success signal is met.

1. **Announcement** — introduce the program, the standard, and the full timeline. Nothing blocks yet. Can go as soon as the strategy is signed off. *Success signal: teams know it exists and the timeline; questions flow to support, not escalation.*
2. **Visibility** — each team receives its non-conformance report and sees exactly where it stands. Gated on the platform report being ready (next sprint). *Success signal: teams engage with their reports; violation counts start moving.*
3. **Supported Remediation** — toolkit, office hours, refreshed reports, visible progress. *Success signal: the remediation curve shows a defensible slope, making the gate date evidence-based.*
4. **Enforcement Warning** — dated gate notice with countdown reminders and the waiver path. *Success signal: remaining non-conformant APIs are remediating fast or have filed waivers.*
5. **Enforcement Live** — gate on in the pipeline; waiver path and support in place. *Success signal: blocks are handled via remediation or waiver, not escalation; the gate stays on.*

## Report and visibility dependency

The **non-conformance report is the primary visibility mechanism**, so Visibility cannot start until the platform (per-API) report is solid — expected next sprint (~3 weeks). This is the real constraint on how fast the program can move.

- **Platform report (per-API specifics)** → powers Visibility and Supported Remediation. In progress; the pacing dependency.
- **Executive report (overall figures)** → the leadership / stakeholder progress view from Remediation onward. Already solid.
- **Pipeline-pushed violations** → a convenience that surfaces violations to a team when its pipeline runs. Supporting aid only, never the primary channel.

Announcement can go now; Visibility waits on the report.

## Waiver / Exception process

Authored now, but surfaced only at Enforcement Warning — not during enablement. Early prominence invites teams to treat waivers as an opt-out instead of remediating. Held to the warning phase, it reads correctly: an escape hatch for genuinely blocked APIs, not an alternative to the work.

## Why this sequencing holds under pressure

For the enforcement-eager conversation: this path is the fast route. Announcing and gating before teams can see and fix their violations generates a wave of pipeline failures and escalations that take months to clear and burn credibility. Announcing now, gating the remediation ask on the report being real, and deriving the date from the curve gets to a gate that stays on — which is faster than a premature gate you have to walk back.

---

# Email content

Fill-ins are in brackets: `[DATE]`, `[Toolkit link]`, `[support: mailbox / Teams link]`, `[office hours schedule]`, `[waiver link]`.

## Phase 1 — Announcement

**To:** All app-dev teams
**Cc:** Engineering managers, infrastructure teams, stakeholders
**Subject:** Introducing API Governance — what's coming and when

Team,

We're introducing API governance across our API estate. Going forward, our APIs will be checked against a shared set of standards so that what we build is consistent, high-quality, and ready to publish.

**Why now.** Following the migration to Amazon API Gateway, our estate isn't yet in a consistent, governed state. Governance gives us a clear, shared definition of a well-formed API and moves the estate toward a published, discoverable state.

**How it works, briefly.** The standards are defined as a rules catalog in Swagger Studio (our governance platform) and documented in the Remediation Toolkit. Validation runs inside our deploy pipeline. The platform team owns that integration — there is minimal setup on your side.

**What's happening today: nothing blocks.** This is the introduction. Over the coming phases you'll see exactly where your APIs stand, get hands-on support to close the gaps, and receive clear, dated notice well before any enforcement begins.

**The path from here:**
- You'll receive a non-conformance report showing where each of your APIs stands.
- We'll run a supported remediation period — toolkit, office hours, and help closing gaps.
- Once remediation is well underway, we'll set and announce a dated enforcement gate, with plenty of notice.
- Enforcement goes live in the pipeline on the announced date.

The enforcement date isn't fixed today — it will be set based on how remediation progresses, and communicated well in advance.

**Learn more.** The Remediation Toolkit, including the full rules catalog, is here: [Toolkit link]

**Questions.** Reach us at [support: mailbox / Teams link].

Nothing is required from you right now — just a read so you know what's coming.

Thanks,
[Platform Team]

## Phase 2 — Visibility (per-team template)

**To:** [Team]
**Cc:** [Team manager]
**Subject:** Your API non-conformance report — where your APIs stand

[Team],

The governance rules are now running in report-only mode. Nothing blocks — the purpose of this phase is simply to show you where your APIs stand against the standards.

**Your report is attached / linked here: [report link].** It lists the specific rule violations for each of your APIs. This is the source of truth for your team's status.

As a convenience, violations are also sent to your team automatically whenever your pipeline runs — but you don't need to run anything to know where you stand. The report is the place to look.

**How to read it.** Each violation maps to a rule in the catalog, which explains what the rule means and how to fix it: [Toolkit link].

**What we're asking now.** Review your report and start scoping the work. Nothing is due yet — the goal at this stage is awareness. Support and office hours are coming in the next phase, and you can reach us any time at [support: mailbox / Teams link].

Thanks,
[Platform Team]

## Phase 3 — Supported Remediation

**To:** All app-dev teams
**Cc:** Engineering managers
**Subject:** Remediating your APIs — toolkit, office hours, and how to get help

Team,

Now that you've seen your non-conformance reports, this phase is about helping you close the gaps.

**What's available to you:**
- The Remediation Toolkit — rules catalog plus fix guidance, grouped by the most common violations: [Toolkit link]
- Office hours — [office hours schedule] — bring anything you're stuck on
- The support channel for questions at any time: [support: mailbox / Teams link]
- Refreshed non-conformance reports, so you can track your own progress

**How to approach it.** Work through your report rule by rule; the toolkit groups the common violations so you can clear several at once. Bring genuine blockers to office hours rather than sitting on them.

**Progress.** We're tracking estate-wide progress and will share it as we go. Teams that clear their APIs early help set the pace for everyone.

**What's next.** Once remediation is well underway across the estate, we'll set and announce a dated enforcement gate — with clear notice and a countdown.

Please work through your report and lean on office hours for anything blocking you.

Thanks,
[Platform Team]

## Phase 4 — Enforcement Warning

**To:** All app-dev teams
**Cc:** Engineering managers, infrastructure teams
**Subject:** API Governance enforcement goes live on [DATE] — action needed

Team,

Governance enforcement will go live in the deploy pipeline on **[DATE]**. From that date, APIs that fail the standards will be blocked at the pipeline gate.

**What this means for you.** By [DATE], your APIs need to pass the governance checks to deploy. Check your latest non-conformance report to confirm where you stand: [report link].

**If you can't remediate in time.** A waiver process is available for genuinely blocked cases: [waiver link]. Waivers are for real exceptions, not a way to defer the work.

**Reminders.** We'll send countdown reminders at 30, 14, 7, and 1 day before the gate goes live.

**Help is still here.** Office hours and the support channel remain available: [office hours schedule] / [support: mailbox / Teams link].

Please finish remediation before [DATE], or file a waiver if you have a genuine blocker.

Thanks,
[Platform Team]

---

*Countdown reminder (short follow-up, reused at T-30 / T-14 / T-7 / T-1):*

**Subject:** Reminder — API Governance enforcement goes live in [N] days ([DATE])

Team, a quick reminder that governance enforcement goes live on **[DATE]** — [N] days away. APIs that fail the standards will be blocked at the pipeline gate from that date. Check your latest report [report link], and if you have a genuine blocker, file a waiver [waiver link]. Questions: [support: mailbox / Teams link].

## Phase 5 — Enforcement Live

**To:** All app-dev teams
**Cc:** Infrastructure teams
**Subject:** API Governance enforcement is now live

Team,

As of today, governance enforcement is live in the deploy pipeline. APIs that don't pass the standards will be blocked at the gate.

**If your deployment is blocked.** Your pipeline output lists the failing rules, and the toolkit explains each fix: [Toolkit link]. Remediate and re-run.

**If you have a genuine blocker.** File a waiver: [waiver link].

**Help is still here.** Office hours and the support channel remain available: [office hours schedule] / [support: mailbox / Teams link].

If you're blocked, remediate or file a waiver, and reach out to support if you're stuck.

Thanks,
[Platform Team]