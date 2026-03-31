# Scaling the Cluster

Ceph scales horizontally — add hosts and OSDs to grow capacity, remove them to decommission hardware. All operations go through `ceph orch` (cephadm) except explicit noout/norebalance flags which talk directly to the monitor.

**When to use this guide:** adding hosts, expanding OSD capacity, decommissioning hardware, resizing the monitor or manager quorum.

---

## Decision Tree

```
Need more capacity?
├── Existing host has free drives  → Add OSDs to Existing Host
├── New server arriving            → Add a New Host → Add OSDs to New Host
└── Spec-managed cluster           → Update OSD spec, cephadm reconciles

Need to remove hardware?
├── Individual OSD failure         → Remove OSDs (leave host in cluster)
├── Full host decommission         → Remove a Host (drains all daemons)
└── Offline / unrecoverable host   → Offline Host Removal
```

---

## Add a New Host

### 1. Prepare the host

The new host must satisfy:
- OS: supported Linux distribution with `podman` or `docker` available
- Python 3 installed
- Chronyd or ntpd running (clock skew breaks Ceph)
- Unique hostname set correctly: `hostname` must return the name you will register

### 2. Copy the cluster SSH key

Run from any admin node:

```bash
# Copy the cluster public key to the new host
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd04

# Verify SSH access before proceeding
ssh root@ceph-osd04 hostname
```

If you do not have the public key locally:

```bash
ceph cephadm get-pub-key > /tmp/ceph.pub
ssh-copy-id -f -i /tmp/ceph.pub root@ceph-osd04
```

### 3. Add the host to the cluster

```bash
# Add host with explicit IP (preferred — avoids DNS dependency)
ceph orch host add ceph-osd04 10.0.1.14

# Add with the _admin label if this host should carry ceph.conf and keyrings
ceph orch host add ceph-osd04 10.0.1.14 --labels _admin
```

The name passed to `ceph orch host add` must exactly match the output of `hostname` on the remote host.

### 4. Verify

```bash
# Confirm host is known and reachable
ceph orch host ls

# Expected:
# HOST        ADDRESS      LABELS   STATUS
# ceph-mon01  10.0.1.10    _admin
# ceph-osd01  10.0.1.11
# ceph-osd04  10.0.1.14             # new host appears here
```

```bash
# Check that cephadm can connect
ceph orch ps ceph-osd04
```

---

## Add OSDs to an Existing Host

### Option A — Add a specific device (immediate, one-shot)

```bash
# First confirm the device is available (Available = Yes)
ceph orch device ls --hostname=ceph-osd01

# Add a single device
ceph orch daemon add osd ceph-osd01:/dev/sde

# Add with separate DB device (e.g. NVMe for metadata)
ceph orch daemon add osd ceph-osd01:data_devices=/dev/sde,db_devices=/dev/nvme0n1
```

Note: `ceph orch daemon add` creates the OSD but does not modify the persistent service spec. If a spec already applies to this host, the new OSD inherits it via `osd.default`.

### Option B — Update the OSD spec (declarative, persistent)

```bash
# View current OSD spec
ceph orch ls --service-type osd

# Edit and re-apply the spec — cephadm will reconcile new devices
cat > /tmp/osd-spec.yaml << 'EOF'
service_type: osd
service_id: default_drive_group
placement:
  hosts:
    - ceph-osd01
    - ceph-osd02
    - ceph-osd03
    - ceph-osd04          # add new host here if needed
spec:
  data_devices:
    rotational: true      # all spinning HDDs
EOF

# Dry run first
ceph orch apply -i /tmp/osd-spec.yaml --dry-run

# Apply
ceph orch apply -i /tmp/osd-spec.yaml
```

Once applied with `ceph orch apply`, any new matching device that appears on a matching host is automatically converted to an OSD.

### Wait for backfill and verify

```bash
# Watch recovery progress
ceph -w

# Or check progress events
ceph progress

# Cluster returns to HEALTH_OK when backfill completes
# Verify OSD count increased
ceph osd stat
# 15 osds: 15 up (since 2m), 15 in; 32 remapped pgs   ← pgs clear when done
```

---

## Add OSDs to a New Host

After running `ceph orch host add` the workflow depends on whether a spec already exists:

**Spec-managed cluster (recommended):**

```bash
# If your OSD spec uses label placement, label the new host
ceph orch host label add ceph-osd04 osd-node

# cephadm reconciles within ~60 seconds — verify
ceph orch device ls --hostname=ceph-osd04
ceph orch ps ceph-osd04
```

**No existing spec:**

```bash
# Consume all available devices on the new host immediately
ceph orch apply osd --all-available-devices --dry-run
ceph orch apply osd --all-available-devices
```

**Individual device targeting:**

```bash
ceph orch daemon add osd ceph-osd04:/dev/sdb
ceph orch daemon add osd ceph-osd04:/dev/sdc
ceph orch daemon add osd ceph-osd04:/dev/sdd
```

---

## Remove OSDs

> **WARNING:** Removing OSDs triggers data migration (backfill) from the removed OSD to survivors. Never remove more than `replica_count − 1` OSDs simultaneously from the same CRUSH subtree (host or rack). Removing all copies of a PG causes data loss.

### Standard removal (cephadm)

```bash
# Step 1: Set noout to prevent CRUSH from marking OSD out prematurely during removal
ceph osd set noout

# Step 2: Schedule the OSD for removal
# ceph orch osd rm drains PGs then purges the OSD
ceph orch osd rm 7

# To remove and wipe the device in one operation (safe for reuse)
ceph orch osd rm 7 --zap

# Remove multiple OSDs (only if enough replicas remain)
ceph orch osd rm 7 8 --zap
```

### Monitor removal progress

```bash
# Poll removal state
ceph orch osd rm status

# Example output:
# OSD_ID  HOST        STATE                    PG_COUNT  REPLACE  FORCE  STARTED_AT
# 7       ceph-osd03  draining                 42        False    False  2025-03-01 14:22:10
# 8       ceph-osd03  done, waiting for purge  0         False    False  2025-03-01 14:22:12
```

```bash
# Watch cluster health in parallel
ceph -w
```

### Step 3: Unset noout after removal completes

```bash
# Confirm OSD is gone
ceph osd stat

# Unset noout
ceph osd unset noout

# Verify cluster returns to HEALTH_OK
ceph status
```

### Cancel a queued removal

```bash
ceph orch osd rm stop 7
# Output: Stopped OSD(s) removal
```

### Prevent automatic redeployment on freed device

After an OSD is removed and its device is zapped, cephadm will redeploy an OSD on it if a matching spec is still active. To prevent this:

```bash
# Mark the spec unmanaged before removing
ceph orch osd set-spec-affinity osd.default_drive_group --unmanaged=true

# Or update the spec YAML to exclude the device, then re-apply
```

---

## Remove a Host

> **WARNING:** Draining a host removes all daemons including MONs and MGRs. Ensure remaining MONs still form a quorum of 3+ before draining any MON host.

### 1. Drain all daemons from the host

```bash
# Schedules removal of all daemons — including OSDs
ceph orch host drain ceph-osd04

# To also wipe OSD devices during drain
ceph orch host drain ceph-osd04 --zap-osd-devices
```

This applies `_no_schedule` (blocks new deployments) and queues OSD removal.

### 2. Monitor OSD drain progress

```bash
# Watch OSD removal
ceph orch osd rm status

# Watch overall cluster rebalance
ceph -w
```

### 3. Confirm all daemons are gone

```bash
ceph orch ps ceph-osd04
# Empty output means all daemons removed
```

### 4. Remove the host from the cluster

```bash
ceph orch host rm ceph-osd04

# Also remove from CRUSH map (requires OSDs already removed)
ceph orch host rm ceph-osd04 --rm-crush-entry
```

### 5. Verify

```bash
ceph orch host ls           # host no longer listed
ceph status                 # HEALTH_OK, OSD count reflects removal
```

### Offline host removal (unrecoverable hardware)

> **WARNING:** This forcibly purges OSDs and can cause permanent data loss if the downed OSDs held the only copy of any PG. Confirm data is fully replicated before proceeding.

```bash
ceph orch host rm ceph-osd04 --offline --force
```

After forced removal, manually update any service specs that still reference the removed host.

---

## Scale MONs

MONs use Paxos consensus. Rules:
- Minimum: **3** (tolerates 1 failure)
- Always use **odd** numbers: 3, 5, 7
- Never scale below 3

### Increase to 5 MONs

```bash
# cephadm places MONs automatically; specify count
ceph orch apply mon 5

# Or pin to specific hosts
ceph orch apply mon --placement="ceph-mon01,ceph-mon02,ceph-mon03,ceph-mon04,ceph-mon05"
```

### Decrease from 5 to 3 MONs

```bash
ceph orch apply mon 3
# cephadm removes 2 MON daemons; quorum re-forms with remaining 3
```

### Verify quorum

```bash
ceph mon stat
# Output: e3: 5 mons at {ceph-mon01,ceph-mon02,ceph-mon03,ceph-mon04,ceph-mon05}, quorum 0,1,2,3,4

ceph quorum_status --format json-pretty | jq '.quorum_names'
# ["ceph-mon01","ceph-mon02","ceph-mon03","ceph-mon04","ceph-mon05"]
```

---

## Scale MGRs

MGRs run active/standby. One active handles requests; standbys take over on failure.

```bash
# Scale to 3 (1 active + 2 standby — recommended for HA)
ceph orch apply mgr 3

# Verify
ceph mgr stat
# Output: active: ceph-mon01.xyzabc, standbys: [ceph-mon02.abcdef, ceph-mon03.fedcba]
```

Two MGRs is the minimum for zero-downtime failover. One MGR is acceptable only in lab environments.

---

## Monitor Rebalance

After any OSD addition or removal, Ceph redistributes placement groups. Large clusters with many PGs can take hours.

### Real-time view

```bash
# Follow log events — shows recovery rates and PG transitions
ceph -w

# Track named background operations
ceph progress
```

### Snapshot view

```bash
ceph status
# data section shows:
#   pgs:     320/2400 objects misplaced (13.3%)
#            896 active+clean
#            128 active+remapped+backfilling
#            16  active+recovering

# OSD performance during recovery
ceph osd perf
```

### Throttle recovery to protect client I/O

If recovery is saturating disks or network, throttle it:

```bash
# Reduce concurrent recovery operations per OSD (default: 3)
ceph config set osd osd_recovery_max_active 1

# Reduce max backfills per OSD (default: 1)
ceph config set osd osd_max_backfills 1

# Restore defaults after recovery completes
ceph config rm osd osd_recovery_max_active
ceph config rm osd osd_max_backfills
```

### Check when rebalance is complete

```bash
ceph status | grep -E "pgs:|active\+clean"
# All PGs active+clean = rebalance finished

ceph osd stat
# N osds: N up, N in  (no "remapped pgs" line = done)
```

### Rebalance with noout during planned maintenance

```bash
# Set before removing a host for maintenance
ceph osd set noout
ceph osd set norebalance   # stronger: also halts backfill

# Do maintenance work...

# Unset both when done
ceph osd unset norebalance
ceph osd unset noout
```

---

## Reference: Device Availability

A device is eligible for OSD deployment only when all conditions are met:
- No existing partitions
- No LVM state
- Not mounted
- No filesystem
- No existing BlueStore OSD
- Size ≥ 5 GB

Check with:

```bash
ceph orch device ls --wide
# REJECT REASONS column explains why any device is unavailable
```

Force a rescan when hardware was hot-added or a device was recently zapped:

```bash
ceph orch device ls --refresh
ceph orch host rescan ceph-osd04 --with-summary
```

---

<!-- Source: ceph/ceph doc/cephadm/host-management.rst, doc/cephadm/services/osd.rst, doc/rados/operations/add-or-rm-osds.rst (fetched 2026-03-29) -->
