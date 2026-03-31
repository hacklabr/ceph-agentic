# Pool Management

Pools are the logical partitions where Ceph stores RADOS objects. Every pool has a type (replicated or erasure-coded), a CRUSH rule that governs placement, and a PG count managed either manually or by the autoscaler. This guide covers the full lifecycle: creation, tuning, CRUSH rule management, and safe deletion.

**When to use this guide:** creating pools for new applications, tuning performance or capacity, adjusting failure domains, configuring erasure coding, or safely removing a pool.

---

## List and Inspect Pools

List pool names only (useful in scripts):

```bash
ceph osd pool ls
```

List pools with pool numbers:

```bash
ceph osd lspools
```

List pools with full detail (type, size, PG count, CRUSH rule, autoscale mode, flags):

```bash
ceph osd pool ls detail
```

Example output:

```
pool 1 'rbd' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 128 pgp_num 128 autoscale_mode on last_change 19 flags hashpspool stripe_width 0 application rbd
```

Inspect a single pool attribute:

```bash
ceph osd pool get <pool-name> <key>
# Examples:
ceph osd pool get rbd size
ceph osd pool get rbd crush_rule
ceph osd pool get rbd pg_num
```

Show utilization statistics across all pools:

```bash
rados df
```

Show I/O statistics for all pools or a specific pool:

```bash
ceph osd pool stats
ceph osd pool stats <pool-name>
```

---

## Create a Replicated Pool

Replicated pools store each object as N copies across distinct failure domains. This is the correct choice for RBD metadata, CephFS metadata, and any workload requiring low-latency reads or writes.

Create a replicated pool with the PG autoscaler managing PG count:

```bash
ceph osd pool create <pool-name> replicated
```

Create a replicated pool targeting a specific device class and failure domain (requires a pre-existing CRUSH rule — see the CRUSH Rules section):

```bash
# First create the rule (if it does not already exist):
ceph osd crush rule create-replicated ssd-rule default host ssd

# Then create the pool referencing that rule:
ceph osd pool create fast-rbd replicated ssd-rule
```

Set replication factor and minimum replicas required for I/O:

```bash
ceph osd pool set <pool-name> size 3
ceph osd pool set <pool-name> min_size 2
```

Associate the pool with its application (required before first use):

```bash
# For RBD:
rbd pool init <pool-name>
# or manually:
ceph osd pool application enable <pool-name> rbd

# For CephFS (set automatically by `ceph fs` commands):
ceph osd pool application enable <pool-name> cephfs

# For RGW (set automatically by radosgw-admin):
ceph osd pool application enable <pool-name> rgw
```

Enable inline compression on a BlueStore pool (optional):

```bash
ceph osd pool set <pool-name> compression_algorithm snappy
ceph osd pool set <pool-name> compression_mode aggressive
```

Valid compression algorithms: `lz4`, `snappy`, `zlib`, `zstd`. Valid compression modes: `none`, `passive`, `aggressive`, `force`.

---

## Create an Erasure Coded Pool

Erasure-coded (EC) pools break objects into K data chunks and M coding (parity) chunks. The overhead factor is `(K+M)/K`. A `k=4 m=2` profile uses 1.5× storage versus the 3× of a `size=3` replicated pool, while tolerating 2 simultaneous OSD failures.

**Choosing K and M:** Use `k=4 m=2` as the default production choice. Do not exceed `k=4` or `m=2` without fully understanding the failure domain topology requirements. A profile with `m=1` means any single OSD failure during a second failure causes data loss — avoid in production.

### Step 1: Create an erasure-code profile

View the built-in default profile:

```bash
ceph osd erasure-code-profile get default
```

```
k=2
m=2
plugin=isa
crush-failure-domain=host
technique=reed_sol_van
```

Create a custom profile for a rack-level failure domain with 4+2 encoding:

```bash
ceph osd erasure-code-profile set my-ec-profile \
    k=4 \
    m=2 \
    crush-failure-domain=rack \
    crush-device-class=hdd \
    plugin=isa \
    technique=reed_sol_van
```

Verify the profile was created:

```bash
ceph osd erasure-code-profile ls
ceph osd erasure-code-profile get my-ec-profile
```

**The profile cannot be changed after pool creation.** If a different profile is needed, create a new pool and migrate objects.

### Step 2: Create the pool

```bash
ceph osd pool create ecpool erasure my-ec-profile
```

Ceph automatically creates a matching CRUSH rule from the profile. The pool is ready for RGW or other full-object-write workloads immediately.

### Step 3: Enable overwrites for RBD and CephFS

By default, EC pools only support full RADOS object writes. Enable partial writes to use the EC pool as an RBD data pool or CephFS data pool:

```bash
ceph osd pool set ecpool allow_ec_overwrites true
```

This requires all OSDs in the pool to be running BlueStore.

### Step 4 (optional): Enable EC optimizations (Tentacle and later)

Reduces padding and space amplification, improves small I/O performance. Cannot be disabled after enabling:

```bash
ceph osd pool set ecpool allow_ec_optimizations true
```

When using optimizations, set a stripe unit of at least 16K in the profile at creation time:

```bash
# Add to profile before pool creation:
ceph osd erasure-code-profile set my-optimized-profile \
    k=4 \
    m=2 \
    crush-failure-domain=host \
    plugin=isa \
    technique=reed_sol_van \
    stripe_unit=16384
```

### EC pool overhead table

| Profile | Overhead factor | Survives |
|---------|----------------|---------|
| k=2 m=1 | 1.5× | 1 OSD (avoid in production) |
| k=2 m=2 | 2.0× | 2 OSDs |
| k=3 m=2 | 1.67× | 2 OSDs, rack-level |
| k=4 m=2 | 1.5× | 2 OSDs (recommended) |
| k=8 m=3 | 1.375× | 3 OSDs (large clusters only) |

### Use an EC pool as an RBD data pool

```bash
# Metadata stays in a replicated pool; data goes into the EC pool:
rbd create --size 100G --data-pool ecpool replicatedpool/myimage
```

---

## PG Autoscaler

The PG autoscaler (enabled by default since Nautilus) monitors pool utilization and adjusts PG counts to keep approximately 100–150 PGs per OSD. Manual PG management is no longer needed for most pools.

Check autoscaler status for all pools:

```bash
ceph osd pool autoscale-status
```

Example output:

```
POOL     SIZE  TARGET SIZE  RATE  RAW CAPACITY  RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE  BULK
rbd      120G  -            3.0   10T           0.01   -             0.01             1.0   128     -           on         False
```

Set autoscale mode per pool:

```bash
# Let the cluster adjust PGs automatically:
ceph osd pool set <pool-name> pg_autoscale_mode on

# Warn only — cluster suggests but does not change:
ceph osd pool set <pool-name> pg_autoscale_mode warn

# Disable autoscaling, manage manually:
ceph osd pool set <pool-name> pg_autoscale_mode off
```

Set the default autoscale mode for all new pools:

```bash
ceph config set global osd_pool_default_pg_autoscale_mode on
```

Mark a pool as bulk (high-data-volume pools that should receive more PGs):

```bash
ceph osd pool set <pool-name> bulk true
```

Manually set PG count (only when autoscaler is disabled for the pool):

```bash
ceph osd pool set <pool-name> pg_num 256
# pgp_num tracks pg_num automatically from Nautilus onward
```

---

## CRUSH Rules

CRUSH rules govern how Ceph maps PGs to OSDs. Each pool references exactly one CRUSH rule. Rules specify three things: the root of the placement hierarchy, the failure domain type (host, rack, datacenter), and optionally a device class (hdd, ssd, nvme).

List existing rules:

```bash
ceph osd crush rule ls
ceph osd crush rule dump
```

Inspect the rule assigned to a pool:

```bash
ceph osd pool get <pool-name> crush_rule
```

View the OSD tree and device classes:

```bash
ceph osd tree
ceph osd crush tree --show-shadow
```

### Set or change a device class on OSDs

OSDs auto-detect their class (hdd, ssd, nvme) at startup. Override manually:

```bash
# Remove existing class first (class cannot be changed directly):
ceph osd crush rm-device-class osd.0 osd.1 osd.2

# Assign new class:
ceph osd crush set-device-class ssd osd.0 osd.1 osd.2
```

### Create a replicated CRUSH rule

```bash
# Syntax: create-replicated <rule-name> <root> <failure-domain> [<device-class>]

# Replicate across hosts (default):
ceph osd crush rule create-replicated replicated-host-rule default host

# Replicate across racks using only SSD OSDs:
ceph osd crush rule create-replicated ssd-rack-rule default rack ssd

# Replicate across datacenters:
ceph osd crush rule create-replicated dc-rule default datacenter
```

### Create an erasure CRUSH rule from a profile

```bash
ceph osd crush rule create-erasure ec-rack-rule my-ec-profile
```

Ceph creates this rule automatically when creating a pool with a profile — explicit creation is only needed when re-using a rule across multiple pools.

### Assign a rule to an existing pool

```bash
ceph osd pool set <pool-name> crush_rule <rule-name>
```

### Delete a CRUSH rule

Only rules not in use by any pool can be deleted:

```bash
# Check which pools use rule ID 5:
ceph osd dump | grep "^pool" | grep "crush_rule 5"

# Delete the rule if unused:
ceph osd crush rule rm <rule-name>
```

---

## CRUSH Map Management

The CRUSH map defines the complete cluster topology (devices, buckets, hierarchy, rules). Under normal operations, Ceph manages it automatically. Manual edits are required only for non-standard topologies (multi-datacenter, custom failure domains, or weight adjustments).

> **WARNING: Any change to the CRUSH map — adding buckets, moving OSDs, reweighting, changing rules — will trigger data rebalancing. Large clusters can rebalance petabytes of data over hours or days. Schedule CRUSH changes during maintenance windows. Always verify cluster health is HEALTH_OK before making changes, and pause further changes until rebalancing completes.**

### Inspect the CRUSH map

View the hierarchy (with weights):

```bash
ceph osd tree
```

View full CRUSH map in text form:

```bash
# Export binary map:
ceph osd getcrushmap -o /tmp/crushmap.bin

# Decompile to text:
crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt

cat /tmp/crushmap.txt
```

### Add a CRUSH bucket

Buckets are intermediate hierarchy nodes (rack, row, datacenter). They are created automatically when OSDs are added with a location. Create them manually when building the hierarchy before adding OSDs:

```bash
ceph osd crush add-bucket rack12 rack
ceph osd crush add-bucket dc1 datacenter
```

Move a bucket to a new parent:

```bash
ceph osd crush move rack12 datacenter=dc1
ceph osd crush move host01 rack=rack12
```

Rename a bucket:

```bash
ceph osd crush rename-bucket oldname newname
```

Remove an empty bucket:

```bash
# Bucket must contain no OSDs or child buckets:
ceph osd crush remove rack12
```

### Add or move an OSD in the CRUSH map

Under normal operations this is handled automatically. Use this command only when relocating an OSD to a different hierarchy position:

```bash
ceph osd crush set osd.0 1.0 root=default datacenter=dc1 rack=rack01 host=node01
```

The weight value (1.0) should equal the drive capacity in TiB.

Reweight an OSD (adjust its relative weight without moving it):

```bash
ceph osd crush reweight osd.0 2.0
```

### Compile and inject a modified CRUSH map

For changes not expressible via CLI commands, edit the text map directly:

```bash
# Export and decompile:
ceph osd getcrushmap -o /tmp/crushmap.bin
crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt

# Edit /tmp/crushmap.txt as needed, then recompile and inject:
crushtool -c /tmp/crushmap.txt -o /tmp/crushmap-new.bin
ceph osd setcrushmap -i /tmp/crushmap-new.bin
```

> **WARNING: Injecting a corrupt or incorrectly modified CRUSH map can make the cluster unable to place data, causing widespread HEALTH_ERR. Always test compiled maps with `crushtool --test` before injecting. Keep the original binary as a backup.**

Test a CRUSH rule against a compiled map before injecting:

```bash
crushtool -i /tmp/crushmap-new.bin --test --show-statistics \
    --rule 0 --num-rep 3 --min-x 0 --max-x 100
```

### Weight sets and the balancer module

The balancer module automatically adjusts CRUSH weight sets to achieve even data distribution. Enable it with:

```bash
ceph mgr module enable balancer
ceph balancer on
```

Check balancer status:

```bash
ceph balancer status
```

---

## Pool Quotas

Quotas cap the maximum bytes or maximum number of objects a pool can store. When a quota is reached, writes return EDQUOT errors.

Set a byte quota:

```bash
ceph osd pool set-quota <pool-name> max_bytes 1099511627776   # 1 TiB
```

Set an object count quota:

```bash
ceph osd pool set-quota <pool-name> max_objects 10000000
```

Set both at once:

```bash
ceph osd pool set-quota <pool-name> max_bytes 5497558138880 max_objects 50000000
```

Remove a quota (set to 0):

```bash
ceph osd pool set-quota <pool-name> max_bytes 0
ceph osd pool set-quota <pool-name> max_objects 0
```

View current quotas:

```bash
ceph osd pool get-quota <pool-name>
```

Example output:

```
quotas for pool 'rbd':
  max objects: 10000000 objects
  max bytes  : 1099511627776 bytes
```

---

## Delete a Pool

> **WARNING: Pool deletion is immediate and irreversible. All objects, snapshots, and associated data in the pool are permanently destroyed. Deletion cannot be undone. Before proceeding: confirm the pool name exactly, verify no application is writing to it, and ensure you have a backup or have confirmed data is not needed.**

### Step 1: Enable pool deletion on monitors

By default, monitors refuse pool deletion requests. Enable the flag:

```bash
ceph config set mon mon_allow_pool_delete true
```

### Step 2: Delete the pool

The pool name must be specified twice and the confirmation flag provided:

```bash
ceph osd pool delete <pool-name> <pool-name> --yes-i-really-really-mean-it
```

### Step 3: Re-disable pool deletion (recommended)

```bash
ceph config set mon mon_allow_pool_delete false
```

### Step 4: Clean up orphaned CRUSH rules

Check whether the pool's CRUSH rule is still in use:

```bash
# Get the rule name the pool was using:
ceph osd pool get <pool-name> crush_rule   # run before deletion

# After deletion, check if the rule is still referenced:
ceph osd dump | grep "^pool" | grep "crush_rule <rule-id>"

# Remove if unused:
ceph osd crush rule rm <rule-name>
```

### Step 5: Remove orphaned auth users

If any auth users had capabilities scoped to the deleted pool, remove them:

```bash
ceph auth ls | grep -C 5 <pool-name>
ceph auth del client.<username>
```

---

<!-- Sources: Ceph official documentation
     doc/rados/operations/pools.rst
     doc/rados/operations/crush-map.rst
     doc/rados/operations/erasure-code.rst
     Ceph main branch, retrieved 2026-03-29 -->
