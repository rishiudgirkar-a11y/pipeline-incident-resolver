# Pipeline Incident Resolver

An agent that triages AWS data pipeline failures the way an on-call engineer would — reads the incident and the log, matches it against a known error catalog and its runbooks, and recommends a fix. It never executes anything on its own: every recommendation goes through a human Approve / Edit / Escalate gate, and the agent refuses and escalates outright when it isn't confident, when its own tools can't get it evidence, or when the log itself looks tampered with.

**Live demo:** https://rishiudgirkar-a11y.github.io/pipeline-incident-resolver/
**Built as part of an Agentic AI capstone** (Discovery → Design → Develop → Deploy).

> All data in this repo — incidents, logs, error codes, SOPs — is synthetic. No real pipeline names, credentials, or company data appear anywhere.

## The problem

When a pipeline fails today, an engineer gets paged, opens the AWS console, reads the raw EMR log line by line, hunts SharePoint for the matching SOP, applies the fix by hand, and updates the ticket manually. That takes 1–2 hours per incident even though most failures fall into a handful of well-documented patterns — and two engineers reading the same log can walk away with different conclusions.

## How it works

One loop, five stages, visible on screen for every run:

1. **Input** — a ServiceNow-style incident (number, DAG name, pipeline, timestamp, severity), picked from the case queue or a one-click demo chip.
2. **Context** — the matching EMR log, the full error catalog (5 known codes + precedence rules), the SOPs, and the escalation policy — all named on screen before any decision is made.
3. **Decision** — a live call to the Anthropic Messages API (`claude-sonnet-4-5`), which classifies the failure, applies precedence when signals conflict, and runs 5 pre-recommendation checks before saying anything.
4. **Output** — a labeled review packet (error code, confidence, checks, matched SOP, recommended steps, citations back to the exact log line and SOP section used) — or a labeled escalation notice. Never a wall of text.
5. **Review** — a human clicks **Approve**, **Edit**, or **Escalate**. On approval, a simulated recovery check decides whether the ticket actually closes or escalates as a failed remediation.

The agent escalates instead of recommending whenever: confidence is below 0.95, the incident/log data is missing or unreadable, the incident is high severity, the fix needs more than a standard restart, the DAG/timestamp don't correlate, remediation was tried and failed, or — checked first, on every run — the log itself contains text that reads like an instruction aimed at the agent rather than a normal log line. See [`policies/escalation_policy.md`](policies/escalation_policy.md) for the exact trigger list and reason strings.

## Try it

Open the [live demo](https://rishiudgirkar-a11y.github.io/pipeline-incident-resolver/), paste your own Anthropic API key into the Settings panel (stored only in your browser's `localStorage` — never written to disk or sent anywhere but the Anthropic API), then click one of the demo chips:

| Chip | What it shows |
|---|---|
| Happy path | Clean OutOfMemoryError, high confidence, approved and resolved |
| Missing/unreadable log | Escalates rather than guessing |
| Low confidence | Escalates when no signature clears the confidence floor |
| **Boundary** | SEV-1 incident whose log the agent's own tool can't fetch — refuses and escalates on screen, in red |
| Cascading failure | Two co-occurring error signatures; agent applies catalog precedence to the root cause, not the obvious symptom |

Or run it locally — it's a single static file:

```bash
git clone https://github.com/rishiudgirkar-a11y/pipeline-incident-resolver.git
cd pipeline-incident-resolver
open index.html   # or just double-click it
```

No build step, no server, no dependencies. Everything runs client-side; the only network call is to the Anthropic API.

## Repo layout

```
index.html                          the entire product — one file
data/
  incidents.json                    15 synthetic ServiceNow-style incidents
  logs.json                         matching EMR/Airflow log snippets
  eval_cases.csv                    the 5 eval cases run against the live agent
policies/
  error_catalog.json                5 known error codes + precedence rules
  escalation_policy.md              all 7 escalation triggers + pre/post checks
  sops/                             one runbook per error code
templates/
  servicenow_update_template.json   shape of the closed-ticket payload
examples/
  sample_review_packet.md           target shape of a decision output
design/
  TOKENS.css, SKINS.md              locked design tokens for the console UI
DEVELOP_PRD_ANSWERS.md              build phase: scope, evals, improvement made, limitations
DEPLOY_PRD_ANSWERS.md               launch phase: go/no-go, risks, pilot plan, monitoring
```

## Evals

5 cases run for real against the live model — 4 pass, 1 recorded honestly as "needs work" rather than smoothed over:

| Case | Expected | Result |
|---|---|---|
| Happy path | OOM @ ≥0.95, all checks pass | **Pass** |
| Missing/unreadable log | Escalate, no guess | **Pass** |
| Low-confidence match | Escalate on confidence floor | **Needs work** — escalates correctly but via a stricter path than the catalog's intended generic-match reasoning; safety held, reasoning diverged. Documented, not silently fixed. |
| Boundary (tooling-blind + SEV-1) | Escalate, name both reasons, never close the ticket | **Pass** |
| Cascading failure | Match root cause via precedence | **Pass** — held even after the log was reworded to remove the literal catalog phrase |

Full detail, including the adversarial prompt-injection test that was found and fixed mid-build, in [`DEVELOP_PRD_ANSWERS.md`](DEVELOP_PRD_ANSWERS.md).

## Known limitations

- Only the 5 error patterns in `error_catalog.json` are handled — anything else needs a new catalog entry and SOP, not a config change.
- Matching leans on literal phrase presence in the log text, not fuzzy/semantic matching.
- `confirm_recovery()` and `update_servicenow()` are simulated from static fields, not a live MWAA/EMR poll or a real ServiceNow call.
- Processes one incident at a time; "Run All" is a demo convenience, not a production dispatcher.

## Pilot plan (if this went further)

3-person on-call rotation, limited to the 5 known error types, 2 weeks, with a named decision owner who can pause it instantly. Success: ≥95% of recommendations approved as-is, zero missed escalations, median time-to-resolution under 15 minutes. A formal privacy review is required before any real incident/log data — as opposed to the synthetic data used here — ever reaches the agent. Full detail in [`DEPLOY_PRD_ANSWERS.md`](DEPLOY_PRD_ANSWERS.md).
