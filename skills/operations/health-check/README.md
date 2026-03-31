# Health Check

Cluster health monitoring starts with `ceph status`. Every section of that output tells a specific story. This guide explains all of it, gives a complete reference to health codes, and provides a production-ready health-check script.

**When to use this guide:** cluster shows anything other than `HEALTH_OK`, you are setting up monitoring, or an automated alert fired and you need to interpret it.

---

## Quick Status

Run `ceph status` (alias: `ceph -s`) to get a complete snapshot. The output groups into four sections:

```
  cluster:
    id:     477e46f1-ae41-4e43-9c8f-72c918ab0a20
    health: HEALTH_WARN
            1 osds down
            Degraded data redundancy: 42/756 objects degraded (5.556%), 14 pgs degraded

  services:
    mon: 3 daemons, quorum ceph-mon01,ceph-mon02,ceph-mon03 (age 2d)
    mgr: ceph-mon01.abcdef(active, since 2d), standbys: ceph-mon02.fedcba
    osd: 12 osds: 11 up (since 3m), 12 in; 14 remapped pgs
    rgw: 2 daemons active (ceph-rgw01.rgw0, ceph-rgw02.rgw0)

  data:
    pools:   4 pools, 128 pgs
    objects: 252 objects, 4.5 GiB
    usage:   31 GiB used, 109 GiB / 140 GiB avail
    pgs:     42/756 objects degraded (5.556%)
             112 active+clean
             14  active+degraded
             2   active+recovering+degraded

  io:
    client:   1.2 MiB/s rd, 512 KiB/s wr, 120 op/s rd, 80 op/s wr
    recovery: 8.5 MiB/s, 4 keys/s, 12 objects/s
```

### cluster section

| Field | What it means |
|-------|---------------|
| `id` | FSID — unique cluster identifier |
| `health: HEALTH_OK` | All checks pass, no warnings |
| `health: HEALTH_WARN` | One or more non-critical conditions — cluster continues to serve I/O |
| `health: HEALTH_ERR` | Critical condition — data may be unavailable or at risk |
| Health text lines | Each line beneath the status is a health check code with a summary count |

Get verbose detail on any health code:

```bash
ceph health detail
# HEALTH_WARN 1 osds down
# [WRN] OSD_DOWN: 1 osds down
#     osd.3 (root=default,host=ceph-osd03) is down
```

### services section

| Field | Healthy value | Concern |
|-------|--------------|---------|
| `mon: 3 daemons, quorum …` | All provisioned mons in quorum | Any mon missing from quorum list |
| `mgr: name(active, …)` | At least one active, one standby | `no active mgr` blocks metrics and dashboard |
| `osd: N osds: N up, N in` | `up == in` count | Any `down` or `out` mismatch |
| `mds:` | `up:active` for each filesystem | `up:replay` or absent MDS |

### data section

| Field | Meaning |
|-------|---------|
| `pools` | Number of pools and total PG count |
| `objects` | Total stored objects and used space |
| `usage` | Raw cluster space — used, available, total |
| `pgs:` state lines | PG state breakdown; `active+clean` is normal |

### io section

Present only when there is active I/O. `recovery:` line appears when replication repair is in progress. High recovery I/O alongside client I/O signals OSD stress — check `ceph osd perf` for latency.

### Watch cluster state live

```bash
ceph -w                        # stream log events
watch -n 5 ceph -s             # refresh full status every 5 seconds
ceph tell osd.* perf dump      # OSD-level performance counters
```

---

## Health Codes

Health codes follow the pattern `COMPONENT_CONDITION`. Severity is either `HEALTH_WARN` or `HEALTH_ERR`.

### Monitor codes

| Code | Severity | Meaning | Action |
|------|----------|---------|--------|
| `MON_DOWN` | WARN | One or more mons are down; quorum at risk | `systemctl start ceph-mon@<name>` or `ceph orch daemon restart mon.<name>` |
| `MON_CLOCK_SKEW` | WARN | Clock drift exceeds `mon_clock_drift_allowed` (default: 50 ms) | Sync with `chrony` or `ntpd`; check `chronyc tracking` |
| `MON_DISK_LOW` | WARN | Mon filesystem free space below 30% (`mon_data_avail_warn`) | Free space or move mon data dir; check for verbose logging |
| `MON_DISK_CRIT` | ERR | Mon filesystem free space below 5% (`mon_data_avail_crit`) | Immediate action — mon may stop writing; free space now |
| `MON_DISK_BIG` | WARN | Mon database exceeds 15 GiB (`mon_data_size_warn`) | Force compaction: `ceph daemon mon.<id> compact` |
| `MON_MSGR2_NOT_ENABLED` | WARN | Mons not configured for msgr2 protocol | `ceph mon enable-msgr2` |
| `MON_NETSPLIT` | WARN | Network partition detected between mons | Check inter-mon network connectivity and routing |
| `DAEMON_OLD_VERSION` | WARN | Multiple Ceph versions detected; persists >1 week | Complete upgrade; or mute: `ceph health mute DAEMON_OLD_VERSION --sticky` |
| `AUTH_INSECURE_GLOBAL_ID_RECLAIM` | WARN | Clients reconnecting with insecure global_id | Upgrade clients; then `ceph config set mon auth_allow_insecure_global_id_reclaim false` |
| `AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED` | WARN | Cluster still allows insecure global_id reclaim | Once all clients upgraded, disable: `ceph config set mon auth_allow_insecure_global_id_reclaim false` |

### Manager codes

| Code | Severity | Meaning | Action |
|------|----------|---------|--------|
| `MGR_DOWN` | WARN | No active MGR daemon | `ceph orch daemon restart mgr.<name>` or `systemctl start ceph-mgr@<name>` |
| `MGR_MODULE_DEPENDENCY` | WARN | An enabled MGR module fails its dependency check | Install missing Python package; restart MGR daemons |
| `MGR_MODULE_ERROR` | ERR | A MGR module threw an unhandled exception | `ceph mgr fail <active-mgr>` to force failover; check `ceph log last 100` |

### OSD codes

| Code | Severity | Meaning | Action |
|------|----------|---------|--------|
| `OSD_DOWN` | WARN | One or more OSDs are marked down | Check daemon: `systemctl status ceph-osd@<id>`; check host and network |
| `OSD_HOST_DOWN` | WARN | All OSDs on a host are down | Check host connectivity and power; start host before OSDs |
| `OSD_FULL` | ERR | OSD(s) exceeded the `full` ratio (default 95%) — writes blocked | Add capacity, delete data, or temporarily raise `ceph osd set-full-ratio 0.97` |
| `OSD_BACKFILLFULL` | WARN | OSD(s) exceeded backfillfull threshold (default 90%) | Add capacity; rebalancing is blocked until space is freed |
| `OSD_NEARFULL` | WARN | OSD(s) exceeded nearfull threshold (default 85%) | Add capacity or rebalance; plan before reaching full |
| `OSD_ORPHAN` | WARN | OSD referenced in CRUSH but does not exist | `ceph osd crush rm osd.<id>` |
| `OSD_OUT_OF_ORDER_FULL` | WARN | nearfull/backfillfull/full thresholds are not ascending | Correct thresholds: `ceph osd set-nearfull-ratio`, `set-backfillfull-ratio`, `set-full-ratio` |
| `OSD_FLAGS` | WARN | Maintenance flags set on OSDs (`noup`, `nodown`, `noin`, `noout`) | Clear when done: `ceph osd unset-group <flag> <who>` |
| `OSDMAP_FLAGS` | WARN | Cluster-wide flags set (`noout`, `nobackfill`, `noscrub`, etc.) | Clear: `ceph osd unset <flag>` |
| `OSD_FILESTORE` | WARN | OSDs using deprecated Filestore backend | Migrate to BlueStore before upgrading to Reef or later |
| `OSD_UNREACHABLE` | WARN | OSD public address outside `public_network` subnet | Fix network config so OSD address falls within `public_network` |
| `OSDMAP_FLAGS` | WARN | Cluster-wide operation flags set | `ceph osd unset <flag>` when maintenance is complete |
| `OLD_CRUSH_TUNABLES` | WARN | CRUSH map uses outdated tunables | Update: `ceph osd crush tunables optimal` |

### BlueStore codes

| Code | Severity | Meaning | Action |
|------|----------|---------|--------|
| `BLUEFS_SPILLOVER` | WARN | BlueStore metadata spilled from fast DB device to slow device | Expand DB partition or accept reduced metadata performance |
| `BLUEFS_LOW_SPACE` | WARN | BlueFS running low on free space | Expand DB partition; check `ceph daemon osd.<id> bluestore bluefs device info` |
| `BLUESTORE_FRAGMENTATION` | WARN | BlueStore free space fragmented (score 0.9–1.0) | Monitor; excessive fragmentation may degrade BlueFS performance |
| `BLUESTORE_DISK_SIZE_MISMATCH` | WARN | BlueStore metadata device size inconsistency | Destroy and reprovision OSD carefully, one at a time |
| `BLUESTORE_LEGACY_STATFS` | WARN | Pre-Nautilus BlueStore volumes without per-pool stats | Stop OSD, run `ceph-bluestore-tool repair`, restart OSD |
| `BLUESTORE_NO_PER_POOL_OMAP` | WARN | Pre-Octopus volumes without per-pool omap tracking | Same repair procedure as BLUESTORE_LEGACY_STATFS |
| `BLUESTORE_NO_COMPRESSION` | WARN | OSD cannot load BlueStore compression plugin | Verify package installation; restart `ceph-osd` daemon |
| `BLUESTORE_SPURIOUS_READ_ERRORS` | WARN | Read errors recovered by BlueStore retry | Investigate disk health; check kernel/firmware updates |
| `BLOCK_DEVICE_STALLED_READ_ALERT` | WARN | Stalled reads on BlueStore block device | Evaluate disk for failure; check SMART data |
| `BLUESTORE_SLOW_OP_ALERT` | WARN | Repeated slow BlueStore operations | Investigate disk health; check I/O scheduler and kernel |

### Device health codes

| Code | Severity | Meaning | Action |
|------|----------|---------|--------|
| `DEVICE_HEALTH` | WARN | OSD device predicted to fail soon | Mark OSD out, migrate data, then replace hardware |
| `DEVICE_HEALTH_IN_USE` | WARN | Predicted-failing OSD still hosting PGs | Wait for migration to complete or investigate why it is stalled |
| `DEVICE_HEALTH_TOOMANY` | ERR | Too many predicted-failing OSDs to auto-heal | Add replacement OSDs immediately; do not delay |

### Data health codes (pools and PGs)

| Code | Severity | Meaning | Action |
|------|----------|---------|--------|
| `PG_AVAILABILITY` | WARN/ERR | One or more PGs cannot service I/O | Find affected PGs: `ceph health detail`; usually OSD_DOWN is the cause |
| `PG_DEGRADED` | WARN | PGs have fewer than desired replicas | Wait for recovery; investigate `OSD_DOWN` if persistent |
| `PG_RECOVERY_FULL` | ERR | Cluster cannot recover PGs — OSDs at full threshold | See `OSD_FULL` remediation |
| `PG_BACKFILL_FULL` | WARN | Cluster cannot backfill PGs — OSDs at backfillfull | See `OSD_BACKFILLFULL` remediation |
| `PG_DAMAGED` | ERR | Scrub found inconsistent data in one or more PGs | Run repair: `ceph pg repair <pgid>`; investigate hardware |
| `OSD_SCRUB_ERRORS` | ERR | Recent scrubs found object inconsistencies | Paired with `PG_DAMAGED`; see same remediation |
| `OSD_TOO_MANY_REPAIRS` | WARN | Read repair count exceeds `mon_osd_warn_num_repaired` (default: 10) | Investigate failing disks; clear: `ceph tell osd.<id> clear_shards_repaired` |
| `LARGE_OMAP_OBJECTS` | WARN | Oversized omap objects detected | Check RGW bucket indices; enable resharding |
| `TOO_FEW_PGS` | WARN | PG count below `mon_pg_warn_min_per_osd` threshold | Increase PG count or enable pg_autoscaler |
| `TOO_MANY_PGS` | WARN | PG count exceeds `mon_max_pg_per_osd` threshold | Add OSDs or reduce pool PG counts |
| `POOL_FULL` | ERR | Pool has reached its quota limit — writes blocked | `ceph osd pool set-quota <pool> max_bytes <n>` or delete data |
| `POOL_TOO_FEW_PGS` | WARN | Pool should have more PGs for current data volume | `ceph osd pool set <pool> pg_autoscale_mode on` |
| `POOL_TOO_MANY_PGS` | WARN | Pool has more PGs than needed for current data volume | `ceph osd pool set <pool> pg_autoscale_mode on` |
| `TOO_FEW_OSDS` | WARN | OSD count below `osd_pool_default_size` — redundancy at risk | Add OSDs |

### Muting and silencing

Mute a specific health check temporarily during planned maintenance:

```bash
ceph health mute OSD_DOWN 4h                # mute for 4 hours
ceph health mute DAEMON_OLD_VERSION --sticky  # sticky: persists across restarts
ceph health unmute OSD_DOWN                  # remove mute
ceph health mutes                            # list active mutes
```

---

## OSD States

An OSD has two independent state axes. Both axes must be understood together.

### State matrix

| | `in` (assigned PGs) | `out` (no PGs assigned) |
|---|---|---|
| **`up`** (daemon running) | **Normal** — serving data | Daemon running but cluster chose not to assign PGs (new or reweighted OSD) |
| **`down`** (daemon unreachable) | **Problem** — PGs degraded, recovery in progress | Cleanly removed from cluster; data already migrated |

The `down+in` combination is the critical state to watch. It means an OSD is unreachable but the cluster still expects it to hold data, so PGs are degraded.

### Auto-out timer

When an OSD goes `down`, Ceph waits before marking it `out`:

```
mon_osd_down_out_interval = 600  # seconds (default: 10 minutes)
```

After 600 seconds `down`, Ceph marks the OSD `out` and begins rebalancing PGs to surviving OSDs. This is intentional: brief OSD outages (daemon restart, reboot) should not trigger full rebalancing.

Check the timer status:

```bash
ceph osd stat
# 12 osds: 11 up (since 3m), 12 in; epoch: e47

ceph osd dump | grep "^osd\." | grep "down"
# osd.3 down weight 1.0000 ...
```

Prevent auto-out during planned maintenance:

```bash
ceph osd set noout           # disable auto-out cluster-wide
ceph osd set-group noout osd.3 osd.4  # disable for specific OSDs
# ... perform maintenance ...
ceph osd unset noout
```

Check which OSDs are down:

```bash
ceph osd tree | grep down
#  3   hdd 1.00000      osd.3              down  1.00000 1.00000

ceph osd find 3
# {"osd":3,"ip":"10.0.1.13:6800/1","crush_location":{"host":"ceph-osd03","root":"default"}}
```

Restart a specific OSD daemon:

```bash
# On the OSD host:
systemctl start ceph-osd@3

# Via cephadm orchestrator (from any admin node):
ceph orch daemon restart osd.3
```

---

## PG States

Placement Groups (PGs) pass through well-defined states. Most states are transient and self-resolving.

### State reference table

| State | Normal? | Meaning | Action |
|-------|---------|---------|--------|
| `active+clean` | Yes — target state | PG is peered, all replicas present and consistent | None |
| `active+clean+scrubbing` | Yes — transient | Periodic light scrub in progress | None; will clear in minutes |
| `active+clean+deep` | Yes — transient | Deep scrub in progress (weekly by default) | None; will clear in hours |
| `active+degraded` | Transient | PG active and serving I/O, but one or more replicas missing | Wait for recovery; check for `OSD_DOWN` |
| `active+recovering` | Transient | OSD returned from `down`; PG is resynchronising objects | Normal; watch progress with `ceph status` |
| `active+remapped` | Transient | CRUSH reassigned PG to new OSD set; data migration in progress | Normal after adding/removing OSDs |
| `active+backfilling` | Transient | New OSD receiving PG data from existing cluster | Normal after adding OSDs; may be slow |
| `active+backfill_wait` | Transient | Backfill queued, not yet started | Normal; `osd_max_backfills` throttles concurrency |
| `active+backfill_toofull` | Persistent | Backfill blocked: target OSD above `backfillfull` threshold | Add capacity or remove data; see `OSD_BACKFILLFULL` |
| `active+undersized` | Persistent | Fewer OSDs available than the pool's replication factor | Add OSDs or reduce pool `size` setting |
| `active+incomplete` | Persistent | PG cannot assemble a complete authoritative log | Urgent: may indicate data loss; run `ceph pg <pgid> query` |
| `peering` | Transient | OSDs agreeing on PG state; normal during OSD restarts | Wait; persists >5 min = investigate |
| `stale` | Persistent | Primary OSD not reporting PG stats to MON | OSD is down; start OSD daemon or investigate host |
| `inactive` | Persistent | PG cannot service reads or writes; waiting for authoritative OSD | Check for `OSD_DOWN`; PG needs enough replicas to peer |
| `unclean` | Persistent | Objects not replicated to desired count for extended time | Investigate OSD availability; check space |
| `inconsistent` | Persistent | Scrub found mismatched data between replicas | `ceph pg repair <pgid>`; investigate hardware |
| `repair` | Transient | Repair of scrub inconsistency in progress | Wait; check outcome with `ceph pg <pgid> query` |
| `creating` | Transient | New pool PGs being created and peered | Wait after pool creation |
| `unknown` | Never | MON has not received a report for this PG | Network or OSD issue |

### Decision tree for stuck PGs

```
PG not active+clean?
│
├── State is "peering" for > 5 minutes
│   ├── Check OSD availability: ceph osd tree
│   └── Query PG: ceph pg <pgid> query — look for "blocked" in peering state
│
├── State is "stale"
│   ├── Primary OSD is down → start the OSD daemon
│   └── If OSD is up: check MON connectivity from OSD host
│
├── State is "inactive" or "incomplete"
│   ├── Not enough OSDs up for this PG to peer
│   ├── Check: ceph pg <pgid> query | jq '.acting, .up'
│   └── URGENT if incomplete: potential data loss; do not proceed without Ceph expertise
│
├── State is "degraded" (active+degraded)
│   ├── Self-resolving: OSD was recently down and is now recovering
│   └── Persistent > 30 min: investigate OSD_DOWN and recovery throttles
│
└── State is "backfill_toofull"
    ├── Target OSD is too full to accept backfill data
    └── Free space or add OSDs; check: ceph osd df
```

### Query a specific PG

```bash
ceph pg 1.4a query
# {
#   "state": "active+degraded",
#   "acting": [2, 5],
#   "up": [2, 5],
#   "acting_primary": 2,
#   "blocked_by": [3],
#   ...
# }

# Find stuck PGs
ceph pg dump_stuck inactive
ceph pg dump_stuck unclean
ceph pg dump_stuck stale
ceph pg dump_stuck degraded
```

### Repair an inconsistent PG

```bash
ceph health detail | grep inconsistent
# pg 2.1f is active+clean+inconsistent

ceph pg repair 2.1f
# instructing pg 2.1f on osd.4 to repair

# Verify repair completed
ceph pg 2.1f query | jq '.state'
```

---

## Capacity Monitoring

### Cluster-level capacity

```bash
ceph df
# --- RAW STORAGE ---
# CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
# hdd    2.7 TiB  2.2 TiB  517 GiB  517 GiB      18.73
# TOTAL  2.7 TiB  2.2 TiB  517 GiB  517 GiB      18.73
#
# --- POOLS ---
# POOL          ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
# device_health   1    1  1.1 MiB        2  3.3 MiB      0    693 GiB
# cephfs.data     2   32  135 GiB    2.24k  404 GiB  16.32    693 GiB
# cephfs.meta     3   16  1.3 MiB      596  3.9 MiB      0    693 GiB
# rbd             4   32  221 GiB    2.57k  663 GiB  26.76    693 GiB
```

`MAX AVAIL` is per-pool usable capacity accounting for replication factor — this is what limits writes to each pool.

### Per-OSD capacity

```bash
ceph osd df
# ID CLASS WEIGHT  REWEIGHT SIZE    RAW USE DATA    OMAP    META    AVAIL   %USE  VAR  PGS  STATUS
#  0   hdd  0.9095   1.0000 931 GiB  71 GiB  69 GiB 286 KiB 2.0 GiB 860 GiB  7.57 0.42   48  up
#  1   hdd  0.9095   1.0000 931 GiB 143 GiB 141 GiB 581 KiB 2.0 GiB 788 GiB 15.37 0.86   48  up
#  2   hdd  0.9095   1.0000 931 GiB 237 GiB 235 GiB 1.0 MiB 2.0 GiB 694 GiB 25.43 1.43   48  up
# ...
# TOTAL                     2.7 TiB 517 GiB 513 GiB 1.9 MiB 6.0 GiB 2.2 TiB 18.73          up

ceph osd df tree          # show CRUSH hierarchy with utilization
```

`VAR` column: values much above 1.0 indicate uneven data distribution — that OSD is holding more than its fair share.

### Fullness thresholds

| Threshold | Default | Health Code | Effect |
|-----------|---------|-------------|--------|
| `nearfull_ratio` | 0.85 (85%) | `OSD_NEARFULL` | Warning only; cluster still operates normally |
| `backfillfull_ratio` | 0.90 (90%) | `OSD_BACKFILLFULL` | Backfill to affected OSD is blocked |
| `full_ratio` | 0.95 (95%) | `OSD_FULL` | All writes to the cluster are blocked |
| `failsafe_full_ratio` | 0.97 (97%) | Internal | OSD will refuse writes even if `full` flag is unset |

View current ratios:

```bash
ceph osd dump | grep -E "(full|nearfull|backfillfull)_ratio"
# full_ratio 0.95
# backfillfull_ratio 0.9
# nearfull_ratio 0.85
```

Adjust thresholds (emergency capacity extension — address root cause immediately):

```bash
ceph osd set-nearfull-ratio 0.87
ceph osd set-backfillfull-ratio 0.92
ceph osd set-full-ratio 0.97
```

Monitor capacity trends:

```bash
# Raw cluster utilisation over time (requires Prometheus)
# Metric: ceph_cluster_total_used_bytes / ceph_cluster_total_bytes

# Per-pool quota
ceph osd pool get-quota rbd
# quotas for pool 'rbd':
#   max objects: N/A
#   max bytes  : 500 GiB

# Set a pool quota
ceph osd pool set-quota rbd max_bytes 500G
```

---

## Service Status

### Orchestrator service inventory

```bash
ceph orch ls
# NAME             PORTS        RUNNING  REFRESHED  AGE  PLACEMENT
# alertmanager     ?:9093           1/1  8s ago     4d   count:1
# crash                             3/3  8s ago     4d   *
# grafana          ?:3000           1/1  8s ago     4d   count:1
# mgr                               2/2  8s ago     4d   count:2
# mon                               3/3  8s ago     4d   count:3
# node-exporter    ?:9100           3/3  8s ago     4d   *
# osd              ?                12   8s ago     4d   <unmanaged>
# prometheus       ?:9095           1/1  8s ago     4d   count:1
# rgw.default      ?:80             2/2  8s ago     4d   count:2

ceph orch ls --service-type osd     # OSDs only
ceph orch ls --format yaml          # full YAML spec
```

### Per-daemon process status

```bash
ceph orch ps
# NAME                    HOST          PORTS  STATUS         REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
# alertmanager.ceph-mon01  ceph-mon01  9093   running (4d)    4s ago    4d    31.3M        -  0.23.0   ba2b418f427c  cd3a84b28de8
# crash.ceph-mon01         ceph-mon01         running (4d)    4s ago    4d    9.9M         -  17.2.7   ...           ...
# mgr.ceph-mon01.abcdef    ceph-mon01  8443   running (4d)    4s ago    4d    478M         -  17.2.7   ...           ...
# mgr.ceph-mon02.fedcba    ceph-mon02  8443   running (4d)    4s ago    4d    213M         -  17.2.7   ...           ...
# mon.ceph-mon01           ceph-mon01         running (4d)    4s ago    4d     34M         -  17.2.7   ...           ...
# osd.0                    ceph-osd01         running (4d)    4s ago    4d    1510M        -  17.2.7   ...           ...

ceph orch ps --daemon-type osd      # list all OSD daemons
ceph orch ps --daemon-type mgr
ceph orch ps --hostname ceph-osd03  # all daemons on a specific host
```

Identify crashed or stopped daemons by the `STATUS` column: `stopped`, `error`, or `crash` require immediate attention.

### Restart a specific daemon

```bash
ceph orch daemon restart osd.3
ceph orch daemon restart mgr.ceph-mon01.abcdef
ceph orch daemon stop osd.3        # stop without restart
ceph orch daemon start osd.3       # start a stopped daemon
```

### Check crash reports

```bash
ceph crash ls
# ID                                                         ENTITY  NEW
# 2024-01-15T14:23:11.000000Z_a1b2c3d4-e5f6-...  osd.3    *

ceph crash info 2024-01-15T14:23:11.000000Z_a1b2c3d4-e5f6-...
# {
#   "crash_id": "2024-01-15T14:23:11.000000Z_...",
#   "timestamp": "2024-01-15 14:23:11.000000Z",
#   "process_name": "ceph-osd",
#   "entity_name": "osd.3",
#   "backtrace": [ ... ]
# }

ceph crash archive-all    # archive after investigation (clears HEALTH_WARN)
```

### Monitor quorum

```bash
ceph quorum_status -f json-pretty
# {
#   "election_epoch": 12,
#   "quorum": [0, 1, 2],
#   "quorum_names": ["ceph-mon01", "ceph-mon02", "ceph-mon03"],
#   "quorum_leader_name": "ceph-mon01",
#   ...
# }

ceph mon stat
# e3: 3 mons at {ceph-mon01=[v2:10.0.0.1:3300/0,...], ...}, election epoch 12
# leader ceph-mon01, quorum 0,1,2

ceph mon dump
# dumped monmap epoch 3
# ...
```

---

## Dashboard Monitoring

### Access the dashboard

The Ceph Dashboard is served by the active MGR daemon:

```bash
ceph mgr services
# {
#   "dashboard": "https://10.0.0.1:8443/",
#   "prometheus": "http://10.0.0.1:9095/",
# }
```

Open `https://<mgr-host>:8443` in a browser. Default admin credentials are set during bootstrap:

```bash
ceph dashboard ac-user-show admin           # verify admin user exists
ceph dashboard ac-user-set-password admin   # reset admin password
```

### Built-in Prometheus and Grafana

cephadm deploys Prometheus and Grafana as managed services alongside the cluster:

| Service | Default port | Purpose |
|---------|-------------|---------|
| Prometheus | 9095 | Metrics scraping and storage |
| Grafana | 3000 | Pre-built Ceph dashboards |
| Alertmanager | 9093 | Alert routing and deduplication |
| node-exporter | 9100 | Per-host OS and hardware metrics |

The dashboard embeds Grafana iframes. Pre-built panels include:

- Cluster Overview — health, capacity, IOPS
- OSDs — per-OSD utilization, latency, apply/commit time
- Pools — per-pool IOPS, bandwidth, PG states
- Hosts — CPU, RAM, network per storage node
- RBD — per-image metrics
- RGW — request rates, error rates by bucket

### Prometheus alerting rules

Ceph ships built-in alert rules deployed by cephadm. View active alerts:

```bash
# From Alertmanager API
curl -s http://<mgr-host>:9093/api/v2/alerts | jq '.[].labels'

# Check Prometheus targets are all up
curl -s http://<mgr-host>:9095/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

Key Prometheus metrics to monitor:

| Metric | Alert threshold | Meaning |
|--------|----------------|---------|
| `ceph_health_status` | > 0 | `1` = WARN, `2` = ERR |
| `ceph_osd_up` | < expected OSD count | OSDs not in `up` state |
| `ceph_pg_degraded` | > 0 for > 10 min | PGs without full redundancy |
| `ceph_cluster_total_used_bytes / ceph_cluster_total_bytes` | > 0.80 | Raw cluster utilization |
| `ceph_osd_apply_latency_ms` | > 100 ms sustained | OSD apply latency |
| `ceph_osd_commit_latency_ms` | > 100 ms sustained | OSD commit latency |

---

## Automated Health Script

Save this as `/usr/local/bin/ceph-health-check.sh` on any admin node. Run it from cron or a CI pipeline.

```bash
#!/usr/bin/env bash
# ceph-health-check.sh — Ceph cluster health check script
# Exit codes: 0 = OK, 1 = WARN, 2 = ERR, 3 = script error
# Usage: ./ceph-health-check.sh [--json] [--quiet]
set -euo pipefail

SCRIPT_NAME="$(basename "$0")"
JSON_OUTPUT=false
QUIET=false
EXIT_CODE=0

for arg in "$@"; do
  case "$arg" in
    --json)   JSON_OUTPUT=true ;;
    --quiet)  QUIET=true ;;
  esac
done

log()  { $QUIET || echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
warn() { echo "[WARN]  $*" >&2; EXIT_CODE=$((EXIT_CODE > 1 ? EXIT_CODE : 1)); }
err()  { echo "[ERROR] $*" >&2; EXIT_CODE=2; }

# --- Verify ceph is reachable ---
if ! timeout 10 ceph status &>/dev/null; then
  echo "[FATAL] Cannot reach Ceph cluster (timeout or auth failure)" >&2
  exit 3
fi

# --- Collect cluster status ---
HEALTH_STATUS=$(ceph health 2>/dev/null)
HEALTH_LEVEL=$(echo "$HEALTH_STATUS" | awk '{print $1}')

log "=== Ceph Health Check ==="
log "Status: $HEALTH_STATUS"

# --- Check overall health ---
case "$HEALTH_LEVEL" in
  HEALTH_OK)
    log "Cluster is healthy"
    ;;
  HEALTH_WARN)
    warn "Cluster in HEALTH_WARN: $HEALTH_STATUS"
    log "Detail:"
    ceph health detail 2>/dev/null | while IFS= read -r line; do log "  $line"; done
    ;;
  HEALTH_ERR)
    err "Cluster in HEALTH_ERR: $HEALTH_STATUS"
    log "Detail:"
    ceph health detail 2>/dev/null | while IFS= read -r line; do log "  $line"; done
    ;;
  *)
    echo "[FATAL] Unexpected health output: $HEALTH_STATUS" >&2
    exit 3
    ;;
esac

# --- OSD status ---
OSD_STAT=$(ceph osd stat 2>/dev/null)
log ""
log "=== OSD Status ==="
log "$OSD_STAT"

OSD_UP=$(echo "$OSD_STAT" | grep -oP '\d+(?= up)')
OSD_IN=$(echo "$OSD_STAT" | grep -oP '\d+(?= in)')
OSD_TOTAL=$(echo "$OSD_STAT" | grep -oP '^\d+')

if [[ "$OSD_UP" -lt "$OSD_TOTAL" ]]; then
  OSDS_DOWN=$(( OSD_TOTAL - OSD_UP ))
  warn "$OSDS_DOWN OSD(s) are down (up=${OSD_UP}, total=${OSD_TOTAL})"
fi

if [[ "$OSD_IN" -lt "$OSD_TOTAL" ]]; then
  OSDS_OUT=$(( OSD_TOTAL - OSD_IN ))
  log "Note: ${OSDS_OUT} OSD(s) are out of cluster (may be intentional)"
fi

# --- PG status ---
log ""
log "=== PG Status ==="
PG_STAT=$(ceph pg stat 2>/dev/null)
log "$PG_STAT"

if echo "$PG_STAT" | grep -qvE "active\+clean"; then
  UNCLEAN=$(ceph pg dump_stuck inactive 2>/dev/null | grep -c "^[0-9]" || true)
  STALE=$(ceph pg dump_stuck stale 2>/dev/null | grep -c "^[0-9]" || true)
  DEGRADED=$(ceph pg dump_stuck degraded 2>/dev/null | grep -c "^[0-9]" || true)
  [[ "$UNCLEAN" -gt 0 ]] && warn "${UNCLEAN} inactive PGs"
  [[ "$STALE" -gt 0 ]]   && warn "${STALE} stale PGs"
  [[ "$DEGRADED" -gt 0 ]] && warn "${DEGRADED} stuck degraded PGs"
fi

# --- Capacity ---
log ""
log "=== Capacity ==="
USAGE_LINE=$(ceph df 2>/dev/null | grep "^TOTAL")
log "$USAGE_LINE"

RAW_USED=$(ceph df 2>/dev/null | awk '/^TOTAL/{print $NF}' | tr -d '%')
if [[ -n "$RAW_USED" ]]; then
  if (( $(echo "$RAW_USED > 90" | bc -l) )); then
    err "Cluster raw utilization critical: ${RAW_USED}% used"
  elif (( $(echo "$RAW_USED > 80" | bc -l) )); then
    warn "Cluster raw utilization high: ${RAW_USED}% used"
  else
    log "Raw utilization: ${RAW_USED}% (OK)"
  fi
fi

# --- Crash reports ---
log ""
log "=== Crash Reports ==="
CRASH_COUNT=$(ceph crash ls 2>/dev/null | grep -c "NEW" || true)
if [[ "$CRASH_COUNT" -gt 0 ]]; then
  warn "${CRASH_COUNT} unacknowledged crash report(s)"
  ceph crash ls 2>/dev/null | while IFS= read -r line; do log "  $line"; done
  log "  Run: ceph crash info <id>  to investigate"
  log "  Run: ceph crash archive-all  to acknowledge after investigation"
else
  log "No unacknowledged crash reports"
fi

# --- Service status (requires cephadm orchestrator) ---
if ceph orch status &>/dev/null 2>&1; then
  log ""
  log "=== Service Status ==="
  STOPPED=$(ceph orch ps 2>/dev/null | grep -c "stopped\|error\|crash" || true)
  if [[ "$STOPPED" -gt 0 ]]; then
    warn "${STOPPED} daemon(s) stopped or in error state"
    ceph orch ps 2>/dev/null | grep -E "stopped|error|crash" | \
      while IFS= read -r line; do log "  $line"; done
  else
    log "All daemons running"
  fi
fi

# --- Summary ---
log ""
log "=== Summary ==="
case "$EXIT_CODE" in
  0) log "Result: OK — cluster is healthy" ;;
  1) log "Result: WARNING — attention required" ;;
  2) log "Result: ERROR — immediate action required" ;;
esac

if $JSON_OUTPUT; then
  python3 -c "
import subprocess, json, sys
s = json.loads(subprocess.check_output(['ceph', '-s', '-f', 'json']))
print(json.dumps({'health': s.get('health', {}), 'osdmap': s.get('osdmap', {})}, indent=2))
"
fi

exit "$EXIT_CODE"
```

Make it executable and test:

```bash
chmod +x /usr/local/bin/ceph-health-check.sh
ceph-health-check.sh                  # standard output
ceph-health-check.sh --json           # include JSON cluster summary
ceph-health-check.sh --quiet 2>&1 | grep -E "WARN|ERROR"  # silent mode
echo $?                                # 0=OK 1=WARN 2=ERR 3=unreachable
```

Schedule via cron (run as root or ceph admin user):

```
# /etc/cron.d/ceph-health
*/5 * * * * root /usr/local/bin/ceph-health-check.sh --quiet >> /var/log/ceph/health-check.log 2>&1
```

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/rados/operations/health-checks.rst
  https://github.com/ceph/ceph/blob/main/doc/rados/operations/monitoring.rst
  https://github.com/ceph/ceph/blob/main/doc/rados/operations/monitoring-osd-pg.rst
-->
