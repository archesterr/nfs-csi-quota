# nfs-csi-quota

Per-PVC capacity enforcement for [`csi-driver-nfs`](https://github.com/kubernetes-csi/csi-driver-nfs), using ext4 project quotas on the NFS server. `csi-driver-nfs` doesn't enforce `spec.capacity` itself — this does, at the filesystem level.

## Quick start

**1. Apply RBAC to your cluster** (read-only: `get,list` on `PersistentVolumes`, nothing else):
```bash
kubectl apply -f manifests/rbac/nfs-csi-quota-reader.yaml
```

**2. Run the role on your NFS server:**
```yaml
# playbook.yml
- hosts: nfs_servers
  roles:
    - role: nfs_csi_quota
      vars:
        nfs_csi_quota_enabled: true
        nfs_csi_quota_api_server: "https://your-api-endpoint:6443"
        nfs_csi_quota_kube_version: "1.31.4"   # match your cluster's minor
```
```bash
ansible-playbook playbook.yml -l nfs_servers --tags nfs_csi_quota
```

First run on a filesystem without ext4 project quota needs one extra flag — **this is real downtime** (stop NFS → umount → fsck → remount):
```bash
ansible-playbook playbook.yml -l nfs_servers --tags nfs_csi_quota -e nfs_csi_quota_allow_offline_enable=true
```

**3. Check it worked:**
```bash
repquota -P -s /mnt/nfs
```
You should see one project per PV.

That's it — it starts in `alert` mode (reporting only, nothing blocked). **Before you switch to `enforce`, read [`docs/enforce-mode-checklist.md`](docs/enforce-mode-checklist.md) — it takes two minutes and will save you an incident.**

## Metrics (optional)

Every sync tick writes Prometheus-format metrics to `/var/lib/node_exporter/textfile/nfs_csi_quota.prom`:

- `nfs_csi_quota_used_bytes{projid}` / `nfs_csi_quota_limit_bytes{projid}` — usage vs. declared size
- `nfs_csi_quota_info{projid,pv,namespace,pvc}` — join label
- `nfs_csi_quota_kubeapi_up` — is the sync script reaching the API server

Point node_exporter at that directory (`--collector.textfile.directory=...`), scrape the NFS server as a static target (it's outside the cluster), done. Example alert rules: [`manifests/alerts-example.yaml`](manifests/alerts-example.yaml) — including a pattern for alerting on the NFS server's own exporter reachability without false-positiving on scrape blips (a real risk for any statically-scraped, out-of-cluster target).

## Key variables

| Variable | Default | Notes |
|---|---|---|
| `nfs_csi_quota_mode` | `alert` | `alert` (report only) or `enforce` (hard limit = PVC size) |
| `nfs_csi_quota_sync_interval` | `5min` | How fast a resized/new PVC takes effect |
| `nfs_csi_quota_allow_offline_enable` | `false` | One-time, downtime-required fs conversion |
| `nfs_csi_quota_api_tls_server_name` | `""` | Only needed if you hit a TLS SAN mismatch — the error tells you the right value |

Full variable list: [`roles/nfs_csi_quota/defaults/main.yml`](roles/nfs_csi_quota/defaults/main.yml) (every one is commented).

## Day-2 basics

- **Resize a PVC:** `kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"<size>"}}}}'` (needs `allowVolumeExpansion: true`). Takes effect on the next sync tick, no pod restart. If the PVC belongs to a StatefulSet, editing the Helm chart's size value alone does **not** resize the live volume — `volumeClaimTemplates` is immutable; the `kubectl patch` above is what actually works.
- **Emergency unblock one PVC** without a full role run: `setquota -P <projid> 0 0 0 0 /mnt/nfs` — temporary, the next sync tick re-applies whatever mode is set.
- **Roll back one environment to alert mode** without touching a shared default: set `nfs_csi_quota_mode: alert` in that environment's inventory only, then `--tags nfs-csi-quota-sync`.

## Why this exists / how it works

<details>
<summary>Expand if you want the background</summary>

`csi-driver-nfs` provisions each PVC as a plain subdirectory on a shared export. `spec.capacity` is used only for PV/PVC matching — nothing enforces it, because NFS has no client-side quota concept. A single runaway workload can fill the export and take down every other consumer; `df` in a pod reports the whole share, not the PVC.

This project maps each PV to a stable ext4 project ID (`md5(pv-name)`, first 7 hex chars), tags its directory with the inherit flag so anything written later is covered automatically, and runs `setquota` on a systemd timer. Accounting is done by the kernel; the sync loop just reconciles project membership and limits against the current PV list.

```
Kubernetes API                                    NFS server
──────────────────────                             ─────────────────────────────
PersistentVolumes ── (read-only SA) ──▶  nfs-csi-quota-sync.sh   (systemd timer)
                                              ├─ chattr -p <projid> +P  /export/pvc-*
                                              ├─ setquota -P <projid>   (alert: 0 · enforce: PVC size)
                                              └─ *.prom ──▶ node_exporter (textfile)
```

RBAC lives in `manifests/rbac/`, applied separately from the role by design — the role only ever *reads* the resulting Secret, never creates cluster objects itself.
</details>

## Caveats

- ext4 only (`chattr`/`setquota`). XFS uses a different mechanism — not implemented, PRs welcome.
- Directory layout must be `<export>/<pv-name>` (this driver's default).
- Project IDs are hash-derived; collision risk is theoretical at reasonable scale, non-zero.

## License

MIT — see [LICENSE](LICENSE).
