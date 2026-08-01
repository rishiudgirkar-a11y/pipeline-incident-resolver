# Sample Review Packet (target shape)

**Incident Number:** INC0001234
**DAG / Pipeline:** daily_sales_ingest — Daily Sales Ingest (EMR)
**Failure Time:** 2026-07-30T02:14:00Z
**Detected Error Code:** EMR_OOM_002 — EMR executor OutOfMemoryError
**Confidence:** 0.96

**Log Evidence:**
> `java.lang.OutOfMemoryError: Java heap space` — Container killed by YARN for exceeding memory limits, stage 4, task 12.

**Check Results:**
- DAG name matches log: PASS
- Log corresponds to failure time: PASS
- Error code identified: PASS
- Matching SOP exists: PASS
- Confidence >= 0.95: PASS

**Matched SOP:** SOP_EMR_OOM_002 — EMR Executor OutOfMemoryError

**Recommended Steps:**
1. Confirm this is a standalone memory issue, not an upstream-timeout symptom.
2. Increase executor memory one tier for this run.
3. Restart the failed EMR step.
4. Check for data skew if it fails again.
5. Confirm downstream row counts once complete.

**Proposed Ticket Update (pending approval):** "Detected EMR_OOM_002 in the EMR log for daily_sales_ingest, run 2026-07-30T02:14:00Z. Applied SOP_EMR_OOM_002: increased executor memory and restarted the step. Recovery confirmed."

**Action Required:** Approve / Edit / Reject-Escalate

*(after approval)*
**Steps Executed:** ✔
**Job Recovered:** ✔ (rerun succeeded)
