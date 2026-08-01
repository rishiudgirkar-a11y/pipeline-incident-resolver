# SOP_UPSTREAM_TIMEOUT_003 — Upstream Dependency Timeout

**Error name:** Upstream dependency timeout (ExternalTaskSensor / sensor timed out)

**Description:** The DAG's sensor waited for an upstream DAG or feed to complete and it did not finish within the configured window. This is a root-cause pattern: if the job proceeded anyway, later-looking failures (e.g. OutOfMemoryError) in the same run are usually downstream symptoms of this timeout, not separate issues.

**Resolution steps:**
1. Check the upstream DAG's own run status for the matching execution date.
2. If the upstream DAG is still running or recently completed, rerun this DAG's sensor task rather than the whole DAG.
3. If the upstream DAG itself failed, this incident should reference the upstream DAG's own incident rather than being resolved independently — flag to the pipeline owner if no upstream incident exists yet.
4. Once the upstream dependency is confirmed complete, rerun the downstream DAG from the failed task forward (not from the start).
5. Confirm the full downstream run completes without the earlier symptom (e.g. memory failure) recurring.
