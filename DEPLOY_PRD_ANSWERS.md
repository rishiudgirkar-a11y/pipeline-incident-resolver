# Deploy PRD — AWS Data Pipeline Incident Resolution Agent

**Live product:** https://rishiudgirkar-a11y.github.io/pipeline-incident-resolver/
**Source:** https://github.com/rishiudgirkar-a11y/pipeline-incident-resolver

## 1. Go / no-go view

**Go — conditioned on a formal privacy review being completed before any real production incident data touches the agent.** Everything built and tested in this capstone was synthetic end to end; that's a real precondition, not a formality, and it's stated here rather than glossed over. Against the other five checks: eval cases pass (4 Pass / 1 Needs work, honestly recorded, not a suspicious clean sweep); the boundary is enforced in the product itself (the tooling-blind + SEV-1 case refuses on screen, and a mid-build stress test caught and fixed a real prompt-injection gap); three roles are named with real authority, not just "the team"; metrics have numeric thresholds, not vibes; and a rollback switch exists with a named owner. The only open item is the privacy review, which is why this is a conditional go, not an unconditional one.

## 2. Privacy & safety risks

The agent receives: the incident record (incident number, DAG name, pipeline, timestamp, severity), the matching EMR log snippet, and the internal error catalog/SOP/escalation policy text — sent to the Anthropic API using the visitor's own key, stored only in that browser's localStorage, never in the deployed files. The run log (every human action: approve/edit/escalate, with timestamps) lives only in that browser's session memory; nothing is persisted server-side or shared across visitors. If a key or a run were ever leaked, what's exposed is whatever incident/log data was in that session — which, in the capstone, is entirely fabricated. In a real pilot, that same exposure would include real DAG names, real pipeline naming conventions, and real (if redacted) error text, which is why: **this capstone is synthetic end to end; a real pilot would require a formal privacy review before any production log or incident data is sent to a third-party model.**

## 3. Human operating model

Three named roles: **Operator** — the L1 data-ops on-call engineer for the shift, who works the queue and clicks Approve/Edit/Escalate on their assigned incidents. **Escalation owner** — the L2 data engineers, who take cases the agent refuses or that L1 escalates, and have the authority to approve or override a recommendation. **Decision owner** — the L2 manager, who makes the call on low-confidence escalations and is the only one authorized to change the error catalog, SOPs, or system prompt. Review gate: every single agent recommendation is reviewed by a human before anything executes — there is no unreviewed path. Weekly ritual: the L2 manager reviews the week's Edit and Escalate entries from the run log for patterns (e.g., a recurring low-confidence case, the same SOP wording needing a fix).

## 4. Quality monitoring

**Quality:** approval rate — at least 95% of recommendations approved as-is or with only minor edits (matches the capstone's own SOP-match accuracy target). Breach: L2 manager reviews the miss pattern within 48 hours. **Value:** time per case — manual baseline is 1-2 hours per incident; target with the agent is under 15 minutes. Breach (no improvement after 2 weeks): pilot pauses for diagnosis, not silently continued. **Risk (hard zeros):** any unapproved ticket closure, or any missed escalation that should have fired — either one pauses the pilot immediately, no exceptions. **Drift defense:** the 5 eval cases are re-run monthly and after any prompt or catalog change — this isn't theoretical; during the capstone build, a stress test on the reasoning path and an adversarial-log test both surfaced real issues that a routine eval re-run would have caught.

## 5. User feedback plan

**Button data:** every Approve/Edit/Escalate action is already logged with a timestamp and the agent's original decision — this is free, always-on feedback the L2 manager reviews weekly, no extra tooling needed. **Reviewer check-ins:** a 15-minute weekly sync between the on-call rotation and the L2 manager, two standing questions: "did anything feel wrong that you approved anyway?" and "did you edit anything twice for the same reason?" **Eval samples:** the 5 eval cases (plus any new hard case found that week, following the same pattern used during Develop) are re-run monthly as a standing check, not just at launch. The decision owner acts on all three — pattern in edits → catalog/SOP fix; recurring "felt wrong" answers → tighten escalation triggers; eval regression → pause and fix before continuing.

## 6. Pilot plan

**Small:** the 3-person L1 on-call rotation for the data platform team, escalating to 2 named L2 data engineers and one L2 manager. **Scope in:** only the 5 known error patterns already in `error_catalog.json` (OOM, file-not-found, upstream timeout, S3 access-denied, generic task failure). **Scope out:** anything else — a new/unrecognized error type is out of scope for the pilot and goes straight to manual handling, not a forced agent guess. **Window:** 2 weeks, fixed. **Reversible:** the L2 manager can pause the pilot at any time; the manual process (checking EMR logs directly in the AWS console) never stops existing during the pilot and is the immediate fallback. **Observed:** every recommendation is reviewed before anything happens — no unattended runs. **Success:** ≥95% approved as-is or with minor edits, zero missed escalations, median time-to-resolution under 15 minutes, and the on-call engineers say — unprompted — that they'd want to keep using it.

## 7. Video outline

- **0:00–0:30 Intro:** who I am, the problem in one sentence — on-call engineers spend 1-2 hours manually reading EMR logs and hunting SharePoint for the right SOP on failures that follow known, repeatable patterns.
- **0:30–1:30 Problem & discovery:** the manual process today (ServiceNow ticket → AWS console → SharePoint → manual fix → manual ticket update), why it's slow and inconsistent across engineers, and the 90%+ SOP-match accuracy target chosen because guessing wrong on a production pipeline is expensive.
- **1:30–2:30 Live demo:** the happy path first — click the chip, run, watch it classify `EMR_OOM_002` with cited evidence, approve, watch it resolve — then the refusal: the SEV-1 tooling-blind boundary case, live, in red, refusing to guess.
- **2:30–3:30 Evidence:** the eval scoreboard (4 Pass / 1 Needs work, not a suspicious clean sweep), the improvement story (the adversarial-injection stress test that found a real gap, the fix, the re-test holding across four runs), and the one honest limitation stated plainly (literal-phrase matching over fuzzy matching).
- **3:30–4:00 Launch:** the pilot in one breath — 3-person on-call rotation, 5 known error types, 2 weeks, named owners, hard rollback — ending on the live link on screen: `rishiudgirkar-a11y.github.io/pipeline-incident-resolver`.
