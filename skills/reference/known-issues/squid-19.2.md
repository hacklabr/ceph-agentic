# Known Issues — Squid 19.2.x

Issues are ordered newest-first. See [README.md](./README.md) for the entry format.

---

### RGW: CopyObject to self erroneously marks tail objects for garbage collection

| Field | Value |
|-------|-------|
| **Affected versions** | 19.2.1 (fixed in 19.2.2) |
| **Component** | RGW |
| **Severity** | Critical (data loss) |
| **Status** | Fixed in v19.2.2 |
| **Tracker** | https://tracker.ceph.com/issues/66286 |
| **PR** | https://github.com/ceph/ceph/pull/62711 |

**Symptom**
Objects whose metadata is updated via a `CopyObject` call to itself (a standard S3 pattern for metadata-only changes) lose their data. Tail objects are erroneously queued for garbage collection and subsequently deleted. The object head remains but reads return empty or corrupted content.

**Cause**
A regression introduced by the fix for tracker #66286 caused tail objects of self-copies to be marked for GC instead of being preserved. This affected any S3 client that issues `PUT /bucket/key` with a `CopySource` equal to the same key.

**Workaround**
**Do not use Squid v19.2.1 in any RGW deployment.** Upgrade directly to v19.2.2 or later.

To identify objects that may have been damaged before upgrading, use the experimental gap-list tool:
```bash
rgw-gap-list --rgw-zone <zone-name> --bucket <bucket-name>
```

**Fix**
Fixed in v19.2.2 (April 10, 2025). Skip v19.2.1 entirely — the release notes carry an explicit `.. warning:: Do not upgrade to Squid v19.2.1`.

---

### RGW: STS authentication bypass via unsupported JWT algorithms (CVE-2024-48916)

| Field | Value |
|-------|-------|
| **Affected versions** | 19.2.0 – 19.2.2 (fixed in 19.2.3) |
| **Component** | RGW / STS |
| **Severity** | High (authentication bypass) |
| **Status** | Fixed in v19.2.3 |
| **Tracker** | — |
| **CVE** | CVE-2024-48916 |
| **PR** | https://github.com/ceph/ceph/pull/62137 |

**Symptom**
An attacker may be able to authenticate to RGW STS endpoints using JWT tokens signed with unsupported or weak algorithms. No visible cluster health warning is generated.

**Cause**
The STS endpoint did not reject JWT tokens that used unsupported algorithms. An attacker able to craft such a token could bypass OIDC/STS authentication.

**Workaround**
Until upgraded, restrict network access to the RGW STS endpoint to trusted networks only:
```bash
# Example: firewall the STS endpoint at the load balancer or host level
# The STS endpoint is typically on the same port as the RGW S3 API
iptables -I INPUT -p tcp --dport <rgw-port> -s <trusted-cidr> -j ACCEPT
iptables -A INPUT -p tcp --dport <rgw-port> -j DROP
```
Audit STS/AssumeRoleWithWebIdentity access logs for anomalous tokens.

**Fix**
Fixed in v19.2.3 (July 28, 2025) via PR #62137. Upgrade to v19.2.3 or later.

---

### RADOS / mClock: Inconsistent client throughput on HDD clusters with multiple OSD shards

| Field | Value |
|-------|-------|
| **Affected versions** | 19.2.0 (default changed in 19.2.1) |
| **Component** | RADOS / OSD / mClock scheduler |
| **Severity** | Medium (performance degradation; no data loss) |
| **Status** | Default configuration changed in v19.2.1 |
| **Tracker** | https://tracker.ceph.com/issues/66289 |
| **PR** | — |

**Symptom**
On HDD-based clusters, client throughput is inconsistent across test runs. During OSD node failures, recovery throughput varies significantly and slow requests are reported more often than expected. The behaviour is worse when multiple OSD shards are configured.

**Cause**
The mClock scheduler does not perform optimally when multiple OSD shards are used on HDD-backed OSDs. Each shard maintains its own mClock queue, leading to uneven I/O distribution and inconsistent QoS enforcement.

**Workaround**
Apply the new recommended defaults manually on 19.2.0 clusters:
```bash
ceph config set osd osd_op_num_shards_hdd 1
ceph config set osd osd_op_num_threads_per_shard_hdd 5
```
Restart OSD daemons to apply:
```bash
ceph orch restart osd
```

**Fix**
Default values changed in v19.2.1:
- `osd_op_num_shards_hdd` = 1 (was 5)
- `osd_op_num_threads_per_shard_hdd` = 5 (was 1)

Upgrade to v19.2.1 or later; the new defaults apply automatically for fresh deployments. Existing clusters with explicit config overrides must update them manually.

---

### MGR: Balancer module crash on upgrade to 19.2.0

| Field | Value |
|-------|-------|
| **Affected versions** | 19.2.0 (fixed in 19.2.1) |
| **Component** | MGR / balancer module |
| **Severity** | High (MGR crash; cluster continues operating but without active balancing) |
| **Status** | Fixed in v19.2.1 |
| **Tracker** | https://tracker.ceph.com/issues/68657 |
| **PR** | — |

**Symptom**
After upgrading to 19.2.0, the active Ceph Manager crashes repeatedly. The balancer module produces a traceback in the MGR logs. PG distribution may become suboptimal as a result.

**Cause**
A performance bottleneck and logic error in the balancer mgr module caused it to crash during normal operation on some cluster configurations.

**Workaround**
Disable the balancer module before upgrading to 19.2.0 and re-enable it after upgrading to 19.2.1+:
```bash
# Before upgrading to 19.2.0
ceph balancer off

# Verify cluster is healthy without balancer
ceph status

# After upgrading all components to 19.2.1+
ceph balancer on
ceph balancer mode upmap
```

**Fix**
Fixed in v19.2.1 (February 6, 2025). Upgrade to v19.2.1 or later before re-enabling the balancer. See tracker https://tracker.ceph.com/issues/68657.

---

### iSCSI: Upgrade regression from 19.1.1 to 19.2.0

| Field | Value |
|-------|-------|
| **Affected versions** | 19.1.1 → 19.2.0 upgrade path |
| **Component** | iSCSI / ceph-iscsi |
| **Severity** | High (iSCSI service disruption during upgrade) |
| **Status** | Known; see tracker for latest status |
| **Tracker** | https://tracker.ceph.com/issues/68215 |
| **PR** | — |

**Symptom**
iSCSI targets become unavailable or fail to reconnect during or after an upgrade from 19.1.1 to 19.2.0. iSCSI initiators may lose access to block devices mid-upgrade.

**Cause**
A bug encountered by Ceph upstream developers during internal upgrade testing from 19.1.1 to 19.2.0 caused iSCSI service disruption. The exact scope is detailed in tracker #68215.

**Workaround**
Read tracker https://tracker.ceph.com/issues/68215 in full before attempting any upgrade if your cluster hosts iSCSI targets.

General mitigation steps:
```bash
# Before upgrading, record existing iSCSI gateway configuration
ceph dashboard iscsi-gateway-list

# Quiesce iSCSI sessions from the initiator side before upgrading
iscsiadm -m node --logout

# Upgrade, then re-establish sessions
iscsiadm -m node --login
```

**Fix**
Consult tracker https://tracker.ceph.com/issues/68215 for the current status and any released fix.

---

### MGR REST module: OOM crash due to unbounded request accumulation

| Field | Value |
|-------|-------|
| **Affected versions** | 19.2.0 (mitigated in 19.2.1) |
| **Component** | MGR / REST module |
| **Severity** | Medium (MGR OOM crash under sustained REST API usage) |
| **Status** | Mitigated in v19.2.1 |
| **Tracker** | — |
| **PR** | — |

**Symptom**
The Ceph Manager process crashes with an Out-of-Memory (OOM) error when the REST manager module is enabled and the cluster receives a sustained volume of REST API requests. The request array grows without bound if old requests are never manually deleted.

**Cause**
The REST module retained all completed requests in memory indefinitely. Without a cap or automatic expiry, long-running clusters with heavy REST usage exhausted MGR memory.

**Workaround**
Set a request retention limit and periodically purge old requests:
```bash
# Limit request history (config key name may vary; check mgr module options)
ceph config set mgr mgr/restful/max_requests 100

# Manually delete old requests via the REST API if the module is responsive
# DELETE /api/request/<id>
```
If the MGR crashes repeatedly, disable the REST module temporarily:
```bash
ceph mgr module disable restful
```

**Fix**
v19.2.1 introduced automatic trimming of REST requests based on the `max_requests` configuration option. Upgrade to v19.2.1 or later.

---

*Source: official Ceph Squid release notes — https://github.com/ceph/ceph/blob/main/doc/releases/squid.rst*
*Last reviewed against: v19.2.3 (July 28, 2025)*
