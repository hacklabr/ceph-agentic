# Upgrades

Upgrading Ceph with cephadm is fully automated. Provide a target container image and cephadm orchestrates every daemon in the correct order, waiting for cluster health to confirm availability before restarting the next daemon.

**When to use this guide:** you need to move from one Ceph point release to a newer one (minor upgrade), or you are planning a major version transition from Reef (18.2.x) to Squid (19.2.x).

---

## Upgrade Order

cephadm enforces a strict upgrade sequence regardless of how you invoke the upgrade:

1. `mgr` — Manager daemons (active + standbys)
2. `mon` — Monitor daemons
3. `crash`
4. `osd` — Object Storage Daemons
5. `mds` — Metadata Servers (CephFS)
6. `rgw` — RADOS Gateway
7. `rbd-mirror`, `cephfs-mirror`, `ceph-exporter`
8. `iscsi`, `nfs`, `nvmeof`, `smb`
9. Monitoring stack: `node-exporter`, `prometheus`, `alertmanager`, `grafana`, `loki`, `promtail`

Each daemon is restarted only after Ceph reports the cluster will remain available. The cluster will briefly enter `HEALTH_WARN` during the upgrade — this is expected.

---

## Pre-Upgrade Checklist

Run through every item before starting any upgrade:

**1. Verify cluster health**

```bash
ceph -s
ceph health detail
```

The cluster must be `HEALTH_OK` or `HEALTH_WARN` with no degraded PGs. Resolve any `HEALTH_ERR` conditions before proceeding. All hosts must be reachable.

**2. Confirm a standby Manager exists**

```bash
ceph orch ps --daemon-type mgr
```

At least one standby `mgr` daemon must be running. Without a standby the upgrade will stall on `UPGRADE_NO_STANDBY_MGR`. Add one if needed:

```bash
ceph orch apply mgr 2
```

**3. Check for known issues**

Review the release notes for the target version before upgrading:
- Reef release notes: https://docs.ceph.com/en/latest/releases/reef/
- Squid release notes: https://docs.ceph.com/en/latest/releases/squid/

**4. Pull and verify the target container image**

```bash
# Confirm the image tag exists before starting
podman pull quay.io/ceph/ceph:v18.2.8
```

For Reef-to-Squid: `quay.io/ceph/ceph:v19.2.3`

**5. Disable the PG autoscaler (recommended for large clusters)**

On large clusters, PG splitting or merging during upgrade can significantly extend the upgrade window:

```bash
ceph osd pool set noautoscale
# Perform the upgrade
ceph osd pool unset noautoscale
```

**6. Back up cluster configuration and key material**

```bash
ceph config dump > /root/ceph-config-backup-$(date +%Y%m%d).json
cp /etc/ceph/ceph.conf /root/ceph.conf.bak
cp /etc/ceph/ceph.client.admin.keyring /root/ceph.client.admin.keyring.bak
```

---

## cephadm Upgrade Process

### Start the upgrade

```bash
ceph orch upgrade start --ceph-version 18.2.8
# or equivalently with an explicit image:
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.8
```

### Monitor progress

```bash
# Summary with progress bar
ceph orch upgrade status

# Full cluster status (shows upgrade progress bar under 'progress:')
ceph -s

# Verbose cephadm log stream
ceph -W cephadm

# Watch which daemon versions are running across the cluster
watch ceph versions
```

### Pause or resume

```bash
ceph orch upgrade pause
ceph orch upgrade resume
```

### Stop the upgrade

```bash
ceph orch upgrade stop
```

Stopping the upgrade halts further restarts but does not roll back daemons already upgraded.

---

## Rollback Options

**Minor upgrades (within a release series, e.g. 18.2.6 → 18.2.8):**

If the upgrade is stopped before completion, mixed-version operation is generally safe for a short period. To roll back individual daemons, redeploy them with the previous image:

```bash
ceph orch daemon redeploy mgr.ceph-mon01.abcdef --image quay.io/ceph/ceph:v18.2.6
```

> **WARNING:** Rolling back individual daemons in a partially upgraded cluster is not supported by upstream and should only be used as a temporary measure while diagnosing a critical issue. Complete the upgrade or restore from a known-good cluster state.

**Major version upgrades (e.g. Reef → Squid):**

> **WARNING:** Downgrading from Squid back to Reef after a major version upgrade is NOT supported. There is no automated rollback path. Once Squid OSDs have written data in the new on-disk format, reverting to Reef will result in data unavailability. Ensure all pre-upgrade checks pass and test the upgrade in a non-production environment first.

---

## Choosing a Guide

| Scenario | Guide |
|----------|-------|
| 18.2.x → 18.2.y (same series) | [minor-upgrade.md](minor-upgrade.md) |
| 18.2.x (Reef) → 19.2.x (Squid) | [reef-to-squid.md](reef-to-squid.md) |

---

## Troubleshooting

### `UPGRADE_NO_STANDBY_MGR`

```bash
ceph orch apply mgr 2
ceph orch ps --daemon-type mgr
# Wait for second mgr to appear, then retry the upgrade
```

### `UPGRADE_FAILED_PULL`

The container image could not be pulled. Check network access to `quay.io` from all cluster hosts, then retry:

```bash
ceph orch upgrade stop
ceph orch upgrade start --ceph-version 18.2.8
```

### `Error ENOENT: Module not found` on `upgrade status`

The orchestrator module has crashed, possibly due to invalid JSON in a manager config key. Check:

```bash
ceph mgr module ls
ceph orch status
```

Restart the active manager to recover the orchestrator module:

```bash
ceph mgr fail
```

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/cephadm/upgrade.rst
  https://github.com/ceph/ceph/blob/main/doc/releases/reef.rst
  https://github.com/ceph/ceph/blob/main/doc/releases/squid.rst
-->
