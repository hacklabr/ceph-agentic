# RBD — RADOS Block Device

RBD provides thin-provisioned block devices backed directly by RADOS. No gateway daemon sits between a client and the OSDs — the `librbd` or `krbd` kernel driver handles striping and replication negotiation directly. This makes RBD the lowest-latency storage path in a Ceph cluster.

**Prerequisites:** At least one replicated pool's worth of available OSDs (`ceph status` shows `HEALTH_OK`). The `ceph-common` package must be installed on any client host using the `rbd` CLI or kernel driver.

---

## Create the RBD Pool

A dedicated pool for RBD images makes quota management and CRUSH rule targeting straightforward. The default pool `rbd` is acceptable for small or test clusters but lacks namespace isolation.

```bash
# Create the pool — adjust pg_num based on OSD count (see pool management guide)
ceph osd pool create rbd 64

# Initialize the pool for RBD use
rbd pool init rbd
```

Set the application label so the cluster knows the pool type:

```bash
ceph osd pool application enable rbd rbd
```

Verify:

```bash
ceph osd pool ls detail | grep rbd
```

Expected output:

```
pool 3 'rbd' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 64 pgp_num 64 autoscale_mode on last_change 42 flags hashpspool stripe_width 0 application rbd
```

---

## Create a Block Device User

Run Ceph daemons and clients with a dedicated key that limits capabilities to the `rbd` pool. Using `client.admin` in production exposes full cluster administrative access.

```bash
# Create a client key with RBD read/write access to the rbd pool
ceph auth get-or-create client.rbd \
  mon 'profile rbd' \
  osd 'profile rbd pool=rbd' \
  mgr 'profile rbd pool=rbd' \
  -o /etc/ceph/ceph.client.rbd.keyring
```

For a read-only client (e.g., a backup host):

```bash
ceph auth get-or-create client.rbd-ro \
  mon 'profile rbd' \
  osd 'profile rbd-read-only pool=rbd' \
  mgr 'profile rbd pool=rbd' \
  -o /etc/ceph/ceph.client.rbd-ro.keyring
```

---

## Create a Block Device Image

Images are thin-provisioned — the `--size` sets the maximum addressable size, but no physical space is allocated until data is written.

```bash
# Create a 50 GiB image named "vm-disk-01" in the rbd pool
rbd create --size 51200 rbd/vm-disk-01
```

List all images in the pool:

```bash
rbd ls rbd
```

```
vm-disk-01
```

Inspect the image:

```bash
rbd info rbd/vm-disk-01
```

```
rbd image 'vm-disk-01':
	size 50 GiB in 12800 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 1a2b3c4d5e6f
	block_name_prefix: rbd_data.1a2b3c4d5e6f
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features:
	flags:
	create_timestamp: Mon Mar  1 10:00:00 2025
	access_timestamp: Mon Mar  1 10:00:00 2025
	modify_timestamp: Mon Mar  1 10:00:00 2025
```

---

## RBD Features and Kernel Compatibility

The default feature set requires a recent kernel. Older kernels (< 5.4) and some guest VMs using the `krbd` kernel driver cannot handle all features. The `exclusive-lock`, `object-map`, `fast-diff`, and `deep-flatten` features require kernel 4.9+ and are fully supported from kernel 5.4+.

| Feature | Kernel driver support | Notes |
|---|---|---|
| `layering` | All kernels | Required for cloning/snapshots |
| `exclusive-lock` | 4.9+ | Prevents simultaneous multi-writer corruption |
| `object-map` | 4.9+ | Speeds up diff and export operations |
| `fast-diff` | 4.9+ | Required for incremental backup |
| `deep-flatten` | 5.1+ | Flattens snapshot chains |
| `journaling` | Not supported in krbd | Only usable via `librbd` (QEMU/RBD mirroring) |

Create an image with a reduced feature set for older kernel clients:

```bash
# Compatible with kernels 4.9+
rbd create --size 51200 \
  --image-feature layering,exclusive-lock,object-map,fast-diff,deep-flatten \
  rbd/vm-disk-compat
```

Disable unsupported features on an existing image:

> **WARNING:** Disabling features on an in-use image can cause data inconsistency if clients depend on those features. Unmount and unmap the image before disabling features.

```bash
rbd feature disable rbd/vm-disk-01 deep-flatten
```

---

## Map and Mount an Image (Kernel Driver)

Use `rbd map` (which uses the `krbd` kernel module) to expose the image as a block device on a Linux host. The block device appears under `/dev/rbd<N>`.

```bash
# Map the image using the rbd client key
rbd map rbd/vm-disk-01 --id rbd --keyring /etc/ceph/ceph.client.rbd.keyring
```

```
/dev/rbd0
```

Verify the mapping:

```bash
rbd showmapped
```

```
id  pool  namespace  image        snap  device
0   rbd              vm-disk-01   -     /dev/rbd0
```

Format the device and mount it (first use only — never reformat an existing image):

> **WARNING:** `mkfs` is destructive. Only run it on a new, empty image. Running it on an image with existing data will destroy all data on that image.

```bash
mkfs.xfs /dev/rbd0
mkdir -p /mnt/rbd0
mount /dev/rbd0 /mnt/rbd0
```

Verify the mount:

```bash
df -h /mnt/rbd0
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0        50G  389M   50G   1% /mnt/rbd0
```

### Persist the Map and Mount

To map automatically after reboots, install `rbdmap` (part of `ceph-common`) and add an entry to `/etc/ceph/rbdmap`:

```
rbd/vm-disk-01  id=rbd,keyring=/etc/ceph/ceph.client.rbd.keyring
```

Enable the service:

```bash
systemctl enable rbdmap.service
```

Add the mount to `/etc/fstab`:

```
/dev/rbd/rbd/vm-disk-01  /mnt/rbd0  xfs  noauto,_netdev  0 0
```

The `noauto,_netdev` options ensure the mount is not attempted before the network comes up.

### Unmap

```bash
umount /mnt/rbd0
rbd unmap /dev/rbd0
```

---

## Resize an Image

Increase the image size (safe at any time — does not require unmounting):

```bash
rbd resize --size 102400 rbd/vm-disk-01
```

After resize, expand the filesystem to use the new space:

```bash
xfs_growfs /mnt/rbd0
```

Decrease the image size:

> **WARNING:** Shrinking a block device truncates data at the new boundary. Shrink the filesystem first, then shrink the image. Never shrink an image below the filesystem's current size.

```bash
# 1. Shrink the filesystem first
xfs_fsr /mnt/rbd0  # XFS does not support shrink — use ext4 with resize2fs if shrink is required

# 2. Then shrink the image
rbd resize --size 25600 rbd/vm-disk-01 --allow-shrink
```

---

## Snapshots

Snapshots are crash-consistent point-in-time copies. They do not require unmounting the image but quiescing I/O produces a more consistent snapshot.

Create a snapshot:

```bash
rbd snap create rbd/vm-disk-01@snap-2025-03-01
```

List snapshots:

```bash
rbd snap ls rbd/vm-disk-01
```

```
SNAPID  NAME             SIZE    PROTECTED  TIMESTAMP
     4  snap-2025-03-01  50 GiB  false      Mon Mar  1 10:30:00 2025
```

Roll back the image to a snapshot:

> **WARNING:** Rollback overwrites all current image data with the snapshot state. Unmount and unmap the image before rolling back. This is irreversible — create a new snapshot of the current state first if you want to preserve it.

```bash
rbd snap rollback rbd/vm-disk-01@snap-2025-03-01
```

Protect a snapshot before cloning from it (protected snapshots cannot be deleted while clones exist):

```bash
rbd snap protect rbd/vm-disk-01@snap-2025-03-01
```

Clone from a protected snapshot (fast — uses copy-on-write):

```bash
rbd clone rbd/vm-disk-01@snap-2025-03-01 rbd/vm-disk-02
```

Flatten the clone to make it independent of the parent snapshot (needed before deleting the parent):

```bash
rbd flatten rbd/vm-disk-02
```

Remove a snapshot:

```bash
rbd snap rm rbd/vm-disk-01@snap-2025-03-01
```

Remove all snapshots from an image:

```bash
rbd snap purge rbd/vm-disk-01
```

---

## Delete an Image

> **WARNING:** `rbd rm` permanently destroys the image and all data in it. There is no undo. Ensure the image is unmapped and unmounted, and that no snapshots exist (or use `rbd rm --no-progress` on an image with no snapshots).

```bash
# Verify nothing is mapped
rbd showmapped

# Remove the image
rbd rm rbd/vm-disk-01
```

To defer deletion (move to trash for later removal):

```bash
rbd trash mv rbd/vm-disk-01
rbd trash ls rbd
```

```
b1c2d3e4f5a6  vm-disk-01  user   Mon Mar  1 10:00:00 2025  Mon Mar  1 11:00:00 2025
```

Restore from trash:

```bash
rbd trash restore rbd/b1c2d3e4f5a6
```

Permanently remove from trash:

```bash
rbd trash rm rbd/b1c2d3e4f5a6
```

---

## Verification Commands

```bash
# List all images in pool
rbd ls rbd

# Show image details
rbd info rbd/vm-disk-01

# Show image usage (actual allocated space, not provisioned size)
rbd du rbd/vm-disk-01

# Show all current kernel driver mappings on this host
rbd showmapped

# Show all snapshots of an image
rbd snap ls rbd/vm-disk-01

# Show RBD pool-level stats
rbd pool stats rbd

# Watch live I/O stats for a mapped image
rbd perf image iostat rbd
```

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/rbd/rados-rbd-cmds.rst
  https://github.com/ceph/ceph/blob/main/doc/rbd/rbd-config-ref.rst
  https://github.com/ceph/ceph/blob/main/doc/man/8/rbd.rst
-->
