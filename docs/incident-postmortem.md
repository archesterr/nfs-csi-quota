# Incident postmortem: enforce mode crash-looped a log store

Generalized from a real rollout. Included because it's the single most instructive failure mode of this tool, and worth reading before you flip `enforce` on for the first time.

## What happened

A log-storage workload (single-binary Loki) had years of accumulated data on a volume with a 10Gi declared PVC size — actual usage was ~48Gi. `enforce` mode was rolled out using the same config that had worked cleanly on several other clusters, without first checking `repquota` against each PVC's real declared size on *this* cluster specifically. The moment the sync timer applied the new 10Gi hard limit, the workload's next write — a startup index/WAL operation, not even new log ingestion — failed with `EDQUOT` and the pod crash-looped.

## Compounding failure

Because the failing write was mid-flush (not a clean rejection before the write started), it left a corrupted WAL segment on disk. Simply lifting the quota limit afterward didn't fix the crash — the application still refused to start, now failing WAL replay with a checksum error on the corrupted segment. This required a second, unrelated fix: quarantining the damaged WAL segment directory directly on the NFS server (moving it aside, not deleting — the actual retained data lived elsewhere and was untouched).

## Root cause behind the root cause

Investigating why usage had grown so far past the declared size found that the workload's own retention/compaction mechanism had a `last_successful_run_timestamp` metric stuck at `0` — meaning its own space-reclaiming logic had **never once completed a cycle**, despite the config flag for it being enabled. The declared 10Gi PVC size had been aspirational the entire time, not enforced by anything, and nothing had reclaimed old data to bring usage back toward it.

## Recovery sequence

1. Emergency mitigation: local `nfs_csi_quota_mode: alert` override on this one environment — unblocked writes within one sync tick.
2. Fixed the corrupted WAL segment (quarantine, not delete).
3. Resized the PVC via `kubectl patch` (not via the Helm chart's values — see the note in the main README about `volumeClaimTemplate` immutability) to a size with real headroom above current usage, not just above the old declared size.
4. Left the underlying retention bug open as a separate, non-blocking follow-up — the resize buys time, it doesn't fix the reason usage grew unbounded in the first place.
5. Removed the `alert` override, confirmed the new hard limit sat comfortably above real usage via `repquota`, confirmed the pod stayed healthy through the transition.

## Lessons generalized into this project

- `docs/enforce-mode-checklist.md` exists because of this incident.
- The role's README explicitly warns that `enforce` doesn't care whether an overage predates the rollout.
- The local per-environment `alert` override pattern (rather than only a global mode) exists specifically so one environment's incident never has to touch a shared default that other clusters depend on.
