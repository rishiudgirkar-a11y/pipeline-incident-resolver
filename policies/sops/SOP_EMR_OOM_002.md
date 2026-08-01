# SOP_EMR_OOM_002 — EMR Executor OutOfMemoryError

**Error name:** OutOfMemoryError (Java heap space / GC overhead limit exceeded)

**Description:** An EMR executor exceeded its allotted memory while processing a stage, usually because a partition is larger than expected, skew has concentrated data on one executor, or executor memory is undersized for the current data volume.

**Resolution steps:**
1. Confirm this is a standalone memory issue and not a symptom of an upstream timeout (check error_catalog.json precedence rules first).
2. Increase the EMR step's executor memory configuration by one tier (e.g. `spark.executor.memory` 4g → 6g) for this run only.
3. Restart the failed EMR step from the EMR console (Steps tab → Resubmit).
4. If the step fails again at the higher memory tier, check for data skew in the source partition before resubmitting a second time.
5. Once the step completes, confirm downstream row counts match the expected range for the run date.
