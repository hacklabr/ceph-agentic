# BlueStore OSD Tuning

## Key parameters

| Parameter | HDD default | SSD/NVMe default | Notes |
|---|---|---|---|
| `bluestore_cache_size_hdd` | 1 GiB | — | Cache budget per OSD on HDD primary device |
| `bluestore_cache_size_ssd` | — | 3 GiB | Cache budget per OSD on SSD/NVMe primary device |
| `bluestore_cache_autotune` | true | true | Automatically resizes cache within `osd_memory_target` |
| `osd_memory_target` | 4 GiB | 4 GiB | Target OSD heap usage; drives auto-cache sizing |
| `bluestore_cache_meta_ratio` | 0.01 | 0.01 | Fraction of cache for BlueStore metadata |
| `bluestore_cache_kv_ratio` | 0.99 | 0.99 | Fraction of cache for RocksDB KV data |
| `bluestore_throttle_bytes` | 128 MiB | 128 MiB | In-flight write bytes before throttling |
| `bluestore_throttle_deferred_bytes` | 128 MiB | 128 MiB | Deferred-write bytes before blocking |
| `bluestore_throttle_cost_per_io_hdd` | 670000 | — | Per-IO cost weight for throttle on HDD |
| `bluestore_throttle_cost_per_io_ssd` | — | 4000 | Per-IO cost weight for throttle on SSD |
| `bluestore_csum_type` | crc32c | crc32c | Checksum algorithm; options: none, crc32c, crc32c_16, crc32c_8, xxhash32, xxhash64 |
| `bluestore_compression_mode` | none | none | none / passive / aggressive / force |

## WAL and DB on fast media

BlueStore has three logical devices:
- **block** — primary data device (usually the HDD or NVMe being deployed)
- **block.db** — RocksDB metadata; separating onto faster media improves metadata operations
- **block.wal** — write-ahead log; placing on fast media reduces write latency

### Decision rules

| Available fast media | Recommendation |
|---|---|
| < 1 GiB fast space | Use as WAL device only |
| ≥ 1 GiB fast space | Use as DB device (WAL is implicitly colocated with DB) |
| All same speed (all HDD or all NVMe) | Do not separate; let BlueStore colocate within block |

### DB sizing guidance

| Workload | Recommended block.db size as % of block |
|---|---|
| RBD | 1–2% |
| CephFS general | 1–2% |
| RGW (many small objects / heavy omap) | ≥ 4% |
| Mixed or unknown | 2–4% |

For a 1 TB HDD OSD with an RGW workload, provision at least 40 GB of DB on SSD.

### Provisioning a split OSD (4 HDDs + 1 SSD example)

```bash
# Create VGs for the four HDD data devices
vgcreate ceph-block-0 /dev/sda
vgcreate ceph-block-1 /dev/sdb
vgcreate ceph-block-2 /dev/sdc
vgcreate ceph-block-3 /dev/sdd

# Create block (data) LVs
lvcreate -l 100%FREE -n block-0 ceph-block-0
lvcreate -l 100%FREE -n block-1 ceph-block-1
lvcreate -l 100%FREE -n block-2 ceph-block-2
lvcreate -l 100%FREE -n block-3 ceph-block-3

# Carve DB LVs from the SSD (200 GB SSD, 4 OSDs — 50 GB each)
vgcreate ceph-db-0 /dev/sdx
lvcreate -L 50GB -n db-0 ceph-db-0
lvcreate -L 50GB -n db-1 ceph-db-0
lvcreate -L 50GB -n db-2 ceph-db-0
lvcreate -L 50GB -n db-3 ceph-db-0

# Deploy the four OSDs
ceph-volume lvm create --bluestore --data ceph-block-0/block-0 --block.db ceph-db-0/db-0
ceph-volume lvm create --bluestore --data ceph-block-1/block-1 --block.db ceph-db-0/db-1
ceph-volume lvm create --bluestore --data ceph-block-2/block-2 --block.db ceph-db-0/db-2
ceph-volume lvm create --bluestore --data ceph-block-3/block-3 --block.db ceph-db-0/db-3
```

### Provisioning a single-device OSD (colocated)

```bash
ceph-volume lvm create --bluestore --data /dev/sda
```

## Recommendations by workload

### HDD-heavy cluster (bulk storage, archival)

- Increase `osd_memory_target` if RAM allows (8 GiB+ per OSD improves RocksDB hit rate)
- Enable compression for cold data: `bluestore_compression_mode = passive`
  — workloads that set a "compressible" hint will be compressed; others will not
- Keep `bluestore_cache_size_hdd` at default (1 GiB) unless RAM is abundant
- Add a small SSD (10–40 GB per OSD) as DB device to avoid metadata landing on the slow HDD

```ini
# ceph.conf [osd] section
[osd]
osd_memory_target = 8589934592        # 8 GiB
bluestore_compression_mode = passive
bluestore_compression_algorithm = snappy
```

### All-SSD / NVMe cluster (latency-sensitive)

- Increase `osd_memory_target` to 4–8 GiB per OSD
- No need to separate WAL/DB (same device class)
- Disable compression unless space is constrained (CPU overhead is measurable at high IOPS)
- Enable RocksDB sharding if OSDs were deployed on Pacific or later

```ini
[osd]
osd_memory_target = 8589934592        # 8 GiB
bluestore_compression_mode = none
```

### RGW workload (many small objects)

- Prioritise DB device size (≥ 4% of data device) — RGW omap metadata fills RocksDB fast
- Consider `aggressive` compression for cold buckets to recover omap space
- Monitor `ceph osd perf` for high `apply_latency_ms`; DB spillback to HDD shows here

```ini
[osd]
osd_memory_target = 8589934592
bluestore_compression_mode = aggressive
bluestore_compression_algorithm = lz4
```

### Mixed HDD + NVMe (tiered write cache)

- Deploy NVMe as DB-only devices; do not create separate WAL — DB placement covers it
- Verify DB fit: run `ceph osd df` and watch `%USE` on DB VG
- If DB becomes full, metadata spills to HDD — monitor with `ceph-bluestore-tool bluefs-stats`

## Applying configuration changes

### Global change (all OSDs)

```bash
# Apply via central config (preferred — no ceph.conf edits needed)
ceph config set osd osd_memory_target 8589934592
ceph config set osd bluestore_compression_mode passive

# Verify
ceph config get osd osd_memory_target
```

### Per-OSD override

```bash
# Override for a specific OSD
ceph config set osd.5 osd_memory_target 4294967296

# Verify
ceph config get osd.5 osd_memory_target
```

### Per-pool compression

```bash
# Set compression on a specific pool (overrides global bluestore setting)
ceph osd pool set mypool compression_algorithm snappy
ceph osd pool set mypool compression_mode aggressive
ceph osd pool set mypool compression_required_ratio 0.85

# Disable compression on a pool
ceph osd pool set mypool compression_mode none
```

### Changes that require OSD restart

These parameters are read at startup and require an OSD restart to take effect:
- `bluestore_cache_size_hdd` / `bluestore_cache_size_ssd`
- `osd_memory_target` (when `bluestore_cache_autotune` is false)

```bash
# Rolling restart via cephadm (one OSD at a time)
ceph orch daemon restart osd.5
# Or for all OSDs on a host:
ceph orch host maintenance enter <hostname>
ceph orch daemon restart osd   # restarts only OSDs on that host
ceph orch host maintenance exit <hostname>
```

### Verify BlueStore cache and device layout

```bash
# Show device layout for all OSDs
ceph osd metadata | jq '.[] | {id: .id, bluestore_bdev_type, bluefs_db_type}'

# Show BlueStore stats per OSD (latency, cache hit rates)
ceph osd perf

# Inspect DB usage on a running OSD
ceph tell osd.0 bluestore stats

# Check RocksDB BluFS stats (run on the OSD host)
ceph-bluestore-tool bluefs-stats --path /var/lib/ceph/osd/ceph-0
```

<!-- source: https://docs.ceph.com/en/latest/rados/configuration/bluestore-config-ref/ — reviewed 2026-03-29 -->
