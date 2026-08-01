# SOP_S3_ACCESS_DENIED_005 — S3 Access Denied

**Error name:** S3 Access Denied (403 Forbidden / AccessDenied)

**Description:** The EMR job's own IAM role could not read or write the S3 object it needed. This is the pipeline's own read/write failing (visible inside the log text) — distinct from the agent's own log-fetch tooling being denied access.

**Resolution steps:**
1. Confirm this is the pipeline's EMR role failing (found in the log text), not the agent's own tooling access — the two look similar but are handled differently.
2. Do not attempt to modify IAM policy or bucket policy directly — this is an out-of-scope SOP action requiring the platform/security team.
3. Present the access-denied finding as information only; recommend restart is not viable until access is restored.
4. Escalate to the platform team with the exact bucket, key prefix, and role ARN from the log.
5. Once access is confirmed restored (by the platform team), rerun the failed EMR step.
