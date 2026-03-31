# Minor Upgrade (Point Release)

A minor upgrade moves a cluster between patch releases within the same named series — for example, Reef 18.2.6 to Reef 18.2.8, or Squid 19.2.1 to Squid 19.2.3. cephadm handles the entire sequence automatically.

**When to use this guide:** you are staying within the same major version (same first two digits of the version number) and upgrading to a newer patch release for bug fixes or security patches.

---

## Confirm You Need This Guide

| Source | Destination | Use this guide? |
|--------|-------------|-----------------|
| 18.2.6 (Reef) | 18.2.8 (Reef) | Yes |
| 19.2.1 (Squid) | 19.2.3 (Squid) | Yes |
| 18.2.8 (Reef) | 19.2.x (Squid) | No — see [reef-to-squid.md](reef-to-squid.md) |

---

## Duration Estimate

| Cluster size | Estimated time |
|-------------|----------------|
| 3 nodes, 12 OSDs | 15–30 minutes |
| 10 nodes, 60 OSDs | 45–90 minutes |
| 30+ nodes, 200+ OSDs | 2–4 hours |

Times depend on OSD count, network speed to the container registry, and current I/O load. The upgrade is non-disruptive to I/O — clients continue reading and writing throughout.

---

## Full Upgrade Procedure

### Step 1: Verify cluster health

```bash
ceph -s
ceph health detail
```

The cluster must show `HEALTH_OK` or `HEALTH_WARN` with no degraded PGs. Do not start an upgrade against a cluster with down OSDs or unclean PGs.

```bash
# All PGs must be in active+clean
ceph pg stat
# Example expected output:
# 256 pgs: 256 active+clean; 512 GiB data, 1.5 TiB used, 10.5 TiB / 12 TiB avail
```

### Step 2: Confirm all hosts are reachable

```bash
ceph orch host ls
```

All hosts must show status `Online`. If any host is `Offline`, resolve connectivity before proceeding.

### Step 3: Confirm a standby Manager is running

```bash
ceph orch ps --daemon-type mgr
```

You need at least one standby `mgr`. If only the active manager is running:

```bash
ceph orch apply mgr 2
# Wait for the standby to appear
ceph orch ps --daemon-type mgr
```

### Step 4: Disable PG autoscaler (recommended)

```bash
ceph osd pool set noautoscale
```

This prevents PG splitting or merging from interfering with the upgrade timeline.

### Step 5: Check the target version exists

```bash
# Reef latest point release as of March 2026
podman pull quay.io/ceph/ceph:v18.2.8
```

For Squid:

```bash
podman pull quay.io/ceph/ceph:v19.2.3
```

If the pull fails, verify network access to `quay.io` from all cluster hosts.

### Step 6: Start the upgrade

For the latest Reef point release:

```bash
ceph orch upgrade start --ceph-version 18.2.8
```

Or with an explicit image reference:

```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.8
```

For Squid:

```bash
ceph orch upgrade start --ceph-version 19.2.3
```

### Step 7: Monitor progress

```bash
# Live upgrade status with daemon counts
ceph orch upgrade status

# Cluster status with progress bar
watch -n 10 ceph -s

# Daemon version distribution (updates as daemons restart)
watch -n 30 ceph versions

# Verbose log from the orchestrator
ceph -W cephadm
```

Expected output from `ceph orch upgrade status` during the upgrade:

```
Upgrading to quay.io/ceph/ceph:v18.2.8
  target image id: sha256:abc123...
  target image name: quay.io/ceph/ceph:v18.2.8
  overall progress: 4/14 daemons
  mgr: complete
  mon: complete
  osd:   4/8 up to date
  mds: waiting
  rgw: waiting
```

The cluster will enter `HEALTH_WARN` during the upgrade. This is expected and does not indicate a problem.

### Step 8: Pause or resume if needed

```bash
# Pause the upgrade (no further daemon restarts)
ceph orch upgrade pause

# Resume from where it stopped
ceph orch upgrade resume
```

### Step 9: Wait for completion

The upgrade is complete when `ceph orch upgrade status` reports no active upgrade and all daemons have been restarted.

```bash
ceph orch upgrade status
# Expected: no output (no upgrade in progress)

ceph versions
# All daemons should report the same target version
```

---

## Verify the Upgrade

Run this full verification sequence after the upgrade completes:

```bash
# 1. Confirm cluster is healthy
ceph -s

# 2. Confirm all daemons are on the new version
ceph versions

# 3. Confirm no PGs are degraded or stuck
ceph pg stat

# 4. Confirm all OSDs are up and in
ceph osd stat

# 5. Re-enable PG autoscaler
ceph osd pool unset noautoscale

# 6. Confirm autoscaler mode is restored
ceph osd pool autoscale-status
```

Expected final state:

```
cluster:
  id:     477e46f1-ae41-4e43-9c8f-72c918ab0a20
  health: HEALTH_OK

services:
  mon: 3 daemons, quorum ceph-mon01,ceph-mon02,ceph-mon03 (age 12m)
  mgr: ceph-mon01.abcdef(active, since 10m), standbys: ceph-mon02.fedcba
  osd: 12 osds: 12 up (since 5m), 12 in
```

---

## Stopping the Upgrade

If you need to abort the upgrade before it completes:

```bash
ceph orch upgrade stop
```

This stops further daemon restarts. Daemons already upgraded remain on the new version. Mixed-version operation is stable for a short period, but complete or roll back promptly.

> **WARNING:** Rolling back daemons already upgraded to the new version by redeploying them with the old image is not supported by upstream and may cause instability. If you must abort, plan to complete the upgrade on the next available maintenance window rather than leaving the cluster in a mixed-version state indefinitely.

---

## Staggered Upgrade (Optional)

For clusters where you want fine-grained control, limit the upgrade to specific daemon types or hosts:

```bash
# Upgrade only MGR and MON daemons first
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.8 --daemon-types mgr,mon

# Then upgrade OSDs on specific hosts
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.8 --daemon-types osd --hosts ceph-osd01,ceph-osd02

# Limit to 3 OSD daemons at a time
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.8 --daemon-types osd --limit 3
```

Note: cephadm still enforces the global ordering constraint — specifying `--daemon-types osd` before `mgr`/`mon` are upgraded will produce a warning.

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/cephadm/upgrade.rst
  https://github.com/ceph/ceph/blob/main/doc/releases/reef.rst
  https://github.com/ceph/ceph/blob/main/doc/releases/squid.rst
-->
