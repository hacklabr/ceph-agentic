# Troubleshooting

Symptom-based decision trees for diagnosing Ceph cluster problems. Each section follows the same structure: symptom → diagnosis commands → identify cause → apply fix.

**When to use this guide:** `ceph status` shows `HEALTH_WARN` or `HEALTH_ERR`, a client cannot write or read, or an alert has fired and you need to trace the root cause.

---

## Start Here

Every investigation starts with three commands in order:

```bash
# 1. Get the high-level picture
ceph status

# 2. Get the specific health codes and affected components
ceph health detail

# 3. Look at the recent log stream for errors and warnings
ceph log last 50
```

Read the output of `ceph health detail` and find the section in this guide that matches your symptom. Work top-down through each decision tree until you identify the cause.

**Prerequisite check:** Before diagnosing OSDs or PGs, confirm monitors are healthy:
```bash
ceph quorum_status --format json-pretty | jq '.quorum_names'
# If the array is missing members, fix MON issues first — see MON Issues section.
```

---

## OSD Issues

### OSD Down

**Symptom:** `ceph health detail` shows `OSD_DOWN`:
```
HEALTH_WARN 1 osds down
[WRN] OSD_DOWN: 1 osds down
    osd.3 (root=default,host=ceph-osd03) is down
```

**Step 1 — Identify which OSD and on which host:**
```bash
ceph health detail
# osd.3 is down since epoch 23, last address 192.168.106.220:6800/11080

ceph osd tree down
# Shows tree view of all down OSDs and their host placement
```

**Step 2 — Check if the process is running on the host:**
```bash
ssh ceph-osd03

# For systemd deployments:
systemctl status ceph-osd@3.service

# For cephadm / containerized deployments:
cephadm ls | grep osd.3
```

If the process is dead → go to Step 3.
If the process is running but OSD is still marked down → go to Step 5 (network/heartbeat).

**Step 3 — Check the OSD log for the reason it stopped:**
```bash
# Systemd deployments:
journalctl -u ceph-osd@3.service --since "30 minutes ago" | tail -50

# File-based logs:
tail -100 /var/log/ceph/ceph-osd.3.log

# Cephadm / containerized:
cephadm logs --name osd.3
```

Look for key patterns:
- `suicide timeout` or `heartbeat` → OSD killed itself; check drive health (Step 4)
- `Corruption` or `assert` → store corruption; drive may need replacement
- `std::bad_alloc` or `OOM` → insufficient RAM; need 4–8 GB per OSD daemon
- `ENOSPC` → device is full; see OSD Full section

**Step 4 — Check drive health:**
```bash
# Identify the device backing osd.3
ls -la /var/lib/ceph/osd/ceph-3/block

# SMART health check:
smartctl -a /dev/sdX

# Kernel messages for the device:
dmesg | grep -i "sdX\|error\|failed\|timeout" | tail -20

# I/O stats (look for await > 200ms on the OSD device):
iostat -x 2 5
```

If the drive shows SMART errors or kernel I/O errors → replace the drive, redeploy the OSD.
If the drive is healthy → restart the OSD:

```bash
# Systemd:
sudo systemctl start ceph-osd@3.service

# Cephadm:
ceph orch daemon restart osd.3
```

**Step 5 — Network / heartbeat investigation (process running but OSD down):**
```bash
# Check connectivity between OSD hosts on the cluster (backend) network:
ping -c 4 <cluster-network-ip-of-ceph-osd03>

# Check for connection tracking table overflow:
dmesg | grep "nf_conntrack: table full"

# Check that ports are reachable:
nc -zv ceph-osd03 6800
nc -zv ceph-osd03 6801

# Check OSD peering / heartbeat config:
ceph daemon osd.3 config show | grep heartbeat
```

If `nf_conntrack` is full → increase the limit:
```bash
sysctl -w net.netfilter.nf_conntrack_max=524288
# Persist in /etc/sysctl.d/99-ceph.conf
```

---

### OSD Full

**Symptom:** `ceph health detail` shows `OSD_FULL`, `OSD_BACKFILLFULL`, or `OSD_NEARFULL`:
```
HEALTH_ERR 1 full osd(s); 1 backfillfull osd(s); 1 nearfull osd(s)
osd.3 is full at 97%
osd.4 is backfill full at 91%
osd.2 is near full at 87%
```

Clients are blocked from writing when any OSD in a pool exceeds the `full_ratio` (default 0.95).

**Step 1 — Identify which OSDs are full and the current utilization:**
```bash
ceph osd df sort --by-use
# Shows utilization per OSD sorted by % used; focus on the most-full outliers

ceph df
# Shows pool-level usage and how much space pools can still absorb
```

**Step 2 — Check the fullness thresholds:**
```bash
ceph config get mon mon_osd_full_ratio        # default 0.95
ceph config get mon mon_osd_backfillfull_ratio # default 0.90
ceph config get mon mon_osd_nearfull_ratio     # default 0.85
```

**Step 3 — Identify the cause of the imbalance:**

If one or a few OSDs are significantly fuller than others → data is not evenly distributed:
```bash
# Rebalance by utilization (moves data away from overloaded OSDs):
ceph osd reweight-by-utilization

# Or enable the balancer module:
ceph mgr module enable balancer
ceph balancer on
ceph balancer status
```

If the cluster is globally full → you need to add capacity or delete data:
```bash
# Add a new OSD:
ceph orch daemon add osd <host>:<device>

# Or temporarily lower thresholds to allow some writes while you add capacity:
ceph osd set-full-ratio 0.97
ceph osd set-backfillfull-ratio 0.95
ceph osd set-nearfull-ratio 0.90
```

If the cluster is full due to a failed OSD whose data has been replicated → mark the OSD out to redistribute data to surviving OSDs:
```bash
ceph osd out osd.X
# Wait for rebalancing to complete, then the surviving OSDs will accept writes again
```

**Do not delete PG directories on a full OSD unless absolutely certain other replicas exist — data loss will result.**

---

### Slow OSD Requests

**Symptom:** Clients report timeouts or high latency; cluster log shows:
```
[WRN] 1 slow requests, 1 included below; oldest blocked for > 30.005692 secs
[WRN] slow request 30.005692 seconds old, received at 2024-01-15 14:23:11.054801:
      osd_op(client.4240.0:8 benchmark_data_ceph-1_39426_object7 [write 0~4194304] 0.69848840)
      currently waiting for subops from [610]
```

The threshold is controlled by `osd_op_complaint_time` (default 30 s).

**Step 1 — Identify how many OSDs are affected:**
```bash
ceph health detail | grep "slow request"
ceph osd perf
# Look for apply_latency_ms or commit_latency_ms > 100ms
```

**Step 2 — Check in-flight and historical operations on the slow OSD:**
```bash
# Show operations currently executing (replace 3 with the affected OSD id):
ceph daemon osd.3 dump_ops_in_flight

# Show the last 20 completed operations and their stage timings:
ceph daemon osd.3 dump_historic_ops
```

Look at the event timeline for each op. Bottleneck stages:
- Long time at `queued_for_pg` → CPU / thread saturation; check `top`, `htop`
- Long time at `waiting for subops from` → replica OSD is slow; investigate the replica OSD
- Long time between `started` and `op_commit` → storage I/O is slow

**Step 3 — Check storage I/O:**
```bash
iostat -x 1 10
# Look at: await (ms), %util on OSD drives
# HDD: await > 20ms is concerning; %util at 100% means saturated
# SSD/NVMe: await > 5ms on writes is a concern

# Check for drive errors:
dmesg | grep -E "(error|timeout|reset)" | grep -i sd | tail -20
smartctl -a /dev/sdX
```

**Step 4 — Check for recovery or scrubbing consuming resources:**
```bash
ceph status | grep -E "recovering|scrubbing|backfilling"

# Throttle recovery to free I/O for client ops:
ceph config set osd osd_recovery_max_active_hdd 1
ceph config set osd osd_max_backfills 1
```

**Step 5 — mClock scheduler (HDD clusters only):**

If OSDs use the mClock scheduler and the cluster is HDD-based, the default shard config can cause queueing:
```bash
# Check current shard config:
ceph config get osd osd_op_num_shards_hdd
ceph config get osd osd_op_num_threads_per_shard_hdd

# If shards=5 and threads_per_shard=1, swap to:
ceph config set osd osd_op_num_shards_hdd 1
ceph config set osd osd_op_num_threads_per_shard_hdd 5
# Requires a rolling OSD restart to take effect
```

---

### OSD Flapping

**Symptom:** The cluster log repeatedly shows OSDs going up and then down:
```
[INF] osd.4 192.168.1.104:6801/543 boot
[WRN] osd.4 failed (ejecting osd.4 from osd map)
[INF] osd.4 192.168.1.104:6801/543 boot
```

**Step 1 — Confirm flapping is occurring:**
```bash
ceph health detail
# [WRN] OSD_FLAPPING: ...

ceph osd dump | grep flags
# Look for: no-up, no-down already set by automation

# Inspect recent OSD map changes for repeated up/down on same OSD:
ceph osd stat
```

**Step 2 — Temporarily freeze OSD state changes to investigate:**
```bash
ceph osd set noup    # prevent OSDs from being marked up
ceph osd set nodown  # prevent OSDs from being marked down

# Confirm flags are set:
ceph osd dump | grep flags
# flags no-up,no-down
```

**Step 3 — Investigate the network:**
```bash
ssh <osd-host>

# Check the cluster (backend/private) network interface:
ip addr show
ethtool <cluster-iface> | grep -E "Speed|Link"

# Check for packet loss on the cluster network to other OSD hosts:
ping -c 100 <another-osd-cluster-ip> | tail -3

# Check for errors on the interface:
ip -s link show <cluster-iface>
# Look for: errors, dropped, overrun in the RX/TX counters

# Check OSD heartbeat timing:
ceph daemon osd.4 config show | grep heartbeat
```

**Step 4 — Identify network root cause:**

If **all** OSDs on a host are flapping → the host's cluster network interface or switch port has a problem. Check the switch for CRC errors, negotiation mismatches, or a faulty cable.

If only **some** OSDs are flapping → those specific OSD processes may be crashing due to I/O timeouts (see Slow OSD Requests above).

If flapping stops when the cluster and public networks are merged (single network) → the private/cluster network link is degraded. Fix or bond it; do not use a degraded dedicated network.

**Step 5 — Restore normal operation:**
```bash
ceph osd unset noup
ceph osd unset nodown
```

---

## MON Issues

### MON Clock Skew

**Symptom:** `ceph health detail` reports:
```
HEALTH_WARN clock skew detected on mon.c
mon.c addr 10.10.0.1:6789/0 clock skew 0.08235s > max 0.05s (latency 0.0045s)
```

The Paxos consensus algorithm requires all monitor clocks to be within 50 ms of each other (configurable via `mon_clock_drift_allowed`).

**Step 1 — Verify clock synchronization status:**
```bash
# Check relative offsets between monitors:
ceph time-sync-status

# On each monitor host, check NTP/Chrony sync:
chronyc tracking
# or:
timedatectl status
# Look for: "System clock synchronized: yes" and offset < 50ms
```

**Step 2 — If NTP is not syncing:**
```bash
# Check chrony sources:
chronyc sources -v
# Stars (*) indicate the active sync source; '+' indicates acceptable sources

# If no sources are reachable, check connectivity to the NTP server:
ping <ntp-server-ip>

# Force a resync:
chronyc makestep
```

**Step 3 — If NTP is syncing but skew persists:**
- Clocks on VMs are unreliable — NTP on bare metal on hypervisor hosts, not inside VMs
- Ensure monitors have multiple NTP peers (each other, internal servers, external pool)
- A skew > 50 ms after NTP is running indicates an NTP configuration problem or a very remote time server

**Step 4 — Temporary mitigation (not a fix):**
```bash
# Raise the tolerance threshold while you fix the underlying NTP issue:
ceph config set mon mon_clock_drift_allowed 0.1
# Do not leave this elevated permanently
```

---

### MON Quorum Lost

**Symptom:** `ceph status` hangs or returns `Error connecting to cluster`, or `ceph health detail` shows monitors out of quorum:
```
mon.a (rank 0) addr 127.0.0.1:6789/0 is down (out of quorum)
```

Quorum requires a majority. A 3-monitor cluster needs at least 2; a 5-monitor cluster needs at least 3.

**Step 1 — Check how many monitors are reachable:**
```bash
# If ceph status hangs, connect to a specific monitor:
ceph status -m <mon-host>

# Check status of each monitor individually (works without quorum):
ceph tell mon.a mon_status
ceph tell mon.b mon_status
ceph tell mon.c mon_status
```

The `state` field in the output of `mon_status` tells you where each monitor is:
- `leader` or `peon` → in quorum
- `probing` → looking for peers (often means it cannot reach other monitors)
- `electing` → in an election; elections should resolve quickly
- `synchronizing` → catching up, will join quorum when done

**Step 2 — If a monitor is stuck `probing`:**
```bash
# The monitor can't reach peers. Check network connectivity:
ssh <mon-host>
nc -zv <other-mon-host> 6789
nc -zv <other-mon-host> 3300

# Check for firewall rules blocking mon ports:
iptables -L -n | grep -E "6789|3300"

# Check whether the monmap has the correct addresses:
ceph tell mon.a mon_status | jq '.monmap.mons'
```

If the monmap has wrong addresses → inject a correct monmap:
```bash
# Extract a good monmap from a working monitor:
ceph mon getmap -o /tmp/monmap
monmaptool --print /tmp/monmap

# Stop the broken monitor, inject the map, restart:
systemctl stop ceph-mon@a
ceph-mon -i a --inject-monmap /tmp/monmap
systemctl start ceph-mon@a
```

**Step 3 — If a monitor is stuck `electing` (election storm):**

Clock skew is the most common cause. Check all monitor clocks (see MON Clock Skew section).

**Step 4 — If all monitors are down:**
```bash
# Check if the mon daemon process is running:
systemctl status ceph-mon@a.service

# If the process died due to store corruption, check the log:
journalctl -u ceph-mon@a --since "1 hour ago" | grep -E "Corruption|assert|ERROR"

# If the store is corrupted, rebuild from OSDs:
# See MON Store Growing section below for recovery commands
```

**Step 5 — Start the surviving monitors first:**
```bash
# On each surviving monitor host:
systemctl start ceph-mon@<id>.service

# Wait for quorum to re-form:
watch ceph quorum_status
```

---

### MON Store Growing

**Symptom:** Disk usage on monitor hosts is growing unexpectedly, or `ceph health detail` reports:
```
HEALTH_WARN mon.a store is getting too large
```

Alternatively, the monitor log shows store corruption after an unclean shutdown.

**Step 1 — Check the size of the monitor store:**
```bash
du -sh /var/lib/ceph/mon/ceph-*/store.db
# A healthy store is typically 1–5 GB; larger indicates a problem
```

**Step 2 — Compact the store:**
```bash
# While the monitor is running:
ceph tell mon.<id> compact

# Check that it reduced after compaction:
du -sh /var/lib/ceph/mon/ceph-*/store.db
```

**Step 3 — If the store is corrupted (log shows `Corruption: error in middle of record`):**

If other monitors are healthy → remove and re-add the corrupted monitor:
```bash
# Remove the broken monitor from the quorum:
ceph mon remove <id>

# On the broken monitor host, stop and wipe the store:
systemctl stop ceph-mon@<id>
rm -rf /var/lib/ceph/mon/ceph-<id>/store.db

# Re-add and allow it to sync from peers:
ceph mon add <id> <ip>:6789
systemctl start ceph-mon@<id>
```

If **all** monitors fail simultaneously → rebuild store from OSDs (extreme case):
```bash
# Collect maps from each OSD on each host:
ms=/root/mon-store
mkdir $ms

for host in ceph-osd01 ceph-osd02 ceph-osd03; do
  ssh $host "for osd in /var/lib/ceph/osd/ceph-*; do
    ceph-objectstore-tool --data-path \$osd --no-mon-config \
      --op update-mon-db --mon-store-path $ms
  done"
done

# Rebuild the monitor store:
ceph-monstore-tool $ms rebuild -- \
  --keyring /etc/ceph/ceph.client.admin.keyring \
  --mon-ids mon-a mon-b mon-c

# Back up old store and replace:
mv /var/lib/ceph/mon/ceph-a/store.db /var/lib/ceph/mon/ceph-a/store.db.bak
mv $ms/store.db /var/lib/ceph/mon/ceph-a/store.db
chown -R ceph:ceph /var/lib/ceph/mon/ceph-a/store.db
```

---

## PG Issues

### Degraded PGs

**Symptom:** `ceph health detail` or `ceph status` shows degraded objects:
```
HEALTH_WARN Degraded data redundancy: 42/756 objects degraded (5.556%), 14 pgs degraded
pg 3.4 is active+degraded
```

`active+degraded` means the PG is serving I/O but does not have all replica copies. Data is not at risk as long as OSDs continue to function, but a second failure could cause data loss.

**Step 1 — Identify the affected PGs and the missing OSD:**
```bash
ceph health detail
# Lists each degraded PG and the acting OSD set

ceph pg dump_stuck degraded
# Detailed dump of all degraded PGs

# Find which OSDs are down that would hold replicas:
ceph osd tree down
```

**Step 2 — If the missing OSD is recoverable:**
```bash
# Restart the down OSD — recovery will begin automatically:
systemctl start ceph-osd@<id>.service
# or:
ceph orch daemon restart osd.<id>

# Watch recovery progress:
watch ceph status
# Look for: recovery: X MiB/s, X objects/s
```

**Step 3 — If recovery is taking too long:**
```bash
# Check current recovery throttle settings:
ceph config get osd osd_recovery_max_active_hdd  # default 3
ceph config get osd osd_recovery_op_priority       # default 3

# Increase recovery throughput (will impact client I/O):
ceph config set osd osd_recovery_max_active_hdd 5

# Check backfill progress:
ceph pg stat
# Look for: X pgs: N active+recovering
```

**Step 4 — If the OSD is permanently lost:**
```bash
# Mark the OSD out — CRUSH redistributes its PGs to other OSDs:
ceph osd out osd.<id>

# Wait for all PGs to recover, then remove the OSD:
ceph osd purge osd.<id> --yes-i-really-mean-it
```

---

### Stale PGs

**Symptom:** `ceph health detail` shows stale PGs:
```
HEALTH_WARN 24 pgs stale; 3/300 in osds are down
pg 2.5 is stuck stale+active+remapped, last acting [2,0]
```

A stale PG is one that has not received a status update from its primary OSD. All OSDs that hold that PG are down.

**Step 1 — Identify the stale PGs and the OSDs responsible for them:**
```bash
ceph pg dump_stuck stale

ceph health detail
# pg 2.5 is stuck stale+active+remapped, last acting [2,0]
# → osd.0 and osd.2 held this PG; both must be down
```

**Step 2 — Find and restart the OSDs:**
```bash
ceph osd tree down
# osd.0 on ceph-osd01
# osd.2 on ceph-osd03

# Restart them:
ssh ceph-osd01 systemctl start ceph-osd@0.service
ssh ceph-osd03 systemctl start ceph-osd@2.service

# Monitor PG recovery:
watch ceph pg stat
```

**Step 3 — If the OSDs cannot be restarted (permanent failure):**

If the PGs are stale because all copies of the data are on dead OSDs, those objects are unavailable (and may be irrecoverable). Check if there is a backup before proceeding.

```bash
# Check if peering is blocked by waiting for the down OSD:
ceph pg 2.5 query | jq '.recovery_state'
# Look for: "peering is blocked due to down osds"

# If you are certain data is lost and want the cluster to move on:
ceph osd lost osd.<id>   # Mark the OSD as permanently lost
# This allows peering to complete; data on that OSD is gone
```

---

### Undersized PGs

**Symptom:** `ceph health detail` shows:
```
HEALTH_WARN 14 pgs undersized
pg 1.3 is active+undersized+degraded
```

An undersized PG has fewer acting OSDs than required by the pool's `size` setting, but meets the `min_size` threshold so it can still serve I/O.

**Step 1 — Check pool size settings vs. available OSDs:**
```bash
ceph osd pool ls detail
# Look for: size X, min_size Y for each pool

ceph osd stat
# X osds: Y up, Z in
# If Y < pool size → PGs will be undersized
```

**Step 2 — Check for down OSDs and recover them:**
```bash
ceph osd tree down
# Recover or replace the down OSDs — PGs will return to full size
```

**Step 3 — If the pool size exceeds the available OSD count permanently:**
```bash
# Lower the pool size to match available OSDs (data risk — do this carefully):
ceph osd pool set <pool-name> size 2
ceph osd pool set <pool-name> min_size 1
```

---

### Inconsistent PGs

**Symptom:** `ceph health detail` shows:
```
HEALTH_ERR 1 pgs inconsistent; 2 scrub errors
pg 0.6 is active+clean+inconsistent, acting [0,1,2]
2 scrub errors
```

Inconsistency is detected during scrubbing: one replica has different content from the authoritative copy.

**Step 1 — Identify which PG and which objects are inconsistent:**
```bash
ceph health detail
# pg 0.6 is active+clean+inconsistent

# Find inconsistent PGs in a specific pool:
rados list-inconsistent-pg <pool-name>
# ["0.6"]

# List the specific inconsistent objects in the PG:
rados list-inconsistent-obj 0.6 --format=json-pretty
```

The output shows:
- `errors` — mismatches between replicas (e.g., `data_digest_mismatch`, `size_mismatch`)
- `shards` — which OSD has the bad copy
- `read_error` — physical I/O error reading from that OSD's storage

**Step 2 — Check for physical storage errors if `read_error` is present:**
```bash
# Identify the device:
ceph osd metadata osd.<id> | jq '.osd_objectstore_dev'
# or:
ls -la /var/lib/ceph/osd/ceph-<id>/block

smartctl -a /dev/sdX
dmesg | grep -i "sdX\|error\|reset" | tail -20
```

If `read_error` appears → replace that OSD's drive before running repair. Repair will promote the good replicas as authoritative and discard the bad copy.

**Step 3 — Repair the inconsistent PG:**
```bash
ceph pg repair 0.6
# Ceph promotes the authoritative copy and overwrites the bad shard

# Monitor scrub to confirm it clears:
watch ceph health detail
```

If the inconsistency reoccurs after repair → the drive has intermittent bad sectors. Replace it.

**Step 4 — Enable automatic repair for minor inconsistencies (optional):**
```bash
ceph config set osd osd_scrub_auto_repair true
ceph config set osd osd_scrub_auto_repair_num_errors 5
# Applies to BlueStore and erasure-coded pools only
```

---

## Service Issues

### MDS Not Active

**Symptom:** CephFS mounts fail, or `ceph status` shows no active MDS:
```
mds: <none>
```
or:
```
mds: 1/1 daemons up, 0 standby
# followed by client mount failures
```

**Step 1 — Check the MDS status:**
```bash
ceph mds stat
# Reports: up:active={X} or no active MDS

ceph fs status
# Shows filesystem name, active rank, standby daemons, pools
```

**Step 2 — If no MDS daemon is running:**
```bash
# Cephadm:
ceph orch ps | grep mds
ceph orch daemon restart mds.<id>

# Systemd:
systemctl status ceph-mds@<id>.service
journalctl -u ceph-mds@<id> --since "30 minutes ago" | tail -50
```

**Step 3 — If MDS is running but stuck in `replay` or `reconnect` state:**
```bash
ceph mds stat
# State: replay or reconnect

# MDS is replaying the journal after a crash. It will proceed automatically.
# If stuck more than 10 minutes:
ceph tell mds.<id> flush journal
```

**Step 4 — If MDS is slow or causing client timeouts:**
```bash
# Check MDS cache usage:
ceph daemon mds.<id> status | grep -E "cache|mds_mem"

# If cache is thrashing, increase the MDS cache size:
ceph config set mds mds_cache_memory_limit 4294967296  # 4 GB
```

---

### RGW Not Responding

**Symptom:** S3/Swift API calls fail, or Ceph Object Gateway (RGW) returns 5xx errors.

**Step 1 — Check RGW daemon status:**
```bash
ceph orch ps | grep rgw

# Direct service check:
curl -s http://<rgw-host>:7480/
# Should return an XML response; connection refused means daemon is down
```

**Step 2 — If the daemon is down:**
```bash
# Cephadm:
ceph orch daemon restart rgw.<id>

# Systemd:
systemctl restart ceph-radosgw@rgw.<zone>.service
journalctl -u ceph-radosgw@rgw.default --since "30 minutes ago" | tail -50
```

**Step 3 — If the daemon is running but returning errors:**
```bash
# Check RGW log for request errors:
tail -100 /var/log/ceph/ceph-client.rgw.*.log
# or:
cephadm logs --name rgw.<id> | tail -100

# Check the RGW metadata pool:
ceph health detail | grep -i rgw
rados -p .rgw.root ls | head -10
```

**Step 4 — If RGW can't reach OSDs (cluster healthy but RGW gets I/O errors):**
```bash
# Verify the RGW keyring has correct caps:
ceph auth get client.rgw.<id>
# Expected caps: mon 'allow rw', osd 'allow rwx', mgr 'allow rw'

# Test direct RADOS access:
rados -p default.rgw.meta ls --max-objects 5
```

---

## Log Locations

| Component | Default Log Path | Cephadm (containerized) |
|-----------|-----------------|-------------------------|
| OSD | `/var/log/ceph/ceph-osd.<id>.log` | `cephadm logs --name osd.<id>` |
| MON | `/var/log/ceph/ceph-mon.<id>.log` | `cephadm logs --name mon.<id>` |
| MGR | `/var/log/ceph/ceph-mgr.<id>.log` | `cephadm logs --name mgr.<id>` |
| MDS | `/var/log/ceph/ceph-mds.<id>.log` | `cephadm logs --name mds.<id>` |
| RGW | `/var/log/ceph/ceph-client.rgw.*.log` | `cephadm logs --name rgw.<id>` |
| Cluster-wide | `ceph log last 100` | same |

For all file-based logs:
```bash
ls /var/log/ceph/
# Files named: ceph-<daemon-type>.<id>.log
# Rotated files: ceph-osd.3.log.1, ceph-osd.3.log.2.gz, ...
```

---

## Collecting Debug Info

When a problem cannot be diagnosed from normal health output, raise debug levels and collect extended logs.

**Raise debug level on a running daemon (no restart required):**
```bash
# For a specific monitor (with or without quorum):
ceph tell mon.<id> config set debug_mon 10/10
ceph tell mon.<id> config set debug_ms 1/5

# For all monitors at once:
ceph tell mon.* config set debug_mon 10/10

# For a specific OSD:
ceph tell osd.<id> config set debug_osd 10/10
ceph tell osd.<id> config set debug_ms 1/5

# Without quorum, use the admin socket directly:
ceph daemon mon.<id> config set debug_mon 10/10
ceph daemon osd.<id> config set debug_osd 10/10
```

**Collect a diagnostic bundle for a daemon:**
```bash
# Admin socket help lists all available debug commands:
ceph daemon osd.<id> help

# Dump performance counters:
ceph daemon osd.<id> perf dump

# Dump current config:
ceph daemon osd.<id> config show

# Dump PG states for the OSD:
ceph daemon osd.<id> dump_pgs_brief

# For monitors:
ceph daemon mon.<id> quorum_status
ceph daemon mon.<id> mon_status
```

**Cluster-wide diagnostic snapshot:**
```bash
# Full cluster report (useful for bug reports):
ceph report > /tmp/ceph-report-$(date +%Y%m%d-%H%M%S).json

# OSD map:
ceph osd dump > /tmp/osdmap.txt

# PG map summary:
ceph pg stat
ceph pg dump > /tmp/pgmap.txt

# Crush map:
ceph osd getcrushmap -o /tmp/crushmap.bin
crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt
```

**Return debug levels to normal when done:**
```bash
ceph tell mon.* config set debug_mon 1/5
ceph tell osd.* config set debug_osd 1/5
```

---

<!-- Source: Official Ceph documentation
     - doc/rados/troubleshooting/troubleshooting-osd.rst
     - doc/rados/troubleshooting/troubleshooting-mon.rst
     - doc/rados/troubleshooting/troubleshooting-pg.rst
     Accessed from https://github.com/ceph/ceph (main branch, March 2026)
-->
