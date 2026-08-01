# SOP_LOG_FILE_NOT_FOUND_001 — Input File / Path Not Found

**Error name:** FileNotFoundException (input path does not exist)

**Description:** The job's expected input path or partition was not present in S3 at run time, usually because the upstream write ran late, wrote to a different path, or did not run at all.

**Resolution steps:**
1. Check the upstream DAG/job's own run status for the same date/hour partition.
2. If the upstream write is simply late, wait for it to complete and rerun this DAG's failed task once the partition exists.
3. If the upstream path is wrong (naming or partitioning mismatch), do not attempt a silent path fix — flag to the pipeline owner as an out-of-scope SOP action.
4. Once the expected input partition is confirmed present, rerun the failed task from the DAG UI.
5. Confirm the task completes and the output partition is written before closing.
