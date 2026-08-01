# SOP_DAG_TASK_FAIL_004 — Generic DAG Task Failure

**Error name:** DAG task failure (non-zero exit code, no specific signature)

**Description:** A task failed with a non-zero exit status and no more specific error signature was found in the available log output.

**Resolution steps:**
1. Confirm no more specific error_catalog signature is present before applying this generic SOP — this is the lowest-precedence pattern.
2. Rerun the failed task from the DAG UI (Airflow: Clear task instance and let it retry).
3. If the rerun fails a second time with the same generic signature, do not rerun a third time — escalate for full human investigation rather than guessing further.
4. Once the task succeeds, confirm downstream tasks in the DAG complete normally.
