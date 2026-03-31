# Known Issues — Reef 18.2.x

Issues are ordered newest-first. See [README.md](./README.md) for the entry format.

---

### OSD: Spurious OSD_UPGRADE_FINISHED warning when upgrading from Pacific

| Field | Value |
|-------|-------|
| **Affected versions** | Pacific → Reef upgrade path (all Reef versions) |
| **Component** | RADOS / OSD |
| **Severity** | Medium (false positive warning; no data loss) |
| **Status** | Upgrade path from Pacific to Reef directly is no longer recommended as of v18.2.8 |
| **Tracker** | — |
| **PR** | — |

**Symptom**
During a rolling upgrade from Pacific to Reef, `ceph health` shows an `OSD_UPGRADE_FINISHED` warning before all OSDs have actually completed the upgrade. The warning can appear even while Pacific-version OSDs are still running.

**Cause**
Pacific OSDs continued using a deprecated connection feature bit that was re-purposed in Reef to indicate a fully-upgraded Reef OSD. When a Pacific OSD connects to the cluster it inadvertently "advertises" Reef compatibility, causing the monitor to believe the upgrade is complete ahead of schedule.

**Workaround**
1. Suppress the spurious warning and continue the upgrade; there are no known operational problems caused by the interoperability itself.
2. Monitor real upgrade progress with:
   ```bash
   ceph versions
   ceph orch ps --refresh
   ```
3. To avoid the issue entirely, upgrade from Pacific to Quincy first, then from Quincy to Reef:
   ```
   Pacific → Quincy → Reef
   ```

**Fix**
No code fix. As of v18.2.8 (March 2026) the direct Pacific → Reef upgrade path is officially unsupported. Always stage through Quincy.

---

### BlueStore: Critical _extend_log regression introduced in 18.2.5 and 18.2.6

| Field | Value |
|-------|-------|
| **Affected versions** | 18.2.5, 18.2.6 (fixed in 18.2.7) |
| **Component** | BlueStore |
| **Severity** | Critical |
| **Status** | Fixed in v18.2.7 |
| **Tracker** | — |
| **PR** | https://github.com/ceph/ceph/pull/61653 |

**Symptom**
OSDs may crash or become unresponsive. BlueStore internal log sequence numbers advance incorrectly, which can lead to data inconsistency or OSD assertion failures.

**Cause**
A regression in `os/bluestore` introduced in the BlueStore log extension path (`_extend_log`) caused the sequence number to advance incorrectly. This was accompanied by two additional race conditions in `BlueFS::truncate/remove` and `ExtentDecoderPartial::_consume_new_blob`.

**Workaround**
Do not remain on v18.2.5 or v18.2.6. Upgrade to v18.2.7 immediately.

If an upgrade is not immediately possible, set the cluster to read-only to prevent further damage:
```bash
ceph osd set noout
ceph osd set norecover
ceph osd set norebalance
```
Then schedule an emergency upgrade.

**Fix**
Fixed in v18.2.7 (May 8, 2025) via:
- https://github.com/ceph/ceph/pull/61653 — fix `_extend_log` seq advance
- https://github.com/ceph/ceph/pull/62840 — fix race in BlueFS truncate/remove
- https://github.com/ceph/ceph/pull/62054 — fix `ExtentDecoderPartial::_consume_new_blob`

---

### RADOS: pre-Reef clients can crash OSDs via pg-upmap-primary interface

| Field | Value |
|-------|-------|
| **Affected versions** | 18.2.0 – 18.2.3 (fixed in 18.2.4) |
| **Component** | RADOS / OSDMap |
| **Severity** | High |
| **Status** | Fixed in v18.2.4 |
| **Tracker** | https://tracker.ceph.com/issues/61948 |
| **PR** | — |

**Symptom**
OSDs and monitors assert/crash when pre-Reef clients connect to a cluster that has `pg-upmap-primary` entries in the OSD map, even when `require-min-compat-client=reef` is set. Health may show `FAILED ASSERT` lines in OSD logs.

**Cause**
Pre-Reef clients were allowed to connect to the `pg-upmap-primary` interface despite the `require-min-compat-client=reef` guard. The resulting incompatibility triggered an assertion in OSDs and monitors.

**Workaround**
Remove all existing `pg-upmap-primary` mappings before upgrading or connecting mixed-version clients:
```bash
# List existing mappings
ceph osd getmap -o /tmp/osdmap
osdmaptool /tmp/osdmap --dump json | jq '.pg_upmap_primaries'

# Remove all pg-upmap-primary mappings (see tracker note #32 for full procedure)
ceph osd rm-pg-upmap-primary <pgid>
```
Do not use `osdmaptool --read` to generate an OSD map with `pg-upmap-primary` entries while running mixed versions.

**Fix**
Fixed in v18.2.4 (July 24, 2024). Note: the fix is minimal; corner cases such as adding a mapping during an upgrade are addressed separately. Avoid using `pg-upmap-primary` until v18.2.4 or later.

---

### Container images: pthread_create crashes on kernels older than CentOS 9 baseline

| Field | Value |
|-------|-------|
| **Affected versions** | 18.2.4 container images |
| **Component** | Container runtime |
| **Severity** | High (container environments on old kernels only) |
| **Status** | Workaround available; no code fix planned — upgrade the host OS |
| **Tracker** | https://tracker.ceph.com/issues/66989 |
| **PR** | — |

**Symptom**
Ceph daemons in containers crash at startup with an error referencing `pthread_create`. Observed on hosts running Ubuntu 18.04 or other distributions with kernels older than the CentOS 9 baseline.

**Cause**
v18.2.4 container images are now based on CentOS 9, which uses a different thread creation method. The new method is incompatible with older host kernels.

**Workaround**
Option 1 — upgrade the host OS to a CentOS 9 / Ubuntu 20.04 equivalent or newer.

Option 2 — pin to v18.2.2 container images if an OS upgrade is not immediately possible, then upgrade the host OS before moving to v18.2.4+:
```bash
# Check current kernel version
uname -r

# If kernel < 5.x, plan OS upgrade before upgrading Ceph containers
```

**Fix**
No container image fix. Upgrade the host OS. See tracker https://tracker.ceph.com/issues/66989 for additional workaround notes.

---

### RGW: S3 multipart SSE objects corrupted after multi-site replication (18.2.0)

| Field | Value |
|-------|-------|
| **Affected versions** | 18.2.0 (fixed in 18.2.1) |
| **Component** | RGW / multi-site replication |
| **Severity** | High (data corruption on decryption) |
| **Status** | Fixed in v18.2.1 |
| **Tracker** | — |
| **PR** | — |

**Symptom**
In a multi-site RGW deployment, objects uploaded using S3 multipart upload with Server-Side Encryption (SSE) are corrupted when read back from a replica zone. Decryption fails or returns garbled data.

**Cause**
A replication defect in v18.2.0 meant that SSE multipart objects were not replicated correctly. The replica stored the object in an incompatible encrypted form.

**Workaround**
After upgrading all zones to v18.2.1 or later, identify and re-replicate the affected objects:
```bash
# Identify affected buckets (run on every zone after upgrading all zones)
radosgw-admin bucket resync encrypted multipart --bucket=<bucket-name>
```
This command increments the `LastModified` timestamp of affected objects by 1 ns, triggering peer zones to replicate them again. Run against every bucket in every zone.

**Fix**
Fixed in v18.2.1 (December 18, 2023). Mandatory remediation step required post-upgrade for any multi-site deployment that used SSE multipart uploads.

---

### CephFS: MDS goes read-only due to session metadata size exceeding RADOS op threshold

| Field | Value |
|-------|-------|
| **Affected versions** | 18.2.0 (mitigated in 18.2.1) |
| **Component** | CephFS / MDS |
| **Severity** | High (MDS becomes read-only) |
| **Status** | Mitigated in v18.2.1 |
| **Tracker** | — |
| **PR** | — |

**Symptom**
The MDS goes read-only. MDS logs show a RADOS operation failing because the encoded session metadata object exceeds the size threshold. Clients stall or are evicted.

**Cause**
Clients that are not advancing their request transaction IDs (tids) cause session metadata to grow unboundedly. When the encoded session metadata exceeds the RADOS operation size limit, the MDS cannot persist it and goes read-only.

**Workaround**
Tune the session metadata size limit to an appropriate value for your workload:
```bash
ceph config set mds mds_session_metadata_threshold 50000000
```
Identify and evict stalled clients:
```bash
ceph tell mds.* session ls
ceph tell mds.<id> session evict id=<client-id>
```

**Fix**
v18.2.1 added automatic eviction of clients not advancing tids and introduced the `mds_session_metadata_threshold` config knob. Upgrade to v18.2.1 or later.

---

*Source: official Ceph Reef release notes — https://github.com/ceph/ceph/blob/main/doc/releases/reef.rst*
*Last reviewed against: v18.2.8 (March 20, 2026)*
