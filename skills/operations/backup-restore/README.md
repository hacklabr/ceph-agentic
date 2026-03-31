# Backup and Restore

Ceph does not provide a single unified backup command. Each subsystem has its own tools and recovery path. This guide covers all of them with complete command sequences.

**When to use this guide:** scheduling routine backups, setting up replication between sites, preparing for a major operation, or responding to data loss.

---

## MON Store Backup

The MON store is the most critical state in a Ceph cluster. It contains the cluster map, OSD map, CRUSH map, authentication keyrings, and configuration. Losing it without a backup requires a full cluster rebuild.

**Schedule daily minimum.** The backup is small (typically under 100 MB) and the command is safe to run on a live cluster.

**Back up the MON store on every monitor node:**

```bash
# Run on each mon host
ceph-mon --extract-monmap /tmp/monmap --id $(hostname -s)
ceph-mon --extract-monmap /tmp/monmap --id $(hostname -s) --keyring /etc/ceph/ceph.client.admin.keyring
```

The simpler and recommended approach uses `ceph-objectstore-tool` directly against the store:

```bash
# Identify the mon data directory (default: /var/lib/ceph/mon/ceph-<hostname>)
MON_DATA=/var/lib/ceph/mon/ceph-$(hostname -s)

# Create a tarball of the entire mon store
systemctl stop ceph-mon@$(hostname -s)
tar czf /backup/mon-$(hostname -s)-$(date +%Y%m%d).tar.gz -C $MON_DATA .
systemctl start ceph-mon@$(hostname -s)
```

> **Warning:** Stopping a monitor reduces quorum temporarily. Only stop one monitor at a time. Confirm quorum is restored before stopping the next.

Check cluster health after restarting each monitor:

```bash
ceph -s
ceph quorum_status --format json-pretty | jq '.quorum_names'
```

**Also back up the admin keyring and ceph.conf:**

```bash
cp /etc/ceph/ceph.conf /backup/ceph.conf-$(date +%Y%m%d)
cp /etc/ceph/ceph.client.admin.keyring /backup/ceph.client.admin.keyring-$(date +%Y%m%d)
```

Store backups off-cluster on a system that is not part of the Ceph cluster.

---

## RBD Snapshots

Snapshots are read-only, crash-consistent point-in-time copies of an RBD image. They do not require the image to be detached.

> **Note:** RBD snapshots are crash-consistent but not application-consistent. Quiesce I/O in the guest OS before snapping for best results. For virtual machines, use `qemu-guest-agent` or `fsfreeze` inside the guest.

### Create a Snapshot

```bash
rbd snap create <pool>/<image>@<snap-name>

# Example
rbd snap create rbd/db-vol@2026-03-29
```

### List Snapshots

```bash
rbd snap ls <pool>/<image>

# Example
rbd snap ls rbd/db-vol
# SNAPID  NAME        SIZE    PROTECTED  TIMESTAMP
# 4       2026-03-29  50 GiB             Sun Mar 29 08:00:01 2026
```

### Roll Back to a Snapshot

> **Warning:** Rolling back overwrites the current image with snapshot data. All writes since the snapshot was taken are lost. This cannot be undone.

```bash
# Stop all clients accessing the image first
rbd snap rollback <pool>/<image>@<snap-name>

# Example
rbd snap rollback rbd/db-vol@2026-03-29
```

Rolling back large images is slow. Cloning from a snapshot is faster than rollback for most recovery scenarios — see the layering section in the RBD snapshot docs.

### Delete a Snapshot

```bash
rbd snap rm <pool>/<image>@<snap-name>

# Example
rbd snap rm rbd/db-vol@2026-03-29
```

Deletion is asynchronous — OSD snaptrim runs in the background. Capacity is not freed immediately.

### Purge All Snapshots

```bash
rbd snap purge <pool>/<image>

# Example
rbd snap purge rbd/db-vol
```

### Protect and Clone a Snapshot

To use a snapshot as a template for clones, protect it first:

```bash
rbd snap protect <pool>/<image>@<snap-name>
rbd clone <pool>/<image>@<snap-name> <pool>/<new-image>

# Example
rbd snap protect rbd/ubuntu-22.04@base
rbd clone rbd/ubuntu-22.04@base rbd/vm-101
```

A protected snapshot cannot be deleted until it is unprotected:

```bash
rbd snap unprotect <pool>/<image>@<snap-name>
```

Flatten a clone to remove its dependency on the parent snapshot:

```bash
rbd flatten <pool>/<image>
```

---

## RBD Export/Import

Export/import transfers image data to files or across clusters. Use this for off-cluster backup, migration, or seeding a secondary cluster.

### Full Export

```bash
rbd export <pool>/<image> /backup/<filename>.raw

# Example — export a specific snapshot
rbd export rbd/db-vol@2026-03-29 /backup/db-vol-2026-03-29.raw
```

Export while the image is in use is safe but the result is only crash-consistent. Snap and export the snapshot for a consistent backup.

### Full Import

> **Warning:** Import creates a new image. If an image with the target name already exists and you redirect to it, existing data is overwritten.

```bash
rbd import /backup/<filename>.raw <pool>/<image>

# Example
rbd import /backup/db-vol-2026-03-29.raw rbd/db-vol-restored
```

### Incremental Export

Incremental export transfers only the blocks that changed between two snapshots. This is significantly faster and smaller than a full export for large images with light churn.

```bash
# Create a fresh snapshot representing the new state
rbd snap create rbd/db-vol@2026-03-30

# Export the delta between two snapshots
rbd export-diff --from-snap 2026-03-29 rbd/db-vol@2026-03-30 /backup/db-vol-diff-2026-03-30.raw
```

### Incremental Import (Merge Diff)

```bash
# Apply the diff on top of the previously imported full backup
rbd import-diff /backup/db-vol-diff-2026-03-30.raw rbd/db-vol-restored
```

The target image must have the base snapshot that the diff was created from. Apply diffs in order — gaps in the chain break the merge.

### Streamed Pipeline (No Intermediate File)

```bash
# Export directly to a remote host via SSH
rbd export rbd/db-vol - | ssh backup-host "rbd import - backup-pool/db-vol"

# Incremental export over SSH
rbd export-diff --from-snap prev-snap rbd/db-vol@new-snap - \
  | ssh backup-host "rbd import-diff - backup-pool/db-vol"
```

---

## RBD Mirroring

RBD mirroring provides asynchronous replication of images between two Ceph clusters. It is the primary mechanism for cross-site DR for block storage.

Two replication modes are available:

| Mode | Mechanism | RPO | Failover Readiness |
|------|-----------|-----|--------------------|
| **Journal-based** | Every write goes to the journal first, then replays on secondary | Very low (near-continuous) | Image is immediately usable after promotion |
| **Snapshot-based** | Periodic mirror-snapshots define sync points | Depends on snapshot interval | Must complete syncing the last snapshot delta before failover |

Journal-based mode roughly doubles write latency because each write is committed to the journal before acknowledging to the client. Snapshot-based mode has no write latency overhead.

### Enable Pool-Level Mirroring (Snapshot Mode)

Run on both clusters:

```bash
# On site-a
rbd --cluster site-a mirror pool enable --site-name site-a image-pool image

# On site-b
rbd --cluster site-b mirror pool enable --site-name site-b image-pool image
```

Modes:
- `image` — mirroring must be enabled per image explicitly
- `pool` — all images with journaling enabled are mirrored automatically

### Bootstrap Peer Authentication

On site-a, generate a bootstrap token:

```bash
rbd --cluster site-a mirror pool peer bootstrap create --site-name site-a image-pool
# Outputs a base64 token — copy it
```

On site-b, import the token:

```bash
cat <<EOF > /tmp/mirror-token
<paste-token-here>
EOF
rbd --cluster site-b mirror pool peer bootstrap import --site-name site-b image-pool /tmp/mirror-token
```

Verify the peer is registered:

```bash
rbd --cluster site-a mirror pool info image-pool
```

### Enable Mirroring on an Individual Image

```bash
# Snapshot-based (no write latency overhead)
rbd --cluster site-a mirror image enable image-pool/image-1 snapshot

# Journal-based (near-continuous replication, ~2x write latency)
rbd --cluster site-a mirror image enable image-pool/image-2 journal
```

For journal-based mode, enable the journaling feature first if the image does not already have it:

```bash
rbd --cluster site-a feature enable image-pool/image-1 journaling
```

### Schedule Automatic Mirror Snapshots (Snapshot Mode)

```bash
# Every 24 hours starting at 06:00 UTC for the whole pool
rbd --cluster site-a mirror snapshot schedule add --pool image-pool 24h 2026-01-01T06:00:00

# Every 6 hours for a specific image
rbd --cluster site-a mirror snapshot schedule add --pool image-pool --image image-1 6h
```

List schedules and status:

```bash
rbd --cluster site-a mirror snapshot schedule ls --pool image-pool --recursive
rbd --cluster site-a mirror snapshot schedule status
```

Trigger a mirror snapshot manually:

```bash
rbd --cluster site-a mirror image snapshot image-pool/image-1
```

### Check Mirror Status

```bash
# Summary for the pool
rbd --cluster site-a mirror pool status image-pool

# Verbose — status per image
rbd --cluster site-a mirror pool status image-pool --verbose

# Status for a single image
rbd --cluster site-a mirror image status image-pool/image-1
```

### Failover: Demote Primary, Promote Secondary

Stop all client I/O to the image on site-a first.

```bash
# Demote on site-a (marks image non-primary)
rbd --cluster site-a mirror image demote image-pool/image-1

# Promote on site-b
rbd --cluster site-b mirror image promote image-pool/image-1
```

To demote/promote the entire pool at once:

```bash
rbd --cluster site-a mirror pool demote image-pool
rbd --cluster site-b mirror pool promote image-pool
```

> **Warning:** Forced promotion (`--force`) without prior demotion creates a split-brain. The image will not resync automatically until you explicitly request it.

### Failback After Recovery

After site-a is restored and replication is caught up:

```bash
# Demote on site-b
rbd --cluster site-b mirror image demote image-pool/image-1

# Promote on site-a
rbd --cluster site-a mirror image promote image-pool/image-1
```

### Recover from Split-Brain

If split-brain occurs (detected by `rbd-mirror` daemon):

```bash
# Demote the stale copy first
rbd --cluster site-a mirror image demote image-pool/image-1

# Request a full resync from the primary
rbd mirror image resync image-pool/image-1
```

The `rbd-mirror` daemon performs the resync asynchronously. Monitor with `mirror image status`.

### Deploy the rbd-mirror Daemon

```bash
# Create a dedicated user
ceph auth get-or-create client.rbd-mirror.site-a \
  mon 'profile rbd-mirror' \
  osd 'profile rbd'

# Deploy via cephadm
ceph orch apply rbd-mirror --placement=host:rbd-mirror-host

# Or enable via systemd
systemctl enable --now ceph-rbd-mirror@rbd-mirror.site-a
```

---

## CephFS Backup

CephFS does not have a single dedicated backup daemon in older releases. Use a combination of snapshots and filesystem-level tools.

### CephFS Snapshots

CephFS snapshots are stored in hidden `.snap` directories within the filesystem. They are crash-consistent at the directory tree level.

Enable snapshots on a filesystem:

```bash
ceph fs set <fs-name> allow_new_snaps true
```

Create a snapshot from a mounted client:

```bash
# Inside the mounted CephFS tree
mkdir /mnt/cephfs/.snap/snapshot-2026-03-29
```

List snapshots:

```bash
ls /mnt/cephfs/.snap/
```

Delete a snapshot:

```bash
rmdir /mnt/cephfs/.snap/snapshot-2026-03-29
```

Snapshots are available per-directory and protect the subtree beneath that directory.

### cephfs-mirror Daemon (Active Replication)

`cephfs-mirror` continuously replicates a CephFS filesystem to a remote CephFS filesystem. Available since Pacific.

Deploy the mirror daemon via cephadm:

```bash
ceph orch apply cephfs-mirror
```

Enable mirroring on the filesystem:

```bash
ceph fs mirror enable <fs-name>
```

Add a remote peer:

```bash
# The peer spec uses: <client>@<fsid>/<fs-name>
cephfs-mirror --id mirror-peer peer_add <local-fs-name> client.mirror@<remote-fsid>/<remote-fs-name>
```

Check mirror status:

```bash
ceph fs snapshot mirror status <fs-name>
```

### rsync / restic Backup

For traditional off-cluster backup of CephFS data, mount the filesystem on a dedicated backup host and use standard tools:

```bash
# rsync to a remote backup server (run inside a snapshot directory for consistency)
rsync -avz --delete /mnt/cephfs/.snap/snapshot-2026-03-29/ backup-host:/backup/cephfs/

# restic backup from a mounted snapshot
restic -r s3:https://s3.example.com/ceph-backups backup /mnt/cephfs/.snap/snapshot-2026-03-29
```

Always backup from a snapshot, not the live tree, to ensure consistency.

### Export the MDS Journal (Pre-Recovery Step)

Before any destructive recovery action, export the MDS journal:

```bash
cephfs-journal-tool journal export /backup/mds-journal-$(date +%Y%m%d).bin
```

Restore from export if recovery goes wrong:

```bash
cephfs-journal-tool journal import /backup/mds-journal-20260329.bin
```

---

## RGW Multisite

RGW multisite provides active-active object replication between zones. Writes to any zone are synchronised to all other zones in the zonegroup. This is both a DR mechanism and a geographic read-locality feature.

**Topology:** realm > zonegroup > zones. One zonegroup master zone is authoritative for metadata (user accounts, bucket names). All zones receive object data via the `ceph-radosgw` sync engine — no separate sync daemon is required.

### Set Up the Master Zone (run on rgw1)

```bash
# 1. Create a realm
radosgw-admin realm create --rgw-realm=prod --default

# 2. Create the master zonegroup
radosgw-admin zonegroup create \
  --rgw-zonegroup=us \
  --endpoints=http://rgw1:80 \
  --rgw-realm=prod \
  --master --default

# 3. Create the master zone
radosgw-admin zone create \
  --rgw-zonegroup=us \
  --rgw-zone=us-east \
  --master --default \
  --endpoints=http://rgw1:80

# 4. Create a system user (used by secondary zones to authenticate)
radosgw-admin user create \
  --uid=sync-user \
  --display-name="Sync User" \
  --system
# Note the access_key and secret_key from the output

# 5. Add the system user credentials to the zone
radosgw-admin zone modify \
  --rgw-zone=us-east \
  --access-key=<access-key> \
  --secret=<secret-key>

# 6. Commit the period
radosgw-admin period update --commit
```

Add `rgw_zone=us-east` to `/etc/ceph/ceph.conf` under `[client.rgw.rgw1]` and restart the gateway:

```bash
systemctl restart ceph-radosgw@rgw.rgw1
```

### Set Up the Secondary Zone (run on rgw2)

```bash
# 1. Pull the realm configuration from the master zone
radosgw-admin realm pull \
  --url=http://rgw1:80 \
  --access-key=<access-key> \
  --secret=<secret-key>

radosgw-admin realm default --rgw-realm=prod

# 2. Pull the current period
radosgw-admin period pull \
  --url=http://rgw1:80 \
  --access-key=<access-key> \
  --secret=<secret-key>

# 3. Create the secondary zone
radosgw-admin zone create \
  --rgw-zonegroup=us \
  --rgw-zone=us-west \
  --access-key=<access-key> \
  --secret=<secret-key> \
  --endpoints=http://rgw2:80

# 4. Commit the period
radosgw-admin period update --commit
```

Add `rgw_zone=us-west` to `/etc/ceph/ceph.conf` under `[client.rgw.rgw2]` and restart:

```bash
systemctl restart ceph-radosgw@rgw.rgw2
```

### Check Synchronisation Status

```bash
radosgw-admin sync status
```

Healthy output shows all shards at incremental sync with "data is caught up with source". A behind shard will self-heal. A recovery shard retries on its own schedule.

### Verify Object Integrity (Optional)

Enable MD5 checksum verification on sync (adds CPU overhead):

```bash
radosgw-admin zone modify --rgw-zone=us-west \
  --rgw_sync_obj_etag_verify=true
radosgw-admin period update --commit
```

---

## Disaster Recovery

Recovery path depends on what is lost. Do not mix recovery procedures for different failure scenarios.

### Decision Tree

```
Has the cluster lost quorum (fewer than (N/2)+1 monitors reachable)?
├── YES → Restore MON store from backup, then rebuild quorum
└── NO  → Quorum is intact
        Has OSD data been permanently lost (drives destroyed, PGs lost)?
        ├── YES → How much is lost?
        │         ├── One or two OSDs (within erasure tolerance) → Mark out, reweight, wait for recovery
        │         └── Multiple OSDs, PGs lost beyond tolerance  → Declare data loss: ceph pg force-recovery / ceph osd force-create-pg
        └── NO  → Is the MDS down:damaged or CephFS unresponsive?
                  ├── YES → CephFS metadata repair (see below)
                  └── NO  → RBD image corruption? → rbd import from last known-good export
```

### Restore MON Quorum from Backup

If a monitor cannot start due to a corrupted store:

```bash
# Stop the damaged monitor
systemctl stop ceph-mon@<hostname>

# Remove the corrupted store
rm -rf /var/lib/ceph/mon/ceph-<hostname>/*

# Restore from backup
tar xzf /backup/mon-<hostname>-<date>.tar.gz -C /var/lib/ceph/mon/ceph-<hostname>/
chown -R ceph:ceph /var/lib/ceph/mon/

# Restart
systemctl start ceph-mon@<hostname>
ceph -s
```

If all monitors are lost and no backup exists, a full cluster rebuild from OSD data may be possible using `ceph-monstore-tool` — contact Ceph support before attempting this.

### Partial OSD Loss (Within Redundancy Tolerance)

```bash
# Mark the dead OSD out to trigger rebalancing
ceph osd out osd.<id>

# Watch recovery
watch ceph -s

# After recovery completes, remove the OSD entry
ceph osd purge osd.<id> --yes-i-really-mean-it
```

### CephFS Metadata Repair

The filesystem must be offline before running repair tools. Take it down first:

```bash
ceph fs fail <fs-name>
```

> **Warning:** Run each step in the exact order shown. Stop and seek expert help if any step fails. These tools can cause data loss if used incorrectly.

**Step 1 — Export the journal before touching anything:**

```bash
cephfs-journal-tool journal export /backup/mds-journal-before-repair.bin
```

**Step 2 — Recover dentries from the journal:**

```bash
cephfs-journal-tool event recover_dentries summary
```

**Step 3 — If the journal is corrupt, truncate it (last resort — causes metadata loss):**

```bash
cephfs-journal-tool --rank=<fs-name>:0 journal reset --yes-i-really-really-mean-it
```

**Step 4 — Reset MDS tables if RADOS objects are missing or inconsistent:**

```bash
cephfs-table-tool all reset session
cephfs-table-tool all reset snap
cephfs-table-tool all reset inode
```

**Step 5 — Rebuild metadata from the data pool (if metadata objects are missing):**

```bash
# Enable progress reporting
ceph mgr module enable cli_api

# Phase 1: scan extents (can be parallelised with --worker_n / --worker_m)
cephfs-data-scan scan_extents <data-pool>

# Phase 2: scan inodes (only after all scan_extents workers finish)
cephfs-data-scan scan_inodes <data-pool>

# Phase 3: fix inode linkages
cephfs-data-scan scan_links

# Cleanup
cephfs-data-scan cleanup <data-pool>
```

**Step 6 — Reset the MDS map to a single rank:**

```bash
ceph fs reset <fs-name> --yes-i-really-mean-it
```

**Step 7 — Bring the filesystem back online and run a full scrub:**

```bash
ceph fs set <fs-name> joinable true
# Wait for MDS to become active
ceph tell mds.<fs-name>:0 scrub start / recursive,repair,force
```

Remount all CephFS clients after the scrub completes.

### RBD Image Recovery

If an RBD image is corrupted and a snapshot exists:

```bash
# Roll back to the last clean snapshot
rbd snap rollback <pool>/<image>@<last-clean-snap>
```

If no snapshot exists but an export backup was taken:

```bash
# Import the backup as a new image first, then verify it, then rename
rbd import /backup/<image-backup>.raw <pool>/<image>-recovered
# Verify the data, then rename if satisfied
rbd rename <pool>/<image>-recovered <pool>/<image>
```

---

<!-- Reference: official Ceph docs used to compile this guide
  - https://docs.ceph.com/en/latest/rbd/rbd-snapshot/
  - https://docs.ceph.com/en/latest/rbd/rbd-mirroring/
  - https://docs.ceph.com/en/latest/radosgw/multisite/
  - https://docs.ceph.com/en/latest/cephfs/disaster-recovery-experts/
-->
