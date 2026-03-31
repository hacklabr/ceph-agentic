# Deploy Services

After OSDs are running and the cluster reaches `HEALTH_OK`, you can deploy the three major client-facing services: RBD block storage, CephFS shared file system, and RGW object storage. Each service is independent — deploy only what your workloads require.

**Prerequisites:** `ceph status` must show all OSDs `up` and `in`, and all placement groups `active+clean`. Do not proceed with any service deployment while the cluster is recovering or has `HEALTH_ERR`.

---

## Service Comparison

| Service | Protocol | Use Case | Daemon(s) |
|---|---|---|---|
| **RBD** | Block (NBD / krbd) | VM disk images, database volumes, Kubernetes PVCs | None — RADOS access by clients |
| **CephFS** | POSIX file system | Shared home directories, HPC scratch, NFS re-export | MDS (Metadata Server) — one per filesystem, standby recommended |
| **RGW** | S3 / Swift HTTP | Object storage, backup targets, static assets | `radosgw` — one or more per zone |

---

## Choose the Right Service

### Use RBD when

- The workload needs a raw block device (VM hypervisor, database, Kubernetes PersistentVolume)
- Exclusive access from a single client is acceptable (unless using RBD mirroring for active/active)
- You want the lowest latency path to RADOS — no daemon sits between the client and OSDs

### Use CephFS when

- Multiple hosts or containers need simultaneous read/write access to the same directory tree
- The workload uses standard POSIX semantics (file locks, hard links, directory traversal)
- You need hierarchical quotas on directories

### Use RGW when

- Workloads speak S3 or Swift — backups, media storage, application data buckets
- You need HTTP/HTTPS access with presigned URLs, lifecycle policies, or bucket versioning
- You are building a private cloud object store to replace or supplement AWS S3

---

## Deployment Order

RBD does not require any gateway daemon and can be used immediately once pools exist. CephFS requires at least one active MDS before the filesystem can be mounted. RGW requires `radosgw` daemons before any S3 or Swift client can connect.

```
OSDs up + HEALTH_OK
        │
        ├── RBD ────────────────────────── create pool → rbd pool init → done
        │
        ├── CephFS ─── deploy MDS ──────── create fs → mount
        │
        └── RGW ────── deploy radosgw ──── realm/zone setup → create S3 user → test
```

No service conflicts with another — all three can be deployed in parallel, but always confirm OSDs are healthy first.

---

## Sub-Guides

| Guide | Content |
|---|---|
| [`rbd.md`](rbd.md) | Create RBD pool, images, map, mount, snapshots, kernel compatibility |
| [`cephfs.md`](cephfs.md) | Deploy MDS via cephadm, create filesystem, kernel and FUSE mounts, quotas |
| [`rgw.md`](rgw.md) | Deploy RGW via cephadm, realm/zone setup, S3 user creation, TLS, multisite overview |

---

## Quick Status Reference

After deploying any service, verify using these commands:

```bash
# Overall cluster health
ceph status

# MDS (CephFS) status
ceph mds stat

# RGW daemons
ceph orch ps --daemon-type rgw

# Running orchestrator services
ceph orch ls
```

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/rbd/rados-rbd-cmds.rst
  https://github.com/ceph/ceph/blob/main/doc/cephfs/createfs.rst
  https://github.com/ceph/ceph/blob/main/doc/cephadm/services/mds.rst
  https://github.com/ceph/ceph/blob/main/doc/cephadm/services/rgw.rst
-->
