# Before you switch to `enforce`

`enforce` sets a real ext4 hard limit at the kernel level. It does not care whether the overage predates the quota rollout — the instant the limit is lower than current usage, the *next write* to that project fails with `EDQUOT`. For a database-like workload (Loki, Prometheus/VictoriaMetrics storage engines, etc.) that write can be a WAL flush or index update, and a failed write mid-operation can leave the application unable to start at all, not just unable to grow.

## Checklist

1. **Run in `alert` mode first**, for at least one full cycle of your largest/most active workload's retention or compaction window. This surfaces real usage without any risk.

2. **Check every PVC against its declared size before switching:**
   ```bash
   repquota -P -s /export/path
   ```
   Compare `used` against each project's real PVC size (`kubectl get pv -o json | jq ...` or cross-reference `nfs_csi_quota_info`/`nfs_csi_quota_limit_bytes`). Any project where `used` is already close to or over its PVC's declared size needs to be resized (via `kubectl patch pvc`, see the main README) **before** enforce is turned on for that volume — not after.

3. **Understand your workload's retention/compaction story.** If a workload is supposed to self-limit its own disk usage (log retention, TSDB compaction, etc.) but its actual usage doesn't match what retention config implies, don't trust the declared retention period — verify directly:
   - Loki: check `loki_compactor_apply_retention_last_successful_run_timestamp_seconds` — if it's `0` or stale, retention has never actually completed a cycle, regardless of what `retention_enabled: true` in config implies.
   - Any TSDB-based system: confirm compaction/retention metrics are non-zero and recent, not just that the config flag is set.

4. **Roll out per-environment, not globally, if you have more than one cluster.** A shared default (e.g. `enforce` as the role default with per-environment overrides) is fine architecturally, but each environment's actual usage-vs-declared-size state is independent — clearing this checklist on one cluster says nothing about another.

5. **If something does crash-loop after enabling enforce:** the fastest safe rollback is a local override, not touching the shared default:
   ```yaml
   # this environment's inventory only
   nfs_csi_quota_mode: alert
   ```
   ```bash
   ansible-playbook playbook.yml -l <this-host> --tags nfs-csi-quota-sync
   ```
   That flips the hard limit back to 0 on the next tick and unblocks writes immediately, with zero effect on any other environment sharing this role.

See [`incident-postmortem.md`](incident-postmortem.md) for a real example of what happens when this checklist is skipped, and how it was recovered.
