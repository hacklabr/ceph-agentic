# CephFS — Ceph File System

CephFS is a POSIX-compliant distributed file system built on top of RADOS. The Metadata Server (MDS) daemon manages the directory tree, file inodes, and access control. OSD data pools store file contents. At least one active MDS must be running for any mount or filesystem operation to succeed.

**Prerequisites:** Cluster at `HEALTH_OK`, at least 3 OSDs available. Hosts intended for MDS must be added to cephadm and labeled before deploying the service spec.

---

## Label MDS Hosts

MDS is stateless with respect to data — it caches in memory and persists to RADOS. Any host with adequate RAM (minimum 4 GiB, 8 GiB recommended per active rank) can run MDS.

```bash
# Label hosts that will run MDS daemons
ceph orch host label add ceph01 mds
ceph orch host label add ceph02 mds
ceph orch host label add ceph03 mds
```

---

## Option A: Deploy via `ceph fs volume create` (Recommended)

The `volume create` command creates the metadata pool, the default data pool, and the MDS service in a single operation. This is the preferred method for new deployments.

```bash
ceph fs volume create cephfs --placement="label:mds"
```

This command:
- Creates pool `cephfs.cephfs.meta` (metadata pool, replicated)
- Creates pool `cephfs.cephfs.data` (default data pool, replicated)
- Deploys MDS daemons on hosts labeled `mds`
- Marks the new filesystem as active

Watch daemons come up:

```bash
ceph orch ps --daemon-type mds
```

```
NAME          HOST    PORTS  STATUS         REFRESHED  AGE  MEM USED  MEM LIMIT  VERSION  IMAGE ID      CONTAINER ID
mds.cephfs.0  ceph01         running (2m)   10s ago    2m    512.3M     -         18.2.7   <id>          <id>
mds.cephfs.1  ceph02         running (1m)   10s ago    1m    105.1M     -         18.2.7   <id>          <id>
mds.cephfs.2  ceph03         running (1m)   10s ago    1m    103.8M     -         18.2.7   <id>          <id>
```

One MDS is active, the rest are standbys.

---

## Option B: Manual Pool Creation + `fs new`

Use this approach when you need control over pool names, placement groups, or device class targeting (e.g., SSD metadata pool, HDD data pool).

### Create Pools

```bash
# Metadata pool — use SSD or NVMe device class for low latency
ceph osd pool create cephfs_metadata 64
ceph osd pool set cephfs_metadata size 3

# Data pool — HDDs are acceptable for bulk data
ceph osd pool create cephfs_data 128
ceph osd pool set cephfs_data size 3
```

Set application labels:

```bash
ceph osd pool application enable cephfs_metadata cephfs
ceph osd pool application enable cephfs_data cephfs
```

### Create the Filesystem

```bash
ceph fs new cephfs cephfs_metadata cephfs_data
```

Verify:

```bash
ceph fs ls
```

```
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

### Deploy MDS via Service Spec

Create `mds.yaml`:

```yaml
service_type: mds
service_id: cephfs
placement:
  count: 3
  label: mds
```

Apply the spec:

```bash
ceph orch apply -i mds.yaml
```

---

## Multiple Active MDS

By default, one MDS is active and all others are standbys. Enable multiple active ranks to parallelize metadata operations across large directory trees. Each additional active MDS takes one standby.

Enable 2 active ranks:

```bash
ceph fs set cephfs max_mds 2
```

Verify both ranks are active:

```bash
ceph mds stat
```

```
cephfs:2/2/2 up {0=cephfs.ceph01.abc=up:active,1=cephfs.ceph02.def=up:active}, 1 up:standby
```

### Service Spec for Multiple Active MDS

Deploy 4 daemons — 2 active + 2 standbys — from the spec:

```yaml
service_type: mds
service_id: cephfs
placement:
  count: 4
  label: mds
```

```bash
ceph orch apply -i mds.yaml
```

After applying, set the active rank count:

```bash
ceph fs set cephfs max_mds 2
```

> **Note:** More active MDS ranks improve throughput for workloads that spread across many directories, but do not help workloads concentrated in a single directory (hotspot). For a single-directory hotspot, a single active MDS with standby replay is more effective.

### Enable Standby Replay

A standby-replay MDS tails the active MDS journal to warm its cache. Failover recovery time drops from ~30 seconds to ~5 seconds.

```bash
ceph fs set cephfs allow_standby_replay true
```

---

## Create a Client User

Do not use `client.admin` for filesystem mounts. Create a restricted key that grants access only to the target filesystem.

```bash
ceph fs authorize cephfs client.cephfs-rw / rw
```

Write the key to disk on the client host:

```bash
ceph auth get client.cephfs-rw -o /etc/ceph/ceph.client.cephfs-rw.keyring
```

Retrieve the key string for FUSE or kernel mount options:

```bash
ceph auth get-key client.cephfs-rw
```

```
AQBxyz123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ==
```

---

## Mount with Kernel Driver

The kernel CephFS driver (`ceph` filesystem type) is available in Linux 4.1+ and well-supported from 5.4+. It offers the best performance for workloads that need raw throughput.

```bash
# Install ceph-common for /sbin/mount.ceph helper
apt-get install -y ceph-common   # Debian/Ubuntu
dnf install -y ceph-common       # RHEL/Rocky

# Create the mount point
mkdir -p /mnt/cephfs

# Mount using the secret key inline
mount -t ceph 10.0.0.10,10.0.0.11,10.0.0.12:/ /mnt/cephfs \
  -o name=cephfs-rw,secret=AQBxyz123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ==

# Or reference the keyring file
mount -t ceph 10.0.0.10,10.0.0.11,10.0.0.12:/ /mnt/cephfs \
  -o name=cephfs-rw,keyring=/etc/ceph/ceph.client.cephfs-rw.keyring
```

Mount a specific subdirectory (useful for multi-tenant layouts):

```bash
mount -t ceph 10.0.0.10,10.0.0.11,10.0.0.12:/data/tenant1 /mnt/tenant1 \
  -o name=cephfs-rw,keyring=/etc/ceph/ceph.client.cephfs-rw.keyring
```

Persist in `/etc/fstab`:

```
10.0.0.10,10.0.0.11,10.0.0.12:/  /mnt/cephfs  ceph  name=cephfs-rw,keyring=/etc/ceph/ceph.client.cephfs-rw.keyring,_netdev,noatime  0 0
```

Verify the mount:

```bash
stat -f /mnt/cephfs
```

```
  File: "/mnt/cephfs"
    ID: 123456abcdef Type: ceph
Block size: 4194304    Fundamental block size: 4194304
Blocks: Total: 2097152 Free: 2097150 Available: 2097150
Inodes: Total: 0 Free: 0
```

---

## Mount with FUSE

FUSE (`ceph-fuse`) runs in userspace. It is slower than the kernel driver but does not require a matching kernel version and works in containers and VMs where the kernel driver is unavailable.

```bash
# Install ceph-fuse
apt-get install -y ceph-fuse   # Debian/Ubuntu
dnf install -y ceph-fuse       # RHEL/Rocky

# Mount
mkdir -p /mnt/cephfs-fuse
ceph-fuse /mnt/cephfs-fuse \
  --id cephfs-rw \
  --keyring /etc/ceph/ceph.client.cephfs-rw.keyring \
  --client-mountpoint /
```

Run as a systemd unit for persistent mounts:

```bash
# Generate the systemd unit file
ceph-fuse /mnt/cephfs-fuse --id cephfs-rw --keyring /etc/ceph/ceph.client.cephfs-rw.keyring &

# Or use the mount helper
mount -t fuse.ceph ceph=/ /mnt/cephfs-fuse \
  -o id=cephfs-rw,conf=/etc/ceph/ceph.conf
```

Unmount:

```bash
fusermount -u /mnt/cephfs-fuse
```

---

## Directory Quotas

Quotas are set on directories and limit the total bytes or number of files stored in the directory tree rooted at that path.

```bash
# Set a 100 GiB quota on a directory
setfattr -n ceph.quota.max_bytes -v 107374182400 /mnt/cephfs/tenant1

# Set an inode count quota
setfattr -n ceph.quota.max_files -v 1000000 /mnt/cephfs/tenant1

# Read back the quota
getfattr -n ceph.quota.max_bytes /mnt/cephfs/tenant1
getfattr -n ceph.quota.max_files /mnt/cephfs/tenant1
```

Remove a quota:

```bash
setfattr -n ceph.quota.max_bytes -v 0 /mnt/cephfs/tenant1
```

> **Note:** Quotas are enforced by the MDS. Enforcement is eventually consistent — brief overages of a few megabytes are possible when multiple clients are writing concurrently. Do not rely on CephFS quotas for strict financial billing without a safety margin.

---

## Add a Data Pool

The default data pool is set at filesystem creation and cannot be changed. Add additional data pools for layout overrides — for example, an erasure-coded pool for archival directories.

```bash
ceph osd pool create cephfs_ec_data 64 64 erasure
ceph osd pool set cephfs_ec_data allow_ec_overwrites true
ceph fs add_data_pool cephfs cephfs_ec_data
```

Set a directory to use the new pool via layout:

```bash
setfattr -n ceph.dir.layout.pool -v cephfs_ec_data /mnt/cephfs/archive
```

---

## Verification Commands

```bash
# Check MDS status and rank assignment
ceph mds stat

# Show filesystem details (pools, flags, max_mds)
ceph fs status cephfs

# Show all filesystems
ceph fs ls

# Check client sessions connected to MDS
ceph tell mds.0 session ls

# Show MDS memory usage
ceph orch ps --daemon-type mds

# Show daemon-level cache stats
ceph tell mds.cephfs.0 dump cache 2>&1 | head -40

# Check client-visible mount
df -h /mnt/cephfs
stat -f /mnt/cephfs
```

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/cephfs/createfs.rst
  https://github.com/ceph/ceph/blob/main/doc/cephadm/services/mds.rst
  https://github.com/ceph/ceph/blob/main/doc/cephfs/mount-using-kernel-driver.rst
  https://github.com/ceph/ceph/blob/main/doc/cephfs/mount-using-fuse.rst
  https://github.com/ceph/ceph/blob/main/doc/cephfs/quota.rst
-->
