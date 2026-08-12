# Design PRD — AWS Data Pipeline Incident Resolution Agent

**Note on this document:** this Design PRD was written after Develop and Deploy, reconstructed from the shipped system rather than drafted on paper beforehand. The blueprint decisions below are exactly what's live in `index.html` and the `policies/` and `data/` files today — nothing here is aspirational or backfilled to look cleaner than the build. Where a decision changed after Design (the two eval cases added post-review), that's noted rather than smoothed over.

## 1. Agent role

The agent is hired to triage AWS data pipeline incident tickets for the on-call data engineer, within the five known error patterns in `error_catalog.json`, escalating to a human whenever confidence is low, evidence is missing or untrustworthy, the incident is high severity, the fix falls outside a standard restart/rerun, the incident record and log don't correlate, remediation was attempted and failed, or the log itself contains an instruction aimed at the agent rather than normal log text. It only ever recommends — a human approves, edits, or escalates every single case.

## 2. Target workflow

1. A ServiceNow-style incident lands in the queue (or is picked via a demo chip).
2. The agent fetches the matching EMR log (`fetch_emr_log()`), the error catalog, the relevant SOP (`get_sop()`), and the escalation policy.
3. The agent runs the five pre-recommendation checks and, if all pass, classifies the failure and proposes a fix; if any check fails, it escalates instead, citing which check failed.
4. The engineer reviews the labeled output and clicks Approve, Edit, or Escalate.
5. On Approve, the agent executes the steps (simulated) and waits on `confirm_recovery()`.
6. Only a positive recovery signal lets the agent call `update_servicenow()` and mark the ticket resolved; a negative or missing signal escalates as a failed remediation instead, and the ticket is never silently closed.

## 3. Agent loop

**Observe** — read the incident record, the fetched log, the full error catalog with precedence rules, all five SOPs, and the escalation policy, all named explicitly in the context sent to the model. **Decide** — apply catalog precedence when signals conflict, run the five pre-recommendation checks, and choose recommend vs. escalate. **Act** — produce either a labeled review packet (error code, confidence, check results, matched SOP, recommended steps, citations) or a labeled escalation notice naming the trigger and reason string — never a wall of text. **Check** — this is a two-stage gate, not one: the human approval gate (Approve/Edit/Escalate) sits before anything executes, and `confirm_recovery()` sits as a second, independent system gate after execution, so an approval alone can never write `resolved`.

## 4. Inputs and context

**Facts:** `data/incidents.json` (15 synthetic ServiceNow-style incidents) and `data/logs.json` (matching EMR/Airflow log snippets, including one empty log, one access-denied log, and one log with two co-occurring error signatures). **Rules:** `policies/error_catalog.json` (5 known error codes with precedence ranks and precedence rules for co-occurring signatures) and `policies/escalation_policy.md` (7 escalation triggers, 5 pre-recommendation checks, 1 post-execution check, each with its own reason string). **Examples:** `templates/servicenow_update_template.json` (target shape of the closed-ticket payload) and `examples/sample_review_packet.md` (target shape of a decision output). All four are named explicitly in the prompt sent to the model — "customer data" was never good enough; every fact traces to a specific file.

## 5. Tools or simulated tools

All four tools are simulated, standing in for real integrations without needing a live AWS/ServiceNow connection: `fetch_emr_log(incident_number)` — returns the matching row from `data/logs.json`, or an empty/access-denied result for the missing-data and tooling-blind cases. `get_sop(error_code)` — looks up the matching runbook from the `SOPS` object. `confirm_recovery()` — returns the incident's `simulated_recovery` field (success, failure, or null) after approved steps are "executed." `update_servicenow(incident_number, resolution_summary)` — the only tool with a real consequence in the demo (marks the ticket resolved), gated so it can only ever be called after a positive `confirm_recovery()` signal. Every tool maps to a step in the loop; nothing was added that the loop doesn't use.

## 6. Memory decision

No memory across incidents, on purpose. Each incident is evaluated fresh, from its own incident record, log, and the shared static catalog/policy files — nothing the agent has seen on a prior incident carries forward into the next one. This was a deliberate scope decision, not a default: a safety-critical triage agent that "learns" patterns from its own prior recommendations risks drifting away from the catalog's actual precedence rules without anyone deciding that on purpose. The Path B stretch considered adding visible memory of edit/escalate patterns (B3), but the team chose B4 (a second reviewer agent) instead, precisely because a second independent check on each individual decision seemed like a better use of the added complexity budget than carrying state across incidents.

## 7. Output format

Every case renders as labeled fields, never free text: on a recommend, ERROR CODE, CONFIDENCE, the 5 CHECK results, MATCHED SOP, RECOMMENDED STEPS, and citation tags naming the exact log line and SOP section used, plus a WHY line stating the reasoning in one sentence. On an escalation, a red STATUS: REFUSED-ESCALATE banner, the reason string from the escalation policy, and (in the boundary case) the raw tool-call result that caused it (e.g. `fetch_emr_log() -> AccessDenied`). A reviewer can judge either output in under a minute without reading a paragraph of prose.

## 8. Escalation rules

Seven triggers, each stopping the agent from recommending and firing `escalate(incident_number, reason)` instead of `update_servicenow()`: (1) confidence below 0.95, (2) incident or log data missing/unreadable, (3) high severity (SEV-1/P1 or possible data loss), (4) the matched SOP's fix needs more than a standard restart/rerun, (5) the incident and log don't correlate on DAG name or failure time, (6) approved steps were executed but `confirm_recovery()` came back negative, and (7) — checked first, on every run, regardless of what else is found — the log text itself contains something that reads like an instruction to the agent rather than normal log content. Trigger 7 was not part of the original design; it was added during Develop after a stress test found the agent's behavior was inconsistent across identical runs on an injected-instruction log, and it's included here because escalation rules are a Design-phase artifact that should reflect what's actually enforced today, not a historical snapshot.

## 9. Human approval point

The gate sits immediately after the agent's recommendation and before anything with a real consequence: Approve, Edit, or Escalate, on every single case, no exceptions. Even after Approve, the ticket does not close on the human's word alone — `confirm_recovery()` has to return success first, which means the actual consequence (a closed ticket) sits behind two gates, not one: a human decision and an independent system check that the fix actually worked.

## 10. Initial eval plan

Five cases were planned and built initially: a happy path (clean OOM, high confidence), a missing/unreadable log (must escalate, never guess), a low-confidence match (must escalate on the 0.95 floor), a boundary case combining tooling blindness with high severity (must refuse and escalate, never call `update_servicenow()`), and a cascading failure requiring root-cause precedence over a more obvious symptom. Every case was scoped to a specific pre-recommendation check or escalation trigger, and the boundary case was chosen specifically to test the refusal behavior the whole design leans on, not just the happy path.

**What the initial plan missed:** none of the five originally-planned cases exercised trigger 5 (DAG/time correlation failure) at all — every seeded incident had a perfectly aligned DAG name and timestamp between the incident record and its log. That gap sat undetected through Develop and the first Deploy review, and was only caught in a later review pass, which is exactly the kind of miss an eval plan should be checked against after the fact rather than assumed complete. Two cases (E-06: wrong DAG's log returned; E-07: stale log from an earlier run of the same DAG) were added post-review, both built so the log content alone would look like a clean, high-confidence match — proof that the correlation check, not pattern-matching, is what has to catch it. Both were run live against the deployed model and passed. Full detail in `DEVELOP_PRD_ANSWERS.md`, sections 4–5.

## Build-Readiness Gate — self-check

1. **Can you state the agent's job in one sentence?** Yes — see section 1.
2. **Can you name the file that grounds each fact the agent uses?** Yes — `data/incidents.json`, `data/logs.json`, `policies/error_catalog.json`, `policies/sops/*.md`, `policies/escalation_policy.md`, all named explicitly in the prompt.
3. **Do you know exactly what happens when data is missing?** Yes — pre-check 2 fails, the agent escalates with reason "missing or unreadable log/incident data," and this is eval case E-02.
4. **Is there a human gate before anything with consequences?** Yes, and it's two gates deep — human Approve, then a system-level `confirm_recovery()` check, both before `update_servicenow()` can ever be called.
5. **Does one eval case test the boundary the agent must refuse?** Yes — E-04 (tooling-blind + SEV-1), and as of the post-review addition, E-06 and E-07 test a second, previously-uncovered boundary (correlation failure) as well.
