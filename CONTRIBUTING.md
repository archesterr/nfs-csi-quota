# Contributing

Issues and PRs welcome. A few areas that would particularly help:

- **XFS support.** Currently ext4-only (`chattr`/`setquota`). XFS project quotas use `xfs_quota` and a different mechanism for tagging directories — a parallel implementation (not a rewrite) would be a good addition.
- **Testing.** There's no CI/molecule setup yet. A test harness that spins up a loopback ext4 filesystem and exercises alert/enforce transitions would be valuable.
- **Custom `subDir` layouts.** The sync script assumes `<export>/<pv-name>`, matching `csi-driver-nfs`'s default. If you use a custom `subDir` template, a PR generalizing the path logic (with a new variable) would help others.
- **Alternative CSI drivers.** The core mechanism (ext4 project quotas keyed off PV name) isn't inherently tied to `csi-driver-nfs` specifically — if you use a different NFS-backed CSI driver with a similar "PV = subdirectory" model, adapting `nfs_csi_quota_csi_driver` and confirming it works is useful data.

Please include a real-world data point (cluster size, PVC count, ext4 kernel version) in any PR touching the sync script — the project ID hashing and `chattr` inheritance logic have edge cases worth stress-testing at scale.
