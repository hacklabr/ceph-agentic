# Known Issues — Tentacle (20.2.x)

This document tracks confirmed issues and notable fixes for Ceph Tentacle (20.2.x). Issues are ordered newest-first.

---

## Tentacle 20.2.2

### EC optimizations require stripe_unit of at least 16K

| Field | Value |
|-------|-------|
| **Affected versions** | 20.2.0 – 20.2.x |
| **Component** | RADOS / Erasure Coding |
| **Severity** | Medium |
| **Status** | Behavior change / design requirement |
| **Tracker** | N/A |
| **PR** | N/A |

**Symptom**
Enabling EC optimizations (`allow_ec_optimizations`) on an erasure-coded pool without a sufficiently large stripe unit results in suboptimal small-I/O performance or configuration errors.

**Cause**
Optimized EC reduces padding and space amplification, but it requires the erasure-code profile to specify a `stripe_unit` of at least 16K at creation time.

**Workaround**
Create the profile before the pool with `stripe_unit=16384`:

```bash
ceph osd erasure-code-profile set my-optimized-profile \
    k=4 \
    m=2 \
    crush-failure-domain=host \
    plugin=isa \
    technique=reed_sol_van \
    stripe_unit=16384
```

**Fix**
Documented design requirement; no code fix needed.

---

### Fast EC rejects non-4K-aligned chunk sizes

| Field | Value |
|-------|-------|
| **Affected versions** | 20.2.0 |
| **Component** | Monitor / Erasure Coding |
| **Severity** | Medium |
| **Status** | Fixed in 20.2.1 |
| **Tracker** | N/A |
| **PR** | N/A |

**Symptom**
Attempts to enable "fast EC" optimizations on an erasure-coded profile with chunk sizes not aligned to 4K are rejected by the monitor.

**Cause**
Fast EC performance and correctness depend on 4K-aligned chunk sizes. Unaligned configurations are now explicitly denied.

**Workaround**
Use 4K-aligned chunk sizes in the erasure-code profile.

**Fix**
Fixed in 20.2.1.

---

### MDS segmentation fault during request retry queueing

| Field | Value |
|-------|-------|
| **Affected versions** | 20.2.0 – 20.2.1 |
| **Component** | CephFS / MDS |
| **Severity** | High |
| **Status** | Fixed in 20.2.2 |
| **Tracker** | https://tracker.ceph.com/issues/76031 |
| **PR** | https://github.com/ceph/ceph/pull/68905 |

**Symptom**
MDS daemon crashes with a segmentation fault during request retry handling.

**Cause**
Incorrect queueing of request retries caused use-after-free-like conditions in the MDS request path.

**Workaround**
Monitor MDS pods and restart any that crash. Schedule upgrade to 20.2.2.

**Fix**
Fixed in Tentacle 20.2.2.

---

### OSD PGLog missing list version mismatch

| Field | Value |
|-------|-------|
| **Affected versions** | 20.2.0 – 20.2.1 |
| **Component** | OSD / BlueStore |
| **Severity** | High |
| **Status** | Fixed in 20.2.2 |
| **Tracker** | N/A |
| **PR** | https://github.com/ceph/ceph/pull/68718 |

**Symptom**
Recovery errors or assertion failures in optimized EC configurations related to missing list entries.

**Cause**
Incorrect version was attached to the missing list when ignoring log entries during peering/recovery.

**Workaround**
Upgrade to 20.2.2.

**Fix**
Fixed in Tentacle 20.2.2.

---

### RGW lifecycle transition fails for encrypted multipart objects

| Field | Value |
|-------|-------|
| **Affected versions** | 20.2.0 – 20.2.1 |
| **Component** | RGW / Lifecycle |
| **Severity** | Medium |
| **Status** | Fixed in 20.2.2 |
| **Tracker** | N/A |
| **PR** | https://github.com/ceph/ceph/pull/68826 |

**Symptom**
Lifecycle transitions that move encrypted multipart objects between storage classes fail or leave objects in an inconsistent state.

**Cause**
Lifecycle transition logic did not correctly handle encrypted multipart uploads.

**Workaround**
Upgrade to 20.2.2 or avoid lifecycle transitions on encrypted multipart objects.

**Fix**
Fixed in Tentacle 20.2.2.

---

### Ceph-volume inventory scans RAM disk devices

| Field | Value |
|-------|-------|
| **Affected versions** | 20.2.0 – 20.2.1 |
| **Component** | ceph-volume |
| **Severity** | Low |
| **Status** | Fixed in 20.2.2 |
| **Tracker** | N/A |
| **PR** | https://github.com/ceph/ceph/pull/68552 |

**Symptom**
`ceph-volume inventory` or `ceph orch device ls` incorrectly lists RAM disk devices (`/dev/ram*`) as candidates for OSD deployment.

**Cause**
Inventory discovery did not skip RAM disk devices.

**Workaround**
Ignore `/dev/ram*` entries in device lists or manually filter them.

**Fix**
Fixed in Tentacle 20.2.2.

---

## Tentacle 20.2.1

### Rocky 10 package-based installs now supported

| Field | Value |
|-------|-------|
| **Affected versions** | 20.2.0 |
| **Component** | Packaging / cephadm |
| **Severity** | Low |
| **Status** | Fixed in 20.2.1 |
| **Tracker** | N/A |
| **PR** | https://github.com/ceph/ceph/pull/65719 |

**Symptom**
cephadm bootstrap or package installation fails on Rocky Linux 10 with the 20.2.0 image.

**Cause**
20.2.0 default image did not include Rocky 10 support.

**Workaround**
Upgrade to 20.2.1 or later.

**Fix**
Rocky 10 support added in 20.2.1; default image updated.

---

### OSD/BlueStore EC recovery assertion failures

| Field | Value |
|-------|-------|
| **Affected versions** | 20.2.0 |
| **Component** | OSD / BlueStore / EC |
| **Severity** | High |
| **Status** | Fixed in 20.2.1 |
| **Tracker** | N/A |
| **PR** | N/A |

**Symptom**
OSD crashes with assertion failures such as `shard_size >= tobj_size` during EC recovery of small objects.

**Cause**
Length calculation bug in `erase_after_ro_offset()` caused empty shards to retain data.

**Workaround**
Upgrade to 20.2.1.

**Fix**
Fixed in Tentacle 20.2.1.

---

## General Notes

- Tentacle introduces `allow_ec_optimizations`. Once enabled on a pool, it cannot be disabled.
- The `bluefs_check_volume_selector_on_umount` debug option was renamed to `bluefs_check_volume_selector_on_mount` and now runs checks on both mount and unmount.
- Default monitoring stack images were updated in 20.2.1 (Prometheus v3.6.0, Alertmanager v0.28.1, Grafana v12.2.0, Node-exporter v1.9.1).

---

*Sources: official Ceph Tentacle release notes at https://docs.ceph.com/en/latest/releases/tentacle/ and https://github.com/ceph/ceph/blob/main/doc/releases/tentacle.rst*
