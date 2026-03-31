# Replication vs Erasure Coding

## Context — read before creating any pool

**The pool type and erasure-code profile cannot be changed after pool creation.**
If you choose wrong, you must create a new pool and migrate all data.
The `crush-failure-domain` can be changed post-creation by swapping the CRUSH rule,
but the k/m values are permanent. Plan carefully.

## Feature comparison

| Feature | Replicated (default) | Erasure Coded |
|---|---|---|
| Storage overhead (size=3) | 3× | 1.5× (k=4 m=2) |
| Storage overhead (minimum durable) | 3× (size=3) | 2× (k=2 m=2) |
| Write performance | Full IOPS per replica write | Lower — parity computed on every write |
| Read performance | Read from any replica (full IOPS) | All k shards must respond (higher latency) |
| Partial overwrites | Yes, always | Requires `allow_ec_overwrites true` (BlueStore only) |
| RBD support | Yes | Yes — with `allow_ec_overwrites true`; metadata pool must be replicated |
| CephFS support | Yes | Yes (data pool only); metadata pool must be replicated |
| RGW support | Yes | Yes — best fit; large sequential objects ideal |
| omap support | Yes | No — metadata (omap) must live in a replicated pool |
| Recovery speed | Fast | Slower — reads k shards to reconstruct each lost shard |
| Minimum OSDs | 1 (size=1, not recommended) | k+m OSDs (e.g., 6 for k=4 m=2) |
| Minimum failure domains | size (typically 3 hosts) | k+m distinct failure domains |
| Complexity | Low | Medium-high |

## Overhead table (k+m / k)

| Profile | Overhead | Survives |
|---|---|---|
| k=2 m=1 | 1.5× | 1 OSD loss — **not recommended for production** |
| k=2 m=2 | 2.0× | 2 OSD losses |
| k=4 m=2 | 1.5× | 2 OSD losses — **recommended general-purpose EC** |
| k=3 m=2 | 1.67× | 2 OSD losses |
| k=8 m=3 | 1.375× | 3 OSD losses — requires 11 failure domains, high recovery cost |

## Recommendations by use case

### RBD (block storage)

**Use replicated pools** unless storage cost is a primary constraint.

Reason: RBD does small random writes constantly. EC adds parity-computation overhead
on every write and requires `allow_ec_overwrites true`. The replicated pool also
avoids the omap limitation (RBD index data uses omap).

If cost forces EC for RBD data:
```bash
# Create the required replicated pool for metadata
ceph osd pool create rbd-meta replicated
rbd pool init rbd-meta

# Create the EC pool for data
ceph osd erasure-code-profile set rbd-ec k=4 m=2 crush-failure-domain=host
ceph osd pool create rbd-data erasure rbd-ec
ceph osd pool set rbd-data allow_ec_overwrites true

# Create image using the split layout
rbd create --size 100G --data-pool rbd-data rbd-meta/myimage
```

### RGW (object storage)

**Use erasure coding** for the data pool; replicated pool for index/metadata.

Reason: RGW objects are large and written once, then read sequentially —
the ideal EC workload. EC roughly halves storage cost with minimal real-world
performance impact for this access pattern.

```bash
# Create EC profile — k=4 m=2 is the recommended starting point
ceph osd erasure-code-profile set rgw-ec k=4 m=2 crush-failure-domain=host
ceph osd pool create rgw.data erasure rgw-ec
ceph osd pool set rgw.data allow_ec_overwrites true

# Replicated pool for bucket index (heavy omap use)
ceph osd pool create rgw.index replicated
ceph osd pool application enable rgw.data rgw
ceph osd pool application enable rgw.index rgw
```

### CephFS

**Use replicated for metadata pool; EC is optional for data pool.**

Reason: The MDS metadata pool requires omap support — EC is incompatible.
The data pool can be EC if files are large and sequential. Small files or
random-access workloads favour replicated data pools.

```bash
# Replicated metadata pool (required)
ceph osd pool create cephfs.meta replicated

# EC data pool (optional, suitable for large-file workloads)
ceph osd erasure-code-profile set cephfs-ec k=4 m=2 crush-failure-domain=host
ceph osd pool create cephfs.data erasure cephfs-ec
ceph osd pool set cephfs.data allow_ec_overwrites true

ceph fs new myfs cephfs.meta cephfs.data
```

### Small clusters (fewer than 6 hosts)

**Use replicated pools.**

Reason: EC requires at least k+m distinct failure domains. With k=4 m=2
that means 6 hosts. Forcing EC onto fewer hosts defeats its durability model.
Replicated `size=3 min_size=2` across 3+ hosts is the correct choice.

```bash
ceph osd pool create mypool replicated
ceph osd pool set mypool size 3
ceph osd pool set mypool min_size 2
```

## Migration path (replicated → EC)

There is no in-place conversion. Migration requires a new pool and a full data copy.

```bash
# 1. Create the new EC pool
ceph osd erasure-code-profile set new-ec k=4 m=2 crush-failure-domain=host
ceph osd pool create new-ec-pool erasure new-ec
ceph osd pool set new-ec-pool allow_ec_overwrites true

# 2. Copy data using rados (for RADOS objects) or RGW sync tools
# For RADOS objects:
rados cppool old-replicated-pool new-ec-pool

# 3. Update application configuration to point to the new pool
# (RBD: update image, RGW: update zone config, CephFS: add new data pool)

# 4. Verify data integrity
rados ls -p new-ec-pool | head -20
ceph health detail

# 5. Delete old pool only after confirming all data is migrated and accessible
ceph osd pool delete old-replicated-pool old-replicated-pool --yes-i-really-really-mean-it
```

<!-- source: https://docs.ceph.com/en/latest/rados/operations/erasure-code/ — reviewed 2026-03-29 -->
