# nfs-csi-quota

Per-PVC capacity enforcement for [`csi-driver-nfs`](https://github.com/kubernetes-csi/csi-driver-nfs), implemented with ext4 project quotas on the NFS server.

## The problem

`csi-driver-nfs` provisions each PVC as a plain subdirectory on a shared export. `spec.capacity` is used only for PV/PVC matching — nothing enforces it. NFS itself has no client-side quota concept. Consequences:

- A single runaway workload can fill the export and take down every other consumer of it.
- `df` inside a pod reports the whole share, not the PVC's declared size.
- `kubelet_volume_stats_*` is meaningless for this driver.
- Kubernetes' own `ResourceQuota` caps what gets *provisioned*, never what gets *written*.

Enforcement has to live where the filesystem lives: on the server.

## How it works

```
Kubernetes API                                    NFS server
──────────────────────                             ─────────────────────────────────
PersistentVolumes ── (read-only SA) ──▶  nfs-csi-quota-sync.sh   (systemd timer)
                                              │
                                              ├─ chattr -p <projid> +P  /export/pvc-*   (one ext4 project per PV)
                                              ├─ setquota -P <projid>                    (alert: 0 · enforce: PVC size)
                                              └─ nfs_csi_quota.prom ──▶ node_exporter (textfile)
                                                                              │
                                                                     your Prometheus/VictoriaMetrics
```

Each PV name maps to a stable ext4 project ID (first 7 hex chars of `md5(pv-name)`). The inherit flag (`+P`) is set on directories; anything written later inherits it automatically, so the recursive walk only happens once per PV. Accounting is done by the kernel — the sync loop just reconciles project membership and limits against the current PV list.

## Modes

| `nfs_csi_quota_mode` | Behavior |
|---|---|
| `alert` (default) | Accounting only — limits stay at 0, nothing is ever blocked. Capacity becomes a *reported* contract; wire the alert rules in `manifests/alerts-example.yaml` to page on breach. |
| `enforce` | Hard limit = PVC's declared size. Writes past capacity fail with `EDQUOT`; `df` in pods and `kubelet_volume_stats_*` become per-PVC accurate. |

**Read this before switching to `enforce`:** [`docs/enforce-mode-checklist.md`](docs/enforce-mode-checklist.md). Flipping straight to enforce on a volume that has already grown past its declared PVC size will crash-loop that workload the instant the hard limit lands — this is not hypothetical, see [`docs/incident-postmortem.md`](docs/incident-postmortem.md) for a real one.

## Install

### 1. Cluster side — RBAC

This role never touches the Kubernetes API to *create* objects, only to *read* one Secret. Apply the RBAC yourself, with whatever tooling manages your cluster (`kubectl apply`, ArgoCD, Flux, Helm — your choice):

```bash
kubectl apply -f manifests/rbac/nfs-csi-quota-reader.yaml
```

This creates a `ServiceAccount` + `ClusterRole` (get/list on `persistentvolumes` only, nothing else) + `ClusterRoleBinding` + a long-lived token `Secret`.

### 2. NFS server side — Ansible role

```yaml
# playbook.yml
- hosts: nfs_servers
  roles:
    - role: nfs_csi_quota
      vars:
        nfs_csi_quota_enabled: true
        nfs_csi_quota_rbac_read_enabled: true
        nfs_csi_quota_api_server: "https://your-api-endpoint:6443"
        nfs_csi_quota_kube_version: "1.31.4"   # match your cluster's minor
```

```bash
ansible-playbook playbook.yml -l nfs_servers --tags nfs_csi_quota
```

First run on a filesystem without the ext4 `project,quota` feature needs an explicit opt-in — this is real downtime (stop NFS → umount → `e2fsck` → `tune2fs` → remount):

```bash
ansible-playbook playbook.yml -l nfs_servers --tags nfs_csi_quota \
  -e nfs_csi_quota_allow_offline_enable=true
```

### 3. Metrics (optional but recommended)

The sync script writes `/var/lib/node_exporter/textfile/nfs_csi_quota.prom` on every tick. Point `node_exporter` at it:

```
--collector.textfile.directory=/var/lib/node_exporter/textfile
```

And scrape the NFS server as a static target — it's outside the cluster, nothing discovers it automatically. See `manifests/alerts-example.yaml` for ready-to-adapt alert rules.

## Key variables

| Variable | Default | Notes |
|---|---|---|
| `nfs_csi_quota_enabled` | `false` | Master switch |
| `nfs_csi_quota_mode` | `alert` | `alert` \| `enforce` |
| `nfs_csi_quota_sync_interval` | `5min` | Worst-case propagation delay for a resized/new PVC |
| `nfs_csi_quota_allow_offline_enable` | `false` | Guards the downtime path — enable only for the initial conversion |
| `nfs_csi_quota_kubeconfig` | `/root/.kube/nfs-csi-quota-reader.conf` | Rendered by the role, disposable — safe to delete, it rebuilds from the cluster Secret |
| `nfs_csi_quota_api_tls_server_name` | `""` | Set only if you hit an x509 SAN mismatch — the error names the valid SANs |
| `nfs_csi_quota_kube_version` | `1.31.4` | Must match your cluster's minor (client skew ±1) |

## Metrics

- `nfs_csi_quota_info{projid,pv,namespace,pvc}` — join metric
- `nfs_csi_quota_limit_bytes{projid}` — PVC declared size
- `nfs_csi_quota_used_bytes{projid}` — kernel accounting
- `nfs_csi_quota_kubeapi_up` — 1/0, whether the last sync tick could reach the API server

## Day-2 operations

**PVC resized larger** — this driver's "resize" is metadata-only (no real block device to grow), so `kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"<new size>"}}}}'` is all that's needed (needs `allowVolumeExpansion: true` on the StorageClass). The new limit lands on the next sync tick, no pod restart. **Editing a Helm chart's `size:` value and re-syncing does not resize a live PVC if that value feeds a StatefulSet's `volumeClaimTemplate`** — that field is immutable at the Kubernetes API level; only the direct `kubectl patch` above works on the live object. Update the chart value too, just don't expect it to reconcile — it exists only so a future recreate doesn't silently revert.

**Emergency override, single project, no Ansible round-trip:**
```bash
setquota -P <projid> 0 0 0 0 /export/path
```
Temporary — the next sync tick re-applies whatever `nfs_csi_quota_mode` says in inventory. For anything longer than "buy a few minutes," change the inventory var instead.

**New/deleted PVCs** — new volumes converge within one tick. Reclaim policy is typically `Retain`; deleted claims keep their directory and quota until you clean up manually.

**Full rebuild** — supported: from a bare NFS server, one Ansible run with `nfs_csi_quota_allow_offline_enable=true` recreates everything.

## Caveats

- Project IDs are hash-derived; collision risk is theoretical at reasonable scale but non-zero.
- Directory layout must be `<export>/<pv-name>` (this driver's default). Adjust the sync script if you use a custom `subDir` pattern.
- ext4 rejects the inherit flag on regular files (`Operation not supported`) — expected, handled by tagging directories and files separately.
- Filesystem must be ext4. XFS project quotas work differently (`xfs_quota`, not `chattr`/`setquota`) and aren't implemented here — contributions welcome.
- This role only reads Kubernetes RBAC objects, never creates them — by design, to keep the credential footprint on the NFS server minimal (read-only PV listing, nothing else).

## License

MIT — see [LICENSE](LICENSE).
