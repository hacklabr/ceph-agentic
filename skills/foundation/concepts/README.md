# Ceph Architecture

Ceph is a unified distributed storage system delivering **object, block, and file storage** from a single cluster. Its foundation is RADOS (Reliable Autonomic Distributed Object Store): a self-managing, self-healing object store built from commodity nodes. All higher-level interfaces — RBD (block), CephFS (file), RGW (S3/Swift object) — are built on top of RADOS.

The key architectural principle: clients compute data placement themselves using the CRUSH algorithm. There is no central lookup table, no proxy, no metadata bottleneck. Clients talk directly to OSDs.

---

## Understand Core Daemons

Every Ceph cluster runs these daemon types. Understand what each one does before touching anything else.

### MON — Monitor

**Role:** Maintains the authoritative cluster map — a collection of five sub-maps (Monitor map, OSD map, PG map, CRUSH map, MDS map). All daemons and clients fetch maps from MONs. MONs use the Paxos consensus algorithm to agree on cluster state.

**Count:** Always deploy an **odd number**: 3 for production, 5 for larger or multi-site clusters. Quorum requires a majority (`floor(n/2) + 1`). A 3-MON cluster tolerates 1 failure; a 5-MON cluster tolerates 2.

**Where:** Dedicated management nodes or co-located with other daemons on small clusters. MONs must have stable, low-latency connectivity to each other. Avoid co-locating with OSD-heavy nodes.

**Port:** 3300 (v2 msgr, default since Nautilus), 6789 (v1 msgr, legacy clients).

**Inspect:**
```bash
ceph mon stat
# e3: 3 mons at {ceph-mon01=[v2:10.0.0.1:3300/0,v1:10.0.0.1:6789/0],...}, election epoch 12, leader 0 ceph-mon01, quorum 0,1,2 ceph-mon01,ceph-mon02,ceph-mon03

ceph mon dump
# dumped monmap epoch 3
# epoch 3
# fsid 1234abcd-5678-ef90-1234-abcdef012345
# ...

ceph quorum_status -f json-pretty
```

### MGR — Manager

**Role:** Provides cluster metrics, dashboard, and REST API. Hosts Python-based modules: `balancer`, `pg_autoscaler`, `orchestrator`, `dashboard`, `prometheus`, `alerts`. MGR does not handle data I/O — it is purely a control/monitoring plane.

**Count:** 1 active + 1 standby minimum. The standby takes over within seconds if the active MGR fails. For production, deploy 2 MGRs (one per mon host is common).

**Where:** Same node as MON is acceptable. Needs access to the network for dashboard/Prometheus exposure.

**Port:** 8443 (dashboard HTTPS), 9283 (Prometheus metrics endpoint).

**Inspect:**
```bash
ceph mgr stat
# {"epoch": 14, "available": true, "active_name": "ceph-mon01.abcdef", "num_standby": 1}

ceph mgr module ls
# enabled modules: balancer, crash, dashboard, devicehealth, iostat, nfs, orchestrator, pg_autoscaler, prometheus, rbd_support, restful, status, telemetry, volumes
```

### OSD — Object Storage Daemon

**Role:** Stores actual data. One OSD daemon per physical drive (or LVM volume). Handles reads, writes, replication, erasure coding, scrubbing, recovery, and backfill. OSDs also peer with each other to detect and report failures to MONs.

**Count:** As many as you have drives. A minimum viable cluster needs 3 OSDs (for 3-replica pools). Production typically starts at 9+ OSDs across 3+ hosts.

**Where:** Every storage node. Each OSD is bound to exactly one physical device via LVM (BlueStore). CPU and RAM requirements scale with OSD count — plan for ~2–4 GB RAM per OSD.

**OSD states:**

| State | Meaning |
|-------|---------|
| `up + in` | Normal: serving data |
| `down + in` | Unreachable but still part of CRUSH; triggers recovery |
| `up + out` | Marked out (no longer owns PGs); reweight to 0 |
| `down + out` | Removed from cluster map; data already rebalanced away |

**Port:** 6800–7300 (dynamically assigned per OSD process).

**Inspect:**
```bash
ceph osd stat
# 12 osds: 12 up (since 2d), 12 in (since 2d); epoch: e45

ceph osd tree
# ID  CLASS  WEIGHT   TYPE NAME          STATUS  REWEIGHT  PRI-AFF
# -1         1.74600  root default
# -3         0.58200      host ceph-osd01
#  0    hdd  0.19400          osd.0          up   1.00000  1.00000
#  1    hdd  0.19400          osd.1          up   1.00000  1.00000
#  2    hdd  0.19400          osd.2          up   1.00000  1.00000
# -5         0.58200      host ceph-osd02
#  3    hdd  0.19400          osd.3          up   1.00000  1.00000
# ...

ceph osd df
# ID  CLASS  WEIGHT   REWEIGHT  SIZE     RAW USE  DATA     OMAP     META     AVAIL    %USE  VAR  PGS  STATUS
#  0    hdd  0.19400   1.00000  199 GiB  1.9 GiB  1.5 GiB  240 KiB  449 MiB  197 GiB  0.95  1.00   33  up
```

### MDS — Metadata Server

**Role:** Manages the CephFS filesystem namespace and directory metadata. MDS does **not** handle file data — data goes directly to RADOS data pools. MDS handles only inode, dentry, and directory operations.

**Count:** 1 active per CephFS filesystem is the baseline. Deploy standby MDS daemons for HA. Large deployments use multiple active MDS (multi-MDS) to shard the namespace.

**Where:** Dedicated nodes or co-located. MDS is CPU- and RAM-intensive for metadata-heavy workloads. Not needed if you are not using CephFS.

**Inspect:**
```bash
ceph fs status
# testfs - 2 clients
# ======
# RANK  STATE   MDS        ACTIVITY  DNS   INOS  DIRS  CAPS
#    0   active  ceph-mds01  Reqs:  0 /s    12     13    12     0
# STANDBY-REPLAY
# ceph-mds02(ceph-mds02)

ceph mds stat
```

### RGW — RADOS Gateway

**Role:** Provides S3-compatible and Swift-compatible object storage API on top of RADOS. RGW stores bucket metadata, object metadata, and object data as RADOS objects in dedicated pools. Each RGW instance is stateless — multiple instances can run behind a load balancer.

**Count:** 2+ for HA. Scale horizontally by adding more RGW instances.

**Where:** Dedicated gateway nodes or shared. Each RGW instance runs as a container (via cephadm) or as `radosgw` process.

**Port:** 80 (HTTP), 443 (HTTPS).

**Inspect:**
```bash
ceph orch ls --service-type rgw
# NAME                     PORTS  RUNNING  REFRESHED  AGE  PLACEMENT
# rgw.default              ?:80       2/2  4s ago     3d   count:2

radosgw-admin bucket list
radosgw-admin user list
```

---

## Understand Data Flow

Trace how a client write reaches disk. Every step matters for diagnosis.

```
Client                    MON                    OSD Primary            OSD Replica(s)
  |                        |                         |                        |
  |--- fetch cluster map ->|                         |                        |
  |<-- cluster map --------|                         |                        |
  |                                                  |                        |
  | (client computes: object → pool → PG → OSD set via CRUSH)               |
  |                                                  |                        |
  |--- write object -------------------------------->|                        |
  |                                            write to disk                  |
  |                                                  |--- replicate -------->|
  |                                                  |<-- ack ----------------|
  |<-- ack ------------------------------------------|                        |
```

**Step by step:**

1. **Client hashes the object name** into a 32-bit hash: `hash(object_name) % pg_num` → PG ID.
2. **CRUSH maps PG → OSD set**: given the PG ID and the CRUSH map, CRUSH deterministically selects the primary OSD and replica OSDs.
3. **Client connects directly to the primary OSD** — no proxy, no MON involvement for the data path.
4. **Primary OSD writes to its local BlueStore**, then simultaneously replicates to replica OSDs.
5. **All replicas acknowledge** to the primary. Primary acknowledges to the client.
6. **If an OSD is down**: CRUSH selects a different OSD. The cluster triggers recovery/backfill to restore replica count.

**Formula:**
```
PG_ID  = hash(pool_id + object_name) mod pg_num
OSD_set = CRUSH(PG_ID, crush_map, crush_rule)
```

**Inspect the path for a specific object:**
```bash
# Find which PG holds an object
ceph osd map <pool-name> <object-name>
# osdmap e45 pool 'rbd' (2) object 'rb.0.1234.0000000000000001' -> pg 2.a1b2c3d4 (2.14) -> up ([3,7,11], p3) acting ([3,7,11], p3)

# Decode: PG 2.14, up set is OSDs [3,7,11], primary is OSD 3
```

---

## Understand the CRUSH Algorithm

CRUSH (Controlled Replication Under Scalable Hashing) is the algorithm that answers: "Given this PG, which OSDs should store it?" CRUSH is a pseudo-random deterministic function — every client and every OSD can compute the same answer independently from the same CRUSH map, eliminating the need for any central lookup service.

### CRUSH Map Structure

The CRUSH map has two parts:

**1. Hierarchy (buckets):** A tree of physical infrastructure. Leaves are OSDs. Internal nodes are `host`, `rack`, `row`, `datacenter`, `root`, etc.

```
root default
├── host ceph-osd01
│   ├── osd.0  (weight 0.194 TiB)
│   ├── osd.1  (weight 0.194 TiB)
│   └── osd.2  (weight 0.194 TiB)
├── host ceph-osd02
│   ├── osd.3  (weight 0.194 TiB)
│   ├── osd.4  (weight 0.194 TiB)
│   └── osd.5  (weight 0.194 TiB)
└── host ceph-osd03
    ├── osd.6  (weight 0.194 TiB)
    ├── osd.7  (weight 0.194 TiB)
    └── osd.8  (weight 0.194 TiB)
```

**2. Rules:** Policies that say how to traverse the hierarchy to pick OSD targets.

A typical replicated rule reads: "Start at `root default`. Choose 3 items, with each item in a different `host`. Select one OSD from each chosen host."

### Failure Domains

The **failure domain** is the level at which CRUSH ensures replicas are separated. If the failure domain is `host`, no two replicas of the same object land on the same host. If it is `rack`, no two replicas share a rack.

| Failure Domain | Survives | Requires |
|----------------|---------|---------|
| `osd` | single drive failure | 3+ OSDs on ≥1 host |
| `host` | complete host loss | 3+ hosts |
| `rack` | entire rack loss | 3+ racks with OSDs |
| `datacenter` | DC-level outage | 3+ DCs |

**Rule: always set failure domain to `host` or higher for production.** The default OSD deployment uses `host` failure domain.

### Device Classes

OSDs are automatically classified as `hdd`, `ssd`, or `nvme`. Device classes create shadow CRUSH hierarchies, allowing pools to target specific media.

```bash
# View device classes in the tree
ceph osd tree
# Each OSD shows CLASS column: hdd, ssd, or nvme

# Create a rule targeting only SSDs
ceph osd crush rule create-replicated replicated-ssd default host ssd

# Apply to a pool
ceph osd pool set <pool-name> crush_rule replicated-ssd

# View shadow hierarchies (one per device class)
ceph osd crush tree --show-shadow
```

### Inspect and Edit CRUSH Maps

```bash
# View current CRUSH hierarchy
ceph osd tree

# List rules
ceph osd crush rule ls
# replicated_rule
# erasure-code

# Dump rules detail
ceph osd crush rule dump
# [{"rule_id": 0, "rule_name": "replicated_rule", "ruleset": 0, "type": 1,
#   "steps": [{"op": "take", "item_name": "default"}, {"op": "chooseleaf_firstn", "num": 0, "type": "host"}, {"op": "emit"}]}]

# Export and decompile CRUSH map for manual editing
ceph osd getcrushmap -o /tmp/crushmap.bin
crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt
# edit /tmp/crushmap.txt
crushtool -c /tmp/crushmap.txt -o /tmp/crushmap-new.bin
ceph osd setcrushmap -i /tmp/crushmap-new.bin

# Adjust OSD weight (0.0 removes it from placement)
ceph osd crush reweight osd.3 0.5

# Move an OSD to a different host bucket
ceph osd crush set osd.5 0.194 root=default host=ceph-osd04
```

**Decision: when to modify CRUSH:**
- New rack added → update CRUSH to reflect rack topology, set failure domain to `rack`
- Mixed HDD/SSD nodes → use device classes + per-class rules
- Uneven distribution → check weights; use `ceph osd reweight-by-utilization` or balancer module
- Never hand-edit CRUSH maps unless you understand the bucket algorithm — incorrect edits cause data movement

---

## Understand Placement Groups (PGs)

### What PGs Are

A Placement Group (PG) is an internal sharding unit. Objects are not placed directly onto OSDs — they are first mapped to a PG, and the PG is then mapped to an OSD set. This two-level mapping is what makes Ceph scalable: instead of tracking millions of objects individually, the cluster tracks thousands of PGs.

```
object "foo/bar.img"
    ↓ hash(pool_id + "foo/bar.img") mod pg_num
PG 2.3f
    ↓ CRUSH(PG 2.3f, crush_map)
OSDs [4, 8, 11]  ← primary=4, replicas=8,11
```

Every PG belongs to exactly one pool. Each pool has its own `pg_num`.

### PG States

A PG is healthy when `active+clean`. Any other state requires attention.

| State | Meaning | Action |
|-------|---------|--------|
| `active+clean` | Normal — all replicas present and synced | None |
| `active+clean+scrubbing` | Routine integrity check | None — expected |
| `active+clean+deep` | Deep scrub (full checksum verify) | None — expected |
| `active+degraded` | Replica missing; fewer copies than configured | Monitor recovery progress |
| `active+recovering` | Recovering lost replica(s) after OSD event | Wait — monitor with `ceph -w` |
| `active+backfilling` | Migrating PG data to a new OSD | Wait — backfill is normal after OSD add/remove |
| `active+remapped` | PG moved to different OSD set but not yet backfilled | Wait |
| `stale` | PG has not reported status recently | Investigate — OSD may be down |
| `undersized` | Fewer OSDs in acting set than pool `size` | Check OSD health |
| `incomplete` | Not enough replicas to determine correct state | Urgent — potential data loss |
| `down` | All replicas unavailable | Urgent — cluster may be unavailable |
| `peering` | OSDs negotiating PG state | Transient — investigate if persistent |

```bash
# Get summary PG status
ceph pg stat
# 513 pgs: 513 active+clean; 1.2 TiB data, 3.6 TiB used, 5.2 TiB avail

# List all PGs with non-clean states
ceph pg ls-by-pool <pool-name>

# Get PG detail
ceph pg <pgid> query
# e.g.: ceph pg 2.3f query

# Watch cluster events including PG transitions
ceph -w
```

### PG Count and the Autoscaler

**Rule of thumb for manual PG sizing:** Target ~100 PGs per OSD, distributed across pools by expected size. PG count must be a **power of 2**.

```
pg_num = (OSDs × 100) / pool_replica_size
# Round up to next power of 2
# Example: 12 OSDs, 3 replicas: (12 × 100) / 3 = 400 → use 512
```

Too few PGs → poor distribution, hotspots. Too many PGs → excess RAM/CPU overhead, slow peering.

**The PG autoscaler** (enabled by default since Nautilus) manages this automatically:

```bash
# Check autoscaler recommendations
ceph osd pool autoscale-status
# POOL    SIZE  TARGET SIZE  RATE  RAW CAPACITY   RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE  BULK
# rbd       1.2T               3.0        9.6T    0.3750                                  1.0      64        128  warn       False
# .mgr        0                3.0        9.6T    0.0000                                  1.0       1               on       False

# Enable autoscaling on a pool
ceph osd pool set <pool-name> pg_autoscale_mode on

# Disable autoscaling on a pool (manage manually)
ceph osd pool set <pool-name> pg_autoscale_mode off

# Set global default for new pools
ceph config set global osd_pool_default_pg_autoscale_mode on

# Set target PG count per OSD (default 100; recommend 200 for most clusters)
ceph config set global mon_target_pg_per_osd 200

# Mark a pool as "bulk" (starts with full PG allocation, scales down if uneven)
ceph osd pool set <pool-name> bulk true
# Use bulk for large RBD and RGW data pools
```

**`pgp_num`** controls actual PG placement (how PGs map to OSDs). Always set equal to `pg_num`. The autoscaler manages both together. When changing manually:

```bash
# Change pg_num first, then pgp_num after PGs finish creating
ceph osd pool set <pool-name> pg_num 128
ceph osd pool set <pool-name> pgp_num 128
```

**Decision tree for PG issues:**
- `HEALTH_WARN too many PGs` → reduce `pg_num` or let autoscaler handle it; check `mon_target_pg_per_osd`
- `HEALTH_WARN too few PGs` → increase `pg_num` or set autoscale to `on`
- PGs stuck in `peering` → check if enough OSDs are `up+in` for quorum on those PGs
- PGs stuck `inactive` → likely a quorum or OSD availability problem; check `ceph osd stat`

---

## Understand Pools

A pool is the logical namespace for storing data. Every RBD image, CephFS filesystem, RGW bucket, or custom application maps to one or more pools.

### Replicated Pools

Each object is stored as N identical copies on N different OSDs (where N = `size`, default 3). Writes go to the primary OSD, which replicates synchronously.

```
size = 3   → 3 full copies; tolerates loss of 2 OSDs (but writes pause below min_size)
min_size = 2  → cluster serves I/O as long as 2 copies are available
```

**Space cost:** 3× raw storage per unit of data.

```bash
# Create a replicated pool
ceph osd pool create rbd 128 128 replicated replicated_rule

# Create pool with autoscaling enabled
ceph osd pool create my-pool --autoscale-mode on

# Set pool parameters
ceph osd pool set rbd size 3
ceph osd pool set rbd min_size 2

# Tag pool for an application (enables application-specific features)
ceph osd pool application enable rbd rbd
# Applications: rbd, cephfs, rgw, openstack
```

### Erasure-Coded Pools

Data is split into K data chunks and M coding chunks. Any K chunks can reconstruct the full object. Total chunks stored = K + M.

**Common profiles:**

| Profile | Chunks | Overhead | Min hosts | Notes |
|---------|--------|---------|-----------|-------|
| `k=2,m=1` | 3 total | 1.5× | 3 | Minimum EC; like RAID-5 |
| `k=4,m=2` | 6 total | 1.5× | 6 | Production standard; tolerates 2 failures |
| `k=8,m=3` | 11 total | 1.375× | 11 | High efficiency; large clusters |
| `k=4,m=1` | 5 total | 1.25× | 5 | Lower durability — avoid |

**Space cost:** For `k=4,m=2`, overhead is `(K+M)/K = 6/4 = 1.5×` — only 1.5× raw vs 3× for replicated.

```bash
# Create erasure code profile
ceph osd erasure-code-profile set my-ec k=4 m=2 crush-failure-domain=host

# View profiles
ceph osd erasure-code-profile ls
ceph osd erasure-code-profile get my-ec
# {"crush-failure-domain": "host", "crush-root": "default", "jerasure-per-chunk-alignment": "false",
#  "k": "4", "m": "2", "plugin": "jerasure", "technique": "reed_sol_van", "w": "8"}

# Create pool with EC profile
ceph osd pool create ecpool 128 128 erasure my-ec
ceph osd pool application enable ecpool rgw
```

**EC limitations:** EC pools do not support RBD or CephFS data directly. RBD uses replicated pools. CephFS uses a replicated metadata pool and can use EC for its data pool. RGW can use EC for its data pool.

### Pool Inspection

```bash
# List all pools
ceph osd lspools
# 1 .mgr
# 2 rbd
# 3 cephfs.testfs.meta
# 4 cephfs.testfs.data

# Detailed pool listing
ceph osd pool ls detail
# pool 2 'rbd' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 128 pgp_num 128
#   pg_num_target 128 pgp_num_target 128 autoscale_mode warn last_change 45 flags hashpspool,selfmanaged_snaps
#   stripe_width 0 application rbd

# Pool usage statistics
ceph df
# --- RAW STORAGE ---
# CLASS    SIZE     AVAIL    USED     RAW USED  %RAW USED
# hdd      9.6 TiB  9.5 TiB  1.4 TiB   1.4 TiB       14.41
# TOTAL    9.6 TiB  9.5 TiB  1.4 TiB   1.4 TiB       14.41
#
# --- POOLS ---
# POOL                   ID  PGS   STORED   OBJECTS  USED     %USED  MAX AVAIL
# rbd                     2  128  1.2 TiB   310.5k   3.5 TiB  36.89  2.2 TiB

# Delete a pool (requires confirmation flags)
ceph osd pool delete <pool-name> <pool-name> --yes-i-really-really-mean-it
```

### Pool Parameters Reference

| Parameter | Default | Effect |
|-----------|---------|--------|
| `size` | 3 | Number of replicas |
| `min_size` | 2 | Minimum replicas for I/O |
| `pg_num` | varies | PG count (set by autoscaler) |
| `crush_rule` | `replicated_rule` | Which CRUSH rule governs placement |
| `pg_autoscale_mode` | `on` (Nautilus+) | `on`/`off`/`warn` |
| `bulk` | false | Bulk pool starts with full PG allocation |
| `compression_mode` | `none` | BlueStore inline compression |
| `target_size_ratio` | — | Hint for autoscaler: fraction of cluster this pool should use |
| `target_size_bytes` | — | Hint for autoscaler: absolute expected pool size |

---

## Understand BlueStore

BlueStore is the default OSD backend since Luminous (12.x). It replaces Filestore. BlueStore writes directly to raw block devices — it does not use a kernel filesystem (no ext4, no XFS). This eliminates the double-write penalty of Filestore.

### Storage Devices

BlueStore manages up to three block devices per OSD:

| Device | Symlink | Role |
|--------|---------|------|
| Primary (data) | `block` | Stores object data; always present |
| DB device | `block.db` | RocksDB metadata (omap keys, internal state); optional |
| WAL device | `block.wal` | Write-ahead log; optional; collocated with DB if DB present |

**When to separate DB and WAL:**
- If you have fast media (NVMe/SSD) and slow media (HDD): put `block` on HDD, `block.db` on NVMe.
- If only fast media: no separation needed — BlueStore colocates everything on `block`.
- WAL alone on fast media provides minimal benefit vs DB device. If you have space, always use a DB device over WAL-only.

**DB device sizing:**
- RBD workloads: 1–2% of `block` size (e.g., 200 GiB HDD → 2–4 GiB DB)
- RGW workloads: 4%+ of `block` size (RGW stores heavy omap metadata in DB)
- A 200 GiB SSD split as DB across 4 × 2 TiB HDDs → 50 GiB per OSD DB device (sufficient)

**Provision a BlueStore OSD:**
```bash
# Single device (all colocated)
ceph-volume lvm create --bluestore --data /dev/sdb

# With separate DB device
ceph-volume lvm create --bluestore --data /dev/sdb --block.db /dev/nvme0n1p1

# With cephadm (recommended)
ceph orch daemon add osd <host>:/dev/sdb
```

### Memory and Cache

BlueStore automatically sizes its cache using TCMalloc and `osd_memory_target` (default: 4 GiB per OSD). Cache is split between RocksDB KV cache, BlueStore metadata, and recently accessed object data.

```bash
# Check current OSD memory target
ceph config get osd.0 osd_memory_target
# 4294967296  (4 GiB)

# Adjust memory target (NVMe OSDs can handle less; HDD OSDs benefit from more)
ceph config set osd osd_memory_target 6442450944   # 6 GiB

# View BlueStore cache allocation
ceph daemon osd.0 bluestore allocator dump block
ceph daemon osd.0 perf dump | grep bluestore_cache
```

### Checksums and Compression

BlueStore checksums all data and metadata by default using `crc32c`.

```bash
# Enable compression on a pool
ceph osd pool set <pool-name> compression_algorithm snappy
ceph osd pool set <pool-name> compression_mode passive
# compression_mode options: none, passive, aggressive, force
# passive: compress only if client hints it's compressible
# aggressive: compress unless client hints it's incompressible

# View OSD BlueStore metadata (shows min_alloc_size, backend)
ceph osd metadata osd.0
```

### Key Tuning Concepts

| Concept | Default | Notes |
|---------|---------|-------|
| `bluestore_min_alloc_size_hdd` | 4096 (4 KB) | Set at OSD creation; do not change after |
| `bluestore_min_alloc_size_ssd` | 4096 (4 KB) | Set at OSD creation; 4 KB aligns with modern SSDs |
| `osd_memory_target` | 4 GiB | Per-OSD heap target; increase for NVMe |
| `bluestore_cache_autotune` | true | Auto-sizes KV/meta/data cache split |
| `bluestore_cache_size_hdd` | 1 GiB | Cache budget for HDD OSDs |
| `bluestore_cache_size_ssd` | 3 GiB | Cache budget for SSD OSDs |

**Inspect BlueStore OSD state:**
```bash
ceph osd metadata osd.0 | python3 -m json.tool | grep -E "bluestore|osd_objectstore|rotational"
# "bluestore_bdev_rotational": "1",
# "bluestore_min_alloc_size": "4096",
# "osd_objectstore": "bluestore"
```

---

## Understand Ceph Versions

Ceph uses alphabetical release names. Each major release is a named series with point releases (`X.2.Y`). The version format is `MAJOR.MINOR.PATCH`:
- **Even MINOR = stable** (`18.2.x`, `19.2.x`)
- **Odd MINOR = development/release candidate** (`18.1.x`, `19.1.x`)

### Current Supported Releases

| Release | Version | Status | EOL |
|---------|---------|--------|-----|
| **Reef** | 18.2.x | Active LTS | ~2025 |
| **Squid** | 19.2.x | Active | ~2026 |
| Quincy | 17.2.x | Maintenance | 2024 |
| Pacific | 16.2.x | EOL | 2023 |

**Reef (18.2.x)** — Stable LTS release. cephadm orchestration mature. RGW multisite improvements. Recommended for new production deployments that require stability.

**Squid (19.2.x)** — Current stable release (as of late 2024). Improvements to crimson OSD (experimental), RGW, CephFS. Use for greenfield deployments or when you need latest features. Reef → Squid upgrade path is supported.

### Naming Convention

```
Argonaut → Bobtail → Cuttlefish → Dumpling → Emperor → Firefly →
Giant → Hammer → Infernalis → Jewel → Kraken → Luminous →
Mimic → Nautilus → Octopus → Pacific → Quincy → Reef → Squid →
Tentacle (next)
```

### Check Cluster Version

```bash
ceph version
# ceph version 18.2.4 (abc123...) reef (stable)

ceph versions
# {
#   "mon": {"ceph version 18.2.4 (abc123...) reef (stable)": 3},
#   "mgr": {"ceph version 18.2.4 (abc123...) reef (stable)": 2},
#   "osd": {"ceph version 18.2.4 (abc123...) reef (stable)": 12},
#   "mds": {},
#   "overall": {"ceph version 18.2.4 (abc123...) reef (stable)": 17}
# }
# All daemons should show the same version during normal operation.
# Mixed versions are expected only during rolling upgrades.
```

**Upgrade rule:** Always upgrade one major version at a time. Reef → Squid is supported. You cannot jump from Pacific directly to Squid without going through Quincy then Reef.

```bash
# Check upgrade readiness
ceph versions           # verify all daemons at same version before starting
ceph health detail      # resolve all HEALTH_ERR and non-trivial HEALTH_WARN before upgrading
ceph osd ok-to-stop <osd-id>  # verify safe to stop individual daemons during upgrade
```

---

<!-- Source: doc/architecture.rst, doc/rados/operations/crush-map.rst, doc/rados/operations/placement-groups.rst, doc/rados/configuration/bluestore-config-ref.rst -->
