# Reef to Squid Upgrade (Major Version)

This guide covers upgrading a Ceph cluster from Reef (18.2.x) to Squid (19.2.x). Squid is the 19th stable release of Ceph, introducing new RGW User Accounts IAM APIs, BlueStore performance improvements, Crimson tech preview, and significant CephFS enhancements.

**When to use this guide:** your cluster runs Reef 18.2.x and you are ready to move to Squid 19.2.x.

---

## Prerequisites

Before starting a Reef-to-Squid upgrade:

**1. Must be on the latest Reef point release**

You must upgrade to Reef 18.2.8 (the final Reef release, March 2026) before upgrading to Squid. Do not attempt a direct upgrade from an older Reef point release to Squid.

```bash
ceph versions
# All daemons must report: ceph version 18.2.8 (...)
```

If not already on 18.2.8, run a [minor upgrade](minor-upgrade.md) to 18.2.8 first.

**2. Skip v19.2.1 — upgrade to v19.2.2 or later**

> **WARNING:** Do not upgrade to Squid v19.2.1. A data-corrupting bug was identified in that release for RGW multisite deployments. Upgrade directly to v19.2.2 or later (v19.2.3 as of July 2025).

**3. Cluster must be stable and fully healthy**

```bash
ceph -s
ceph health detail
ceph pg stat
```

All PGs must be `active+clean`. No OSDs can be down. No hosts can be offline.

**4. Ensure iSCSI deployments are safe**

If your cluster has iSCSI gateways, read [Tracker Issue 68215](https://tracker.ceph.com/issues/68215) before upgrading. A bug was encountered during upgrades with iSCSI active.

---

## Breaking Changes in Squid

Review these changes before upgrading. Some require action before, during, or after the upgrade.

### Monitoring

- `mon_cluster_log_file_level` and `mon_cluster_log_to_syslog_level` config options have been **removed**. Replace them with the unified option `mon_cluster_log_level` before upgrading.

### RADOS / OSD

- `POOL_APP_NOT_ENABLED` health warnings will now be reported for all pools where `application enable` has not been set, regardless of whether the pool is in use. Tag all pools after upgrading:
  ```bash
  ceph osd pool application enable <pool-name> <app>
  ```
- `PG dump --format json` no longer includes the `network_ping_times` section. Scripts parsing this output must be updated.
- The `get_pool_is_selfmanaged_snaps_mode` C++ API is deprecated. Use `pool_is_in_selfmanaged_snaps_mode`.

### CephFS / MDS

- **`ceph fs rename` requires offline filesystem:** Before running `ceph fs rename`, the target filesystem must be offline and `refuse_client_session` must be set. After renaming, unset the config and bring the filesystem back online.
- **`root_squash` clients require new feature bit:** Clients using credentials with `root_squash` must support the `client_mds_auth_caps` feature bit. Clients that do not support this bit will trigger `MDS_CLIENTS_BROKEN_ROOTSQUASH` (`HEALTH_ERR`). Update CephFS clients before upgrading MDS daemons.
- **`mds_client_delegate_inos_pct` defaults to 0:** Async dirops in the kernel client (kclient) are now disabled by default.
- **Snap-schedule commands require `--fs` argument:** For clusters with multiple CephFS filesystems, all snap-schedule commands now require the `--fs` argument.
- **Period specifier change:** In snap-schedule, `m` now means minutes and `M` means months. Update any automation that uses these specifiers.
- **`ceph mds fail` / `ceph fs fail` require confirmation flag** when MDSs exhibit `MDS_TRIM` or `MDS_CACHE_OVERSIZED` health warnings.
- The CephFS automatic metadata load balancer is now **disabled by default**. Re-enable after upgrade if needed: `ceph fs set <fs_name> balance_automate true`

### RGW

- **`rgw_realm` config is now validated at startup.** If radosgw previously ignored an invalid `rgw_realm` value, it will now fail to start. Remove or correct the config before upgrading RGW daemons.
- **`realm create` and `realm pull` no longer set a default realm** without `--default`. Update automation scripts that rely on this behavior.
- **Notification topics are now owned by the creating user.** Only the owner can read/write their topics by default. Preexisting topics are ownerless and remain writable by all users until explicitly recreated. Review SNS topic policies after upgrading.
- **RGW SNS topic naming is enforced:** Topic names must match `[A-Za-z0-9_-]{1,256}`. Topics created with non-conforming names under Reef will continue to work but cannot be recreated with the same name after upgrading.
- **Notification v2 format (opt-in after upgrade):** The new per-topic RADOS object layout for bucket notifications is not enabled by default on upgrade. Enable after all zones are upgraded: `ceph zone feature enable notification_v2`. The v1 format is deprecated.

### RBD

- **`rbd-nbd` `try-netlink` is now the default and deprecated as an explicit option.** Remove `try-netlink` from any rbd-nbd map commands.
- Moving an RBD image that is a member of a group to trash is no longer allowed (`rbd trash mv` now behaves like `rbd rm` in this case).

---

## Pre-Upgrade Steps

### Step 1: Migrate removed config options

```bash
# Remove deprecated monitoring log level options
ceph config rm mon mon_cluster_log_file_level
ceph config rm mon mon_cluster_log_to_syslog_level

# Set the replacement option if log level control is needed
ceph config set mon mon_cluster_log_level info
```

### Step 2: Validate cluster health

```bash
ceph -s
ceph health detail
ceph pg stat
ceph osd stat
```

All must be clean before proceeding.

### Step 3: Confirm standby Manager

```bash
ceph orch ps --daemon-type mgr
# Must show at least 1 active + 1 standby
```

If only one manager is running, add a second:

```bash
ceph orch apply mgr 2
```

### Step 4: Disable PG autoscaler

```bash
ceph osd pool set noautoscale
```

### Step 5: Back up configuration and keyrings

```bash
ceph config dump > /root/ceph-config-backup-prereef-$(date +%Y%m%d).json
cp /etc/ceph/ceph.conf /root/ceph.conf.reef.bak
cp /etc/ceph/ceph.client.admin.keyring /root/ceph.client.admin.keyring.reef.bak
```

### Step 6: Pull the target image on all hosts

```bash
podman pull quay.io/ceph/ceph:v19.2.3
```

Verify the pull succeeds on each host that will run Ceph daemons. Network access to `quay.io` is required.

### Step 7: Set `noout` flag (optional, recommended)

```bash
ceph osd set noout
```

This prevents OSDs from being marked out during daemon restarts.

---

## Upgrade Procedure

### Start the upgrade

```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v19.2.3
```

### Monitor progress

```bash
# Live upgrade status
ceph orch upgrade status

# Full cluster status
watch -n 15 ceph -s

# Daemon version distribution
watch -n 30 ceph versions

# Verbose orchestrator log
ceph -W cephadm
```

cephadm will upgrade daemons in this order automatically: `mgr` → `mon` → `crash` → `osd` → `mds` → `rgw` → `rbd-mirror` → monitoring stack.

### Pause or resume if needed

```bash
ceph orch upgrade pause
ceph orch upgrade resume
```

### Watch for the balancer issue (v19.2.0 only)

If upgrading to v19.2.0 (not recommended — use v19.2.3), watch for a balancer module bug. If the balancer module crashes:

```bash
ceph balancer off
```

This is fixed in v19.2.1 and later.

---

## Post-Upgrade Tasks

Run these tasks after all daemons report the Squid version.

### Verify all daemons are upgraded

```bash
ceph versions
# All daemons must report version 19.2.3

ceph orch upgrade status
# Should show: no upgrade in progress
```

### Verify cluster health

```bash
ceph -s
ceph health detail
ceph pg stat
ceph osd stat
```

### Unset `noout` and re-enable PG autoscaler

```bash
ceph osd unset noout
ceph osd pool unset noautoscale
```

### Enable Squid-only OSD features

After all OSDs are upgraded, allow the cluster to use Squid-specific OSD features:

```bash
ceph osd require-osd-release squid
```

### Tag pools with application (if warnings appear)

If `POOL_APP_NOT_ENABLED` warnings appear:

```bash
# List all pools
ceph osd lspools

# Tag each pool with its application type
ceph osd pool application enable <pool-name> rbd      # for RBD pools
ceph osd pool application enable <pool-name> cephfs   # for CephFS pools
ceph osd pool application enable <pool-name> rgw      # for RGW pools
```

Silence warnings for internal pools not tied to a specific application:

```bash
ceph health mute POOL_APP_NOT_ENABLED
```

### Enable RGW notification v2 (multisite deployments)

After all RGW daemons across all zones have upgraded:

```bash
ceph zone feature enable notification_v2
```

### Update cephadm package

```bash
# On the bootstrap host
dnf update cephadm
# or
apt-get install --only-upgrade cephadm
```

### Verify CephFS (if applicable)

```bash
ceph fs status
ceph mds stat

# Check for any broken root_squash clients
ceph health detail | grep MDS_CLIENTS_BROKEN_ROOTSQUASH
```

If `MDS_CLIENTS_BROKEN_ROOTSQUASH` appears, identify the offending clients and update their CephFS client software before the MDS enforces the error.

### Verify RGW startup (if applicable)

```bash
ceph orch ps --daemon-type rgw
```

If any RGW daemons fail to start, check the logs for `failed to load realm` errors and correct the `rgw_realm` config option:

```bash
ceph log last 100 | grep rgw
ceph orch logs --name rgw.<service-name>
```

---

## Rollback Considerations

> **WARNING:** Rolling back from Squid to Reef after a major version upgrade is NOT supported. Once Squid OSDs have written data in the Squid on-disk format, the cluster cannot be safely reverted to Reef. There is no automated downgrade path.

**Mitigation:** If the upgrade fails part-way through (before all OSDs have been upgraded to Squid):

1. Stop the upgrade immediately: `ceph orch upgrade stop`
2. Do not allow any Squid-version OSD to start writing in Squid-only format. Mixed-version operation remains safe as long as `ceph osd require-osd-release squid` has not been run.
3. Assess whether completing the upgrade forward is the safer path — it usually is.
4. If you must recover, restore the cluster from a snapshot or backup taken before the upgrade began.

**Before upgrading to Squid, ensure you have:**
- A recent backup or snapshot of all data pools
- Tested the upgrade procedure in a non-production cluster
- Verified application compatibility with Squid-level changes

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/cephadm/upgrade.rst
  https://github.com/ceph/ceph/blob/main/doc/releases/reef.rst
  https://github.com/ceph/ceph/blob/main/doc/releases/squid.rst
-->
