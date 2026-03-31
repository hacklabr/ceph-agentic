# Bootstrap a Ceph Cluster

Bootstrap creates the cluster's first Monitor and Manager, generates cluster keys, and places cephadm in control of all future service deployments. Every subsequent operation — adding hosts, deploying OSDs, enabling the dashboard — runs through the orchestrator that bootstrap creates.

**Prerequisites:** All hosts must be fully prepared before proceeding. See `skills/foundation/host-preparation/README.md`. Bootstrap will fail silently or produce hard-to-diagnose errors if time synchronization, container runtime, or disk state is wrong.

---

## Bootstrap the First Node

### Generate Cluster Spec (optional — recommended for reproducible deployments)

Bootstrap can apply a full cluster spec immediately after creating the cluster. Prepare the spec before running bootstrap if you want a single-command bring-up. This is optional; you can also apply services individually after bootstrap.

Create `cluster-spec.yaml` on the bootstrap host:

```yaml
---
service_type: host
hostname: ceph01
addr: 10.0.0.10
labels:
  - _admin
  - mon
  - mgr
---
service_type: host
hostname: ceph02
addr: 10.0.0.11
labels:
  - mon
  - mgr
  - osd
---
service_type: host
hostname: ceph03
addr: 10.0.0.12
labels:
  - mon
  - osd
---
service_type: mon
placement:
  hosts:
    - ceph01
    - ceph02
    - ceph03
---
service_type: mgr
placement:
  hosts:
    - ceph01
    - ceph02
---
service_type: osd
service_id: default_drive_group
placement:
  host_pattern: 'ceph0[123]'
spec:
  data_devices:
    all: true
```

If you use this spec, pass it to bootstrap with `--apply-spec cluster-spec.yaml`. The SSH key must reach all listed hosts before bootstrap runs — copy `/etc/ceph/ceph.pub` to each host after bootstrap completes, or pre-distribute keys as described in host-preparation.

### Run Bootstrap

Run on the **bootstrap host only** (`ceph01`, IP `10.0.0.10`). Bootstrap is a one-time operation — do not re-run it on an existing cluster.

```bash
cephadm bootstrap \
  --mon-ip 10.0.0.10 \
  --cluster-network 10.0.1.0/24 \
  --allow-fqdn-hostname \
  --ssh-user root \
  --log-to-file
```

Flag reference:

| Flag | Purpose |
|---|---|
| `--mon-ip` | IP address of the first Monitor. Must be reachable from all hosts on the public network. |
| `--cluster-network` | Subnet (CIDR) used for OSD replication and heartbeat traffic. Separating this from the public network is strongly recommended for performance. |
| `--allow-fqdn-hostname` | Required when `hostname` returns a FQDN (e.g. `ceph01.example.com`). Without this flag, bootstrap rejects FQDN hostnames. |
| `--ssh-user root` | The user cephadm will SSH as when managing hosts. Must have passwordless sudo if non-root. |
| `--log-to-file` | Write daemon logs to `/var/log/ceph/<fsid>/` in addition to journald. Useful during initial bring-up. |

**To apply a spec immediately:**

```bash
cephadm bootstrap \
  --mon-ip 10.0.0.10 \
  --cluster-network 10.0.1.0/24 \
  --allow-fqdn-hostname \
  --ssh-user root \
  --log-to-file \
  --apply-spec cluster-spec.yaml
```

Bootstrap takes 2–5 minutes. Successful output ends with:

```
Ceph Dashboard is now available at:

             URL: https://ceph01.example.com:8443/
            User: admin
        Password: <generated-password>

You can access the Ceph CLI as following in case of multi-host cluster:

        sudo /usr/sbin/cephadm shell --fsid <fsid> -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only interested in the admin shell:

        sudo /usr/sbin/cephadm shell

Please consider enabling telemetry to help improve Ceph:

        ceph telemetry on --license sharing-1-0

Bootstrap complete.
```

**Enable the Ceph CLI immediately after bootstrap:**

```bash
cephadm install ceph-common
ceph -v
ceph status
```

### What Bootstrap Creates

| Component | Detail |
|---|---|
| **MON** | One Monitor daemon on the bootstrap host. Forms the initial quorum. Stores the cluster map, authentication keys, and CRUSH map. |
| **MGR** | One Manager daemon on the bootstrap host. Hosts the cephadm orchestration module, Prometheus metrics exporter, and Dashboard module. A second MGR is deployed automatically as standby when a second host is added. |
| **Crash agent** | `crash` daemon on every host — collects crash dumps from Ceph daemons and stores them in the cluster. |
| **Monitoring stack** | Prometheus, Grafana, Alertmanager, and node_exporter containers. Deployed automatically unless `--skip-monitoring-stack` is passed. |
| **SSH key** | A cluster-specific SSH keypair stored in the Monitor config database. Public key written to `/etc/ceph/ceph.pub`. |
| **Config files** | `/etc/ceph/ceph.conf` (minimal cluster config) and `/etc/ceph/ceph.client.admin.keyring` on the bootstrap host. |
| **`_admin` label** | Applied to the bootstrap host. Hosts with this label receive `/etc/ceph/ceph.conf` and `ceph.client.admin.keyring` automatically. |

---

## Add Hosts to the Cluster

Distribute the cephadm SSH key to each additional host before adding it. Run these commands from the bootstrap host.

```bash
# Copy the cephadm public key to each additional host
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.0.11
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.0.12

# Add hosts to the cluster with IP and labels
ceph orch host add ceph02 10.0.0.11 --labels=mon,mgr,osd
ceph orch host add ceph03 10.0.0.12 --labels=mon,osd

# Add the _admin label to ceph02 so it also gets the admin keyring
ceph orch host label add ceph02 _admin
```

Verify all hosts are registered:

```bash
ceph orch host ls
```

Expected output:

```
HOST    ADDR        LABELS              STATUS
ceph01  10.0.0.10   _admin,mon,mgr
ceph02  10.0.0.11   _admin,mon,mgr,osd
ceph03  10.0.0.12   mon,osd
3 hosts in cluster
```

The `STATUS` column is blank for healthy hosts. `offline` or `maintenance` indicate problems — check SSH connectivity and time sync before continuing.

---

## Deploy MONs

A Ceph cluster requires an odd number of MONs for Paxos quorum. Use 3 MONs for clusters with 3–4 nodes and 5 MONs for clusters with 5 or more nodes. cephadm will auto-deploy up to 5 MONs as hosts are added, but explicit placement avoids surprises.

```bash
ceph orch apply mon --placement="ceph01,ceph02,ceph03"
```

Watch MONs come up:

```bash
ceph orch ps --daemon-type mon
```

```
NAME      HOST    PORTS  STATUS         REFRESHED  AGE  MEM USED  MEM LIMIT  VERSION  IMAGE ID      CONTAINER ID
mon.ceph01  ceph01         running (5m)   10s ago   5m    57.2M     -          18.2.7   <id>          <id>
mon.ceph02  ceph02         running (3m)   10s ago   3m    55.1M     -          18.2.7   <id>          <id>
mon.ceph03  ceph03         running (2m)   10s ago   2m    54.8M     -          18.2.7   <id>          <id>
```

Verify quorum:

```bash
ceph mon stat
```

```
e3: 3 mons at {ceph01=[v2:10.0.0.10:3300/0,v1:10.0.0.10:6789/0],ceph02=[v2:10.0.0.11:3300/0,v1:10.0.0.11:6789/0],ceph03=[v2:10.0.0.12:3300/0,v1:10.0.0.12:6789/0]}, election epoch 6, leader 0 ceph01, quorum 0,1,2 ceph01,ceph02,ceph03
```

All three MONs must appear in `quorum` before deploying OSDs. A MON not in quorum indicates a network, time sync, or hostname resolution problem.

**Quorum loss:** With 3 MONs, the cluster tolerates the loss of 1 node. With 5 MONs, it tolerates the loss of 2 nodes. Never run a 2-MON cluster — it has no fault tolerance.

---

## Deploy MGRs

The Manager runs in active/standby mode. The active MGR hosts all enabled modules (dashboard, cephadm, prometheus). The standby takes over automatically if the active MGR dies, with 30–60 seconds of module reload time.

```bash
ceph orch apply mgr --placement="ceph01,ceph02"
```

Verify active and standby:

```bash
ceph orch ps --daemon-type mgr
```

```
NAME       HOST    PORTS  STATUS         REFRESHED  AGE  MEM USED  MEM LIMIT  VERSION  IMAGE ID      CONTAINER ID
mgr.ceph01  ceph01  *:8443  running (10m)  10s ago  10m   509.2M     -          18.2.7   <id>          <id>
mgr.ceph02  ceph02         running (8m)   10s ago   8m   158.3M     -          18.2.7   <id>          <id>
```

The `PORTS` column shows `*:8443` on the active MGR only. The standby keeps running but does not bind the dashboard port.

```bash
ceph mgr stat
```

```
{
    "epoch": 12,
    "available": true,
    "active_name": "ceph01",
    "num_standby": 1
}
```

---

## Deploy OSDs

### Using OSD Spec

An OSD spec gives you precise control over which devices become OSDs and how they are configured. Use this method for production clusters with mixed device types (HDDs + SSDs).

Create `osd-spec.yaml`:

```yaml
service_type: osd
service_id: default_drive_group
placement:
  host_pattern: 'ceph0[123]'
spec:
  data_devices:
    rotational: 1          # select HDDs for data
    size: '1T:'            # only drives >= 1 TB
  db_devices:
    rotational: 0          # select SSDs for BlueStore DB
    size: ':500G'          # only drives <= 500 GB
  wal_devices:
    model: 'WDS200T3X0C'   # optional: target specific NVMe model for WAL
  encryption: false
  osds_per_device: 1
```

Apply the spec:

```bash
ceph orch apply -i osd-spec.yaml
```

Preview what will be deployed before committing:

```bash
ceph orch apply -i osd-spec.yaml --dry-run
```

```
NAME                  HOST    DATA      DB          WAL
default_drive_group   ceph01  /dev/sdb  /dev/nvme0  -
default_drive_group   ceph01  /dev/sdc  /dev/nvme0  -
default_drive_group   ceph01  /dev/sdd  /dev/nvme0  -
default_drive_group   ceph02  /dev/sdb  /dev/nvme0  -
default_drive_group   ceph02  /dev/sdc  /dev/nvme0  -
default_drive_group   ceph02  /dev/sdd  /dev/nvme0  -
default_drive_group   ceph03  /dev/sdb  -           -
default_drive_group   ceph03  /dev/sdc  -           -
default_drive_group   ceph03  /dev/sdd  -           -
```

If the dry-run output matches the intended layout, apply without `--dry-run`.

**HDD-only spec (simpler — no SSD tiering):**

```yaml
service_type: osd
service_id: default_drive_group
placement:
  host_pattern: 'ceph0[123]'
spec:
  data_devices:
    all: true
```

**NVMe-only spec:**

```yaml
service_type: osd
service_id: nvme_drive_group
placement:
  host_pattern: 'ceph0[123]'
spec:
  data_devices:
    rotational: 0
  osds_per_device: 1
```

### Using All Available Devices

For homogeneous clusters where every available disk should become an OSD:

> **WARNING:** `--all-available-devices` consumes every disk that Ceph considers "available" — no partition table, no LVM, no filesystem, >= 5 GB. This includes disks you may not intend to use. Verify disk inventory with `ceph orch device ls` before running this command.

```bash
# Review what will be consumed
ceph orch device ls

# Deploy OSDs on all available devices
ceph orch apply osd --all-available-devices
```

This creates a persistent spec — any new clean disk added to a host in the cluster will automatically become an OSD. To disable auto-deployment of future disks:

```bash
ceph orch apply osd --all-available-devices --unmanaged=true
```

### Verify OSD Deployment

Watch OSDs come up (this takes several minutes):

```bash
watch ceph orch ps --daemon-type osd
```

Check the OSD tree for correct bucket placement:

```bash
ceph osd tree
```

```
ID  CLASS  WEIGHT    TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.08817  root default
-3         0.02939      host ceph01
 0    hdd  0.00980          osd.0       up   1.00000   1.00000
 1    hdd  0.00980          osd.1       up   1.00000   1.00000
 2    hdd  0.00980          osd.2       up   1.00000   1.00000
-5         0.02939      host ceph02
 3    hdd  0.00980          osd.3       up   1.00000   1.00000
 4    hdd  0.00980          osd.4       up   1.00000   1.00000
 5    hdd  0.00980          osd.5       up   1.00000   1.00000
-7         0.02939      host ceph03
 6    hdd  0.00980          osd.6       up   1.00000   1.00000
 7    hdd  0.00980          osd.7       up   1.00000   1.00000
 8    hdd  0.00980          osd.8       up   1.00000   1.00000
```

All OSDs must show `up` status. An OSD that stays `down` after 2–3 minutes indicates a disk problem, missing BlueStore label, or container image pull failure.

Check OSD status summary:

```bash
ceph osd status
```

```
ID  HOST    USED  AVAIL  WR OPS  WR DATA  RD OPS  RD DATA  STATE
 0  ceph01  1026M  9074M      0        0       0        0   exists,up
 1  ceph01  1026M  9074M      0        0       0        0   exists,up
 2  ceph01  1026M  9074M      0        0       0        0   exists,up
 3  ceph02  1026M  9074M      0        0       0        0   exists,up
 4  ceph02  1026M  9074M      0        0       0        0   exists,up
 5  ceph02  1026M  9074M      0        0       0        0   exists,up
 6  ceph03  1026M  9074M      0        0       0        0   exists,up
 7  ceph03  1026M  9074M      0        0       0        0   exists,up
 8  ceph03  1026M  9074M      0        0       0        0   exists,up
```

Check device inventory and availability:

```bash
ceph orch device ls
```

After OSD deployment, previously available devices now show `Available: No`.

---

## Enable the Dashboard

Bootstrap deploys the dashboard module automatically unless `--skip-monitoring-stack` was passed. The module may still need a user account and TLS certificate created.

```bash
# Enable the dashboard module (no-op if already enabled)
ceph mgr module enable dashboard

# Generate a self-signed TLS certificate
ceph dashboard create-self-signed-cert

# Create the admin user
# Write the password to a temp file — the CLI reads from a file, not stdin
echo "ChangeMe123!" > /tmp/dashboard-password.txt
ceph dashboard ac-user-create admin -i /tmp/dashboard-password.txt administrator
rm /tmp/dashboard-password.txt

# Confirm dashboard is running
ceph mgr services
```

```
{
    "dashboard": "https://10.0.0.10:8443/",
    "prometheus": "http://10.0.0.10:9283/"
}
```

Access the dashboard at `https://<active-mgr-host>:8443`. The active MGR host is shown in `ceph mgr stat`. Use the `admin` account with the password you set above.

The dashboard will show a certificate warning in the browser because the certificate is self-signed. For production, replace the self-signed cert with a certificate from your internal CA:

```bash
ceph dashboard set-ssl-certificate -i /path/to/cert.pem
ceph dashboard set-ssl-certificate-key -i /path/to/key.pem
ceph mgr fail   # restart MGR to apply new cert
```

---

## Export Cluster Spec

After the cluster is running, export the current service configuration as a YAML spec. This is the authoritative record of intended cluster state — store it in version control.

```bash
ceph orch ls --export > cluster-spec.yaml
```

Save an immutable original copy for future diffing:

```bash
cp cluster-spec.yaml cluster-spec.yaml.orig
```

The `.orig` file represents the cluster state at bootstrap completion. When making changes, diff against `.orig` to understand what has changed from the initial deployment:

```bash
diff cluster-spec.yaml.orig cluster-spec.yaml
```

Example exported spec (truncated):

```yaml
service_type: mon
service_name: mon
placement:
  hosts:
  - ceph01
  - ceph02
  - ceph03
---
service_type: mgr
service_name: mgr
placement:
  hosts:
  - ceph01
  - ceph02
---
service_type: osd
service_id: default_drive_group
service_name: osd.default_drive_group
placement:
  host_pattern: ceph0[123]
spec:
  data_devices:
    all: true
  encryption: false
  osds_per_device: 1
```

---

## Verify Bootstrap

Run `ceph status` after all services are deployed. A healthy freshly-bootstrapped cluster shows:

```bash
ceph status
```

```
  cluster:
    id:     a3f8c1d2-7e4b-4a1f-8c9d-2b5e3f7a1c4d
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph01,ceph02,ceph03 (age 12m)
    mgr: ceph01(active, since 15m), standbys: ceph02
    osd: 9 osds: 9 up (since 3m), 9 in (since 3m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   27 MiB used, 81 GiB / 81 GiB avail
    pgs:     1 active+clean
```

Key assertions before the cluster is production-ready:

| Check | Expected |
|---|---|
| `health` | `HEALTH_OK` |
| `mon` | All 3 (or 5) daemons in quorum |
| `mgr` | Active MGR listed, at least 1 standby |
| `osd` | All OSDs `up` and `in` |
| `pgs` | All placement groups `active+clean` |

A `HEALTH_WARN` with `noout flag(s) set` after fresh deployment is normal — OSDs are still being marked `in`. It clears within a few minutes.

**If health does not reach `HEALTH_OK` within 10 minutes:**

```bash
# Show all health detail
ceph health detail

# List any failed orchestrator tasks
ceph orch ps | grep -v running

# Check daemon logs for a specific daemon
ceph orch daemon logs osd.0
```

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/cephadm/install.rst
  https://github.com/ceph/ceph/blob/main/doc/cephadm/host-management.rst
  https://github.com/ceph/ceph/blob/main/doc/cephadm/services/osd.rst
  https://github.com/ceph/ceph/blob/main/doc/cephadm/services/mon.rst
  https://github.com/ceph/ceph/blob/main/doc/cephadm/services/mgr.rst
-->
