# Performance Diagnosis

Benchmarking, latency attribution, and tuning for Ceph clusters. Use this guide when clients report slowness, when you want a baseline before a hardware change, or when you suspect a specific subsystem is the bottleneck.

**Prerequisite:** Confirm the cluster is `HEALTH_OK` before benchmarking. Active recovery, scrubbing, or degraded PGs will contaminate results and slow client I/O.

---

## Quick Performance Check

Get a cluster-wide snapshot of current throughput and latency before digging deeper:

```bash
# Summary throughput and op rates across all pools
ceph osd pool stats
```

Expected output:
```
pool rbd id 1
  client io 125 MiB/s rd, 43 MiB/s wr, 2.11 kop/s rd, 890 op/s wr

pool cephfs_data id 2
  client io 0 B/s rd, 0 B/s wr, 0 op/s rd, 0 op/s wr
```

```bash
# Per-OSD apply and commit latency
ceph osd perf
```

Expected output:
```
osd fs commit_latency(ms) apply_latency(ms)
  0               2                 2
  1               3                 3
  2               4                 4
  3             142               142   ← investigate this OSD
```

Any OSD with `apply_latency_ms > 50` (HDD) or `> 10` (SSD/NVMe) warrants investigation. Jump to [Identify Slow OSDs](#identify-slow-osds).

```bash
# Watch the cluster summary in real time (refresh every 2 s):
watch -n 2 ceph status
# Look at: client io line under SERVICES; recovery io if any
```

---

## Benchmarking with rados bench

`rados bench` drives I/O directly against a RADOS pool, bypassing RBD and CephFS. It measures raw object storage performance.

**Create a dedicated benchmark pool to avoid polluting production data:**

```bash
ceph osd pool create bench 32
```

### Write benchmark (4 MB objects, 16 concurrent writers, 30 seconds)

```bash
rados bench -p bench 30 write --no-cleanup -b 4M -t 16
```

Expected output (HDD cluster, 12 OSDs, 3× replication):
```
Maintaining 16 concurrent writes of 4194304 bytes to objects of size 4194304 for up to 30 seconds or 0 objects
Total time run:         30.0012
Total writes made:      4089
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     544.983
Stddev Bandwidth:       44.253
Max bandwidth (MB/sec): 614.000
Min bandwidth (MB/sec): 448.000
Average Latency(s):     0.117
Stddev Latency(s):      0.039
Max latency(s):         0.514
Min latency(s):         0.040
Finished.
```

### Sequential read benchmark (read the written objects back in order)

```bash
rados bench -p bench 30 seq -t 16
```

### Random read benchmark

```bash
rados bench -p bench 30 rand -t 16
```

### Cleanup after benchmarking

Always remove benchmark objects when done:

```bash
rados bench -p bench 30 write --cleanup
# or manually:
rados -p bench cleanup
ceph osd pool delete bench bench --yes-i-really-really-mean-it
```

### Interpreting rados bench results

| Media Type | Expected Write BW | Expected Rand Read BW | Expected Write Latency |
|------------|-------------------|-----------------------|------------------------|
| HDD (7200 RPM), 12 OSDs, 3× rep | 400–600 MB/s | 1–2 GB/s | 80–150 ms |
| SSD (SATA), 12 OSDs, 3× rep | 1–2 GB/s | 3–5 GB/s | 5–15 ms |
| NVMe, 12 OSDs, 3× rep | 3–6 GB/s | 8–15 GB/s | 1–5 ms |

Rules of thumb:
- Sequential write BW ≈ `(raw device throughput × OSD count) / replication_factor`
- Write latency is dominated by the slowest OSD in the acting set
- If `Stddev Bandwidth` is high relative to average (> 20%), there is a single slow OSD dragging down the mean

---

## Benchmarking with fio

`fio` with the `rbd` ioengine measures RBD performance end-to-end, including client-side caching and the RBD kernel path.

### Install and prepare

```bash
# Install fio with librbd support:
apt-get install fio   # Debian/Ubuntu
dnf install fio       # RHEL/Rocky

# Create a dedicated RBD image for testing (do NOT use production images):
rbd create bench/fio-test --size 20G
```

### fio job file for 4K random write IOPS (NVMe-class target)

```ini
# /tmp/ceph-fio.ini
[global]
ioengine=rbd
clientname=admin
pool=bench
rbdname=fio-test
invalidate=0
rw=randwrite
bs=4k
numjobs=4
iodepth=64
runtime=60
time_based=1
group_reporting=1

[rbd-randwrite-4k]
```

Run it:

```bash
fio /tmp/ceph-fio.ini
```

Expected output snippet:
```
  write: IOPS=48.2k, BW=188MiB/s (197MB/s)(11.0GiB/60001msec)
    lat (usec): min=312, max=8942, avg=530.81, stdev=204.30
```

### fio job file for 128K sequential write throughput

```ini
[global]
ioengine=rbd
clientname=admin
pool=bench
rbdname=fio-test
invalidate=0
rw=write
bs=128k
numjobs=1
iodepth=16
runtime=60
time_based=1
group_reporting=1

[rbd-seqwrite-128k]
```

### Performance expectations by media type

| Metric | HDD OSDs | SATA SSD OSDs | NVMe OSDs |
|--------|----------|---------------|-----------|
| 4K rand write IOPS (RBD, 3× rep) | 500–2 000 | 20 000–60 000 | 80 000–200 000 |
| 128K seq write BW | 200–500 MB/s | 800–2 000 MB/s | 2–5 GB/s |
| 4K rand write latency (avg) | 10–50 ms | 1–5 ms | 0.2–1 ms |
| 4K rand read IOPS | 1 000–3 000 | 30 000–80 000 | 100 000–300 000 |

HDD clusters should use larger block sizes (64K+) for benchmarking; 4K random I/O with HDDs is nearly always disk-seek-limited and does not reflect the cluster's practical throughput for most workloads.

### Cleanup

```bash
rbd rm bench/fio-test
```

---

## Identify Slow OSDs

### Sort by apply latency

```bash
ceph osd perf | sort -k3 -n -r | head -10
```

The third column is `apply_latency_ms`. The highest values are at the top. Any OSD standing out significantly from its peers (e.g., 150 ms vs. 5 ms for all others) is a bottleneck.

### Investigate an outlier OSD (example: osd.7)

**Step 1 — Check in-flight operations:**

```bash
ceph daemon osd.7 dump_ops_in_flight
```

Output shows each operation currently executing with a timeline of events. Look for which event stage is consuming time:

```
{ "ops": [
    { "description": "osd_op ... [write 0~4194304]",
      "age": 1.234,
      "events": [
        { "event": "queued_for_pg",        "time": "2026-03-29 10:00:00.000" },
        { "event": "reached_pg",           "time": "2026-03-29 10:00:00.001" },
        { "event": "started",              "time": "2026-03-29 10:00:00.002" },
        { "event": "waiting for subops",   "time": "2026-03-29 10:00:00.950" }
      ]
    }
  ]
}
```

| Long delay between these events | Likely cause |
|---------------------------------|--------------|
| `queued_for_pg` → `reached_pg` | CPU / thread saturation |
| `started` → `op_commit` | Storage I/O is slow |
| `waiting for subops from [X]` | A replica OSD (osd.X) is slow |
| `commit_sent` → `applied` | Journal or WAL flush contention |

**Step 2 — Check historical operations:**

```bash
ceph daemon osd.7 dump_historic_ops | python3 -m json.tool | head -80
```

Patterns in completed ops reinforce whether I/O or CPU is the sustained bottleneck.

**Step 3 — Check device I/O on the OSD host:**

```bash
# Identify the backing device:
ceph osd metadata 7 | grep -E "osd_objectstore_dev|bluestore_bdev_dev_node"

# Real-time I/O stats (look at await and %util):
iostat -x 2 10 /dev/sdX
```

Thresholds:
- HDD: `await > 20 ms` or `%util = 100%` → disk saturated
- SSD: `await > 5 ms` → investigate firmware/kernel issues
- NVMe: `await > 1 ms` → suspect queue depth configuration

**Step 4 — Check SMART health:**

```bash
smartctl -a /dev/sdX | grep -E "Reallocated|Uncorrectable|Media_Wearout|Seek_Error|Runtime_Bad"
# Any non-zero count for Reallocated_Sector_Ct or Uncorrectable_Sector_Ct → plan drive replacement
```

---

## OSD Perf Counters

The admin socket exposes detailed per-OSD performance counters. Use these for deep investigation.

```bash
# Dump all perf counters for osd.7:
ceph daemon osd.7 perf dump
```

### Key counter groups and what to look for

**BlueStore I/O latency histograms** (`bluestore`):

```bash
ceph daemon osd.7 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
bs = d.get('bluestore', {})
for k in ['bluestore_onode_hits', 'bluestore_onode_misses',
          'bluestore_write_big', 'bluestore_write_small',
          'bluestore_write_big_deferred', 'bluestore_txc']:
    print(f'{k}: {bs.get(k, \"n/a\")}')
"
```

| Counter | Meaning | Action if high |
|---------|---------|----------------|
| `bluestore_write_small` | Writes smaller than `min_alloc_size` — go through WAL | Expected for small-object workloads |
| `bluestore_write_big_deferred` | Large writes that had to defer to WAL | Indicates I/O pressure; tune `prefer_deferred_size` |
| `bluestore_onode_misses` | Cache miss on object metadata | Increase `osd_memory_target` |
| `bluestore_throttle_deferred_count` | Ops waiting in deferred throttle queue | Storage device is sustaining writes slowly |

**Op queue depth** (`osd`):

```bash
ceph daemon osd.7 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
osd = d.get('osd', {})
for k in ['op_latency', 'op_process_latency', 'op_r_latency',
          'op_w_latency', 'op_wip']:
    print(f'{k}: {osd.get(k, \"n/a\")}')
"
```

`op_wip` (work in progress) consistently > 64 on an HDD OSD suggests the OSD is saturated and needs fewer PGs.

**RocksDB compaction** (under `bluestore`):

```bash
ceph daemon osd.7 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
bs = d.get('bluestore', {})
for k in ['kv_flush_lat', 'kv_commit_lat', 'kv_sync_lat']:
    v = bs.get(k, {})
    print(f'{k}: avgtime={v.get(\"avgtime\", \"n/a\")}')
"
```

High `kv_commit_lat` (> 50 ms) indicates RocksDB flush latency — the DB device is slow or the DB device is full and has spilled to the HDD.

---

## BlueStore Tuning

### osd_memory_target — primary memory knob

BlueStore autotunes its cache (onode cache, data cache, RocksDB block cache) to fit within this target. The default is 4 GB per OSD.

```bash
# Check current setting:
ceph config get osd.7 osd_memory_target

# Increase to 8 GB for all OSDs (recommended for NVMe or high-density nodes):
ceph config set osd osd_memory_target 8589934592   # 8 GiB

# Set per-OSD if hardware varies:
ceph config set osd.7 osd_memory_target 12884901888  # 12 GiB
```

Rule of thumb: Allocate `(total node RAM − 4 GB for OS) / OSD count`. For a 128 GB node with 8 OSDs: `(128 − 4) / 8 = 15.5 GB` per OSD.

### bluestore_cache_size — manual cache override

If `osd_memory_target` autotuning is not meeting latency goals, pin the cache size manually:

```bash
# For HDD OSDs (default is 1 GiB):
ceph config set osd bluestore_cache_size_hdd 2147483648   # 2 GiB

# For SSD/NVMe OSDs (default is 3 GiB):
ceph config set osd bluestore_cache_size_ssd 6442450944   # 6 GiB
```

Setting `bluestore_cache_size` directly overrides the device-class-specific values and disables autotuning.

### bluestore_min_alloc_size — allocation granularity

This is set at OSD creation time and cannot be changed without recreating the OSD. Check the current value:

```bash
ceph osd metadata 7 | grep min_alloc_size
# "bluestore_min_alloc_size": "4096"
```

Defaults (Quincy / Reef and later):
- HDD: 4 KiB (`bluestore_min_alloc_size_hdd`)
- SSD/NVMe: 4 KiB (`bluestore_min_alloc_size_ssd`)

For workloads with objects consistently larger than 64 KiB (e.g., RBD with 4 MB objects), increasing `min_alloc_size` at OSD creation reduces metadata overhead:

```bash
# Set before creating OSDs — applies to new OSDs only:
ceph config set osd bluestore_min_alloc_size_hdd 65536   # 64 KiB
```

For mixed QLC/SMR drives with an internal unit size of 16 KiB, match the device's optimal I/O size:

```bash
cat /sys/block/sdX/queue/optimal_io_size   # e.g., 16384
# Set min_alloc_size to match at OSD creation
```

### bluestore_prefer_deferred_size — control WAL bypass

Writes smaller than this value go through the WAL (deferred write path); writes larger bypass it for direct placement. Tuning this can improve write tail latency on mixed workloads.

```bash
# Default: 0 (disabled — BlueStore decides based on min_alloc_size)
# For HDD with many small writes (e.g., RGW small objects):
ceph config set osd bluestore_prefer_deferred_size_hdd 65536   # 64 KiB

# For SSD where WAL overhead hurts large writes:
ceph config set osd bluestore_prefer_deferred_size_ssd 0
```

### Summarised tuning table

| Parameter | HDD default | SSD default | Recommended for HDD workload | Recommended for NVMe workload |
|-----------|-------------|-------------|------------------------------|-------------------------------|
| `osd_memory_target` | 4 GiB | 4 GiB | 6–8 GiB | 8–16 GiB |
| `bluestore_cache_size_hdd` | 1 GiB | — | 2–4 GiB | — |
| `bluestore_cache_size_ssd` | — | 3 GiB | — | 6–12 GiB |
| `bluestore_min_alloc_size_hdd` | 4 KiB | — | 4–64 KiB (workload dependent) | — |
| `bluestore_min_alloc_size_ssd` | — | 4 KiB | — | 4–16 KiB |
| `bluestore_prefer_deferred_size_hdd` | 0 | — | 32–64 KiB (small objects) | — |

Apply changes and verify they took effect without a restart:

```bash
ceph config show osd.7 | grep -E "memory_target|cache_size|min_alloc|prefer_deferred"
```

Most BlueStore cache parameters take effect immediately. `min_alloc_size` requires OSD recreation.

---

## Network Performance

Network issues present as high latency across all OSDs, not a single outlier. Test before concluding the storage is the problem.

### iperf3 bandwidth test between OSD hosts

Run the server on one OSD node and the client on another. Test both the public network and the cluster network separately.

```bash
# On ceph-osd01 (server):
iperf3 -s -B <cluster-network-ip>

# On ceph-osd02 (client):
iperf3 -c <ceph-osd01-cluster-ip> -t 30 -P 4
```

Expected output on a 10 GbE cluster network:
```
[SUM]  0.00-30.00 sec  33.4 GBytes  9.55 Gbits/sec   0  sender
[SUM]  0.00-30.00 sec  33.4 GBytes  9.55 Gbits/sec      receiver
```

Anything below 80% of the nominal link rate warrants investigation.

### MTU verification (jumbo frames)

If the cluster network is configured with jumbo frames (MTU 9000), verify end-to-end:

```bash
# On an OSD host, ping another OSD host with a 8972-byte payload (9000 − 28 headers):
ping -M do -s 8972 <cluster-ip-of-peer>
# If this fails with "Message too long" → a switch or NIC in the path has a lower MTU
```

Verify the NIC MTU:

```bash
ip link show <cluster-iface> | grep mtu
# Should show: mtu 9000
# If not: ip link set <cluster-iface> mtu 9000
```

### Check for packet loss and errors

```bash
# Run on each OSD host:
ip -s link show <cluster-iface>
# Look for: RX errors, TX errors, RX dropped, TX dropped > 0

ethtool -S <cluster-iface> | grep -v ": 0" | grep -v "^$"
# Any non-zero counter for crc_error, rx_missed, tx_timeout is significant
```

### Check for nf_conntrack exhaustion

Connection-tracking overflow causes intermittent TCP resets that manifest as OSD flapping and client timeouts:

```bash
dmesg | grep "nf_conntrack: table full"
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# If count is close to max, increase the limit:
sysctl -w net.netfilter.nf_conntrack_max=524288
echo "net.netfilter.nf_conntrack_max = 524288" >> /etc/sysctl.d/99-ceph.conf
```

---

## Common Bottlenecks

Work through this decision tree when a client reports high latency or low throughput.

### Step 1 — Is it a single OSD or cluster-wide?

```bash
ceph osd perf | sort -k3 -n -r | head -5
```

- **One OSD has outlier latency** → go to [disk bottleneck](#disk-bottleneck)
- **All OSDs have elevated latency** → go to [Step 2 (network or PG)](#step-2--is-the-cluster-network-saturated)

---

### Step 2 — Is the cluster network saturated?

```bash
# On all OSD hosts simultaneously (tmux or parallel-ssh):
sar -n DEV 1 5 | grep <cluster-iface>
# Look for rxkB/s and txkB/s approaching interface capacity
```

- **Interface near 100%** → add a second cluster network interface or upgrade link speed
- **Interface has headroom** → go to [Step 3 (PG pressure)](#step-3--are-too-many-pgs-per-osd-causing-cpu-pressure)

---

### Step 3 — Are too many PGs per OSD causing CPU pressure?

```bash
ceph osd df | awk 'NR>1 {sum += $NF; count++} END {print "avg PG/OSD:", sum/count}'
# Healthy range: 50–200 PGs per OSD
# > 300 PGs/OSD → CPU scheduling overhead

# Check CPU saturation on OSD hosts:
top -bn1 | grep ceph-osd
```

- **High PG count and CPU > 90%** → merge pools or reduce `pg_num`:
  ```bash
  ceph osd pool set <pool> pg_num_min 32
  ceph osd pool set <pool> pg_num 64
  ```
- **Normal PG count** → go to [Step 4 (recovery competition)](#step-4--is-recovery-or-scrubbing-competing-with-client-io)

---

### Step 4 — Is recovery or scrubbing competing with client I/O?

```bash
ceph status | grep -E "recovery|scrub|backfill"
# recovery: 500 MiB/s, 250 keys/s — this will impact client I/O
```

- **Active recovery** → throttle it:
  ```bash
  ceph config set osd osd_recovery_max_active_hdd 1
  ceph config set osd osd_max_backfills 1
  # Optionally pause recovery entirely:
  ceph osd set norecover
  ```
- **Active scrubbing** → defer scrub windows:
  ```bash
  ceph config set osd osd_scrub_begin_hour 22
  ceph config set osd osd_scrub_end_hour 6
  ```
- **No recovery or scrub** → go to [disk bottleneck](#disk-bottleneck)

---

### Disk bottleneck

One OSD or all OSDs on a node show high `apply_latency_ms` and high `iostat await`:

```bash
iostat -x 2 10
# Look at: await (ms), r_await, w_await, %util
```

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| `%util = 100%`, `await` high, but SMART clean | Disk is at saturation | Reduce PGs/OSD, rebalance with fewer OSDs per node |
| `w_await` high but `r_await` normal | Write I/O path saturated | Check `bluestore_prefer_deferred_size`, check DB device fullness |
| Intermittent `await` spikes (not sustained) | Disk seek contention from mixed workload | Separate RBD/RGW pools to different OSD classes |
| SMART shows `Reallocated_Sector_Ct > 0` | Drive is failing | Replace the drive, purge OSD |
| High latency only during compaction | RocksDB compaction on WAL/DB device | Move DB device to a larger or faster SSD |

**Check if the BlueStore DB has spilled back to HDD:**

```bash
ceph daemon osd.7 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(d.get('bluestore', {}).get('kv_flush_lat', 'n/a'))
"
# avgtime > 50ms means the DB flush is slow → DB device is full or slow
```

If the DB device is full:

```bash
ceph osd metadata 7 | grep -E "db_type|bluestore_db"
# Check remaining space on the DB device's VG:
ssh <osd-host> vgs
```

Extend or migrate the DB LV to a larger device (requires OSD downtime and `ceph-bluestore-tool`).

---

<!-- Sources:
     - doc/rados/troubleshooting/troubleshooting-osd.rst (github.com/ceph/ceph, main branch, March 2026)
     - doc/rados/configuration/bluestore-config-ref.rst (github.com/ceph/ceph, main branch, March 2026)
     - https://docs.ceph.com/en/latest/rados/configuration/bluestore-config-ref/
     - https://docs.ceph.com/en/latest/rados/troubleshooting/troubleshooting-osd/
-->
