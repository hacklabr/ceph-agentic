# Host Preparation

Prepare every node before running `cephadm bootstrap`. Skipping steps here — missing time sync, dirty disks, no container runtime — causes bootstrap failures or subtle cluster problems that are hard to diagnose after the fact.

---

## Supported Operating Systems

| Distribution | Versions | Ceph Support Level |
|---|---|---|
| Ubuntu | 22.04 LTS (Jammy) | A — packages + comprehensive tests |
| Ubuntu | 24.04 LTS (Noble) | A — packages + comprehensive tests (Squid/Tentacle) |
| Rocky Linux / RHEL | 8.x | A — packages + comprehensive tests (Reef) |
| Rocky Linux / RHEL | 9.x | A — packages + comprehensive tests (Reef/Squid/Tentacle) |
| Debian | 12 (Bookworm) | C — packages only, no upstream test coverage |
| CentOS Stream | 9 | A — packages + comprehensive tests |

**Recommended:** Ubuntu 22.04 or Rocky Linux 9 for new production deployments. RHEL 9 and CentOS Stream 9 are the primary RPM targets upstream.

Ubuntu 20.04 is only supported through Reef (18.2.z). Do not deploy it for new clusters. Ubuntu 24.04 is supported from Squid onward.

---

## Prerequisites Checklist

Complete every item on every node that will join the cluster — bootstrap host and all additional nodes.

1. OS is freshly installed or baseline-imaged from a known-good template.
2. Hostname is set correctly (`hostnamectl set-hostname <name>`).
3. `/etc/hosts` or DNS resolves all cluster node hostnames.
4. A container runtime (Podman or Docker) is installed and the daemon is running.
5. Python 3.6 or later is installed (`python3 --version`).
6. systemd is the init system (`systemctl --version`).
7. LVM2 is installed (`lvmconfig --version`).
8. Chrony is installed, enabled, and synchronized.
9. SSH daemon is running on all nodes (`systemctl status sshd`).
10. The bootstrap node can reach all other nodes over SSH as root (or a passwordless-sudo user).
11. Firewall rules are open for all required Ceph ports (see [Configure Firewall](#configure-firewall)).
12. OSD disks are clean — no existing partition tables, LVM signatures, or filesystems.
13. Network interfaces on the public and (if separate) cluster network are up and addressed.
14. NTP synchronization is confirmed (`chronyc tracking` shows offset < 50 ms).

---

## Install Container Runtime

cephadm requires Podman or Docker. Podman is preferred on RHEL-family systems because it ships in the standard repos and does not require a daemon. Docker is required on Ubuntu where Podman packages are older.

### Podman (recommended on Rocky/RHEL)

**Rocky Linux 9 / RHEL 9:**

```bash
dnf install -y podman
systemctl enable --now podman.socket
podman --version
```

**Rocky Linux 8 / RHEL 8:**

```bash
dnf install -y podman
podman --version
```

Podman on RHEL 8/9 does not require a daemon — it runs rootless or rootful containers directly. The socket unit is only needed for Docker-API compatibility; cephadm does not require it.

**Ubuntu 22.04 / 24.04:**

Ubuntu's default Podman packages may lag behind the versions cephadm supports. Check the Ceph compatibility matrix before using Podman on Ubuntu. Docker is the safer choice on Ubuntu.

```bash
apt-get update
apt-get install -y podman
podman --version
```

### Docker (alternative, preferred on Ubuntu)

**Ubuntu 22.04 / 24.04:**

```bash
apt-get update
apt-get install -y ca-certificates curl gnupg lsb-release

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io
systemctl enable --now docker
docker --version
```

**Rocky Linux 9 / RHEL 9 (Docker alternative):**

Note: Red Hat does not support Docker on RHEL. Use Podman instead. If Docker is required for operational reasons:

```bash
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io
systemctl enable --now docker
docker --version
```

**Rocky Linux 8:**

```bash
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io
systemctl enable --now docker
docker --version
```

---

## Install cephadm

Use either the package manager method or the curl method — not both on the same host.

### From Package Manager

**Ubuntu 22.04 / 24.04:**

```bash
apt-get update
apt-get install -y cephadm
cephadm version
```

**Rocky Linux 9 / CentOS Stream 9:**

```bash
dnf search release-ceph
dnf install -y centos-release-ceph-squid
dnf install -y cephadm
cephadm version
```

For Reef on Rocky 8:

```bash
dnf search release-ceph
dnf install -y centos-release-ceph-reef
dnf install -y cephadm
cephadm version
```

### From Curl (standalone)

Use this method for air-gapped environments, quick-start setups, or when you need a specific version.

```bash
CEPH_RELEASE=19.2.2   # replace with the active release from https://docs.ceph.com/en/latest/releases/
curl --silent --remote-name --location \
  https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm
./cephadm version
```

To install cephadm system-wide after verifying the standalone binary:

```bash
./cephadm add-repo --release squid
./cephadm install
which cephadm   # should return /usr/sbin/cephadm
```

For air-gapped environments, transfer the binary to each host manually and install it into `/usr/sbin/`:

```bash
install -m 0755 cephadm /usr/sbin/cephadm
```

---

## Configure Time Synchronization

MON daemons use Paxos consensus. Clock skew above ~0.05 seconds causes `HEALTH_WARN` and above 0.5 seconds causes MON quorum loss. Every node in the cluster must synchronize to the same NTP source.

**Ubuntu 22.04 / 24.04:**

```bash
apt-get install -y chrony
systemctl enable --now chronyd
```

Edit `/etc/chrony/chrony.conf` to point at your NTP servers (or leave the default pool):

```
pool ntp.ubuntu.com iburst maxsources 4
```

Restart and verify:

```bash
systemctl restart chronyd
chronyc tracking
chronyc sources -v
```

The `System time` line in `chronyc tracking` output should show offset in milliseconds, not seconds. `Reference ID` should not be `7F7F0101` (local fallback).

**Rocky Linux 8/9 / RHEL 8/9:**

chrony is installed by default. Verify and enable:

```bash
systemctl enable --now chronyd
chronyc tracking
```

Edit `/etc/chrony.conf` to specify NTP servers if the defaults are unavailable:

```
server ntp1.example.com iburst
server ntp2.example.com iburst
```

```bash
systemctl restart chronyd
chronyc sources -v
```

Confirm offset is below 50 ms on every node before proceeding to bootstrap.

---

## Configure Firewall

Open required ports before bootstrapping. cephadm does not manage host firewall rules.

### firewalld (Rocky Linux / RHEL)

```bash
# MON
firewall-cmd --permanent --add-port=3300/tcp
firewall-cmd --permanent --add-port=6789/tcp

# OSD, MGR, MDS (dynamic range)
firewall-cmd --permanent --add-port=6800-7568/tcp

# MGR Dashboard
firewall-cmd --permanent --add-port=8443/tcp

# MGR Prometheus exporter
firewall-cmd --permanent --add-port=9283/tcp

# RGW (if deploying object storage)
firewall-cmd --permanent --add-port=7480/tcp

firewall-cmd --reload
firewall-cmd --list-ports
```

If using a dedicated cluster network on a separate interface (e.g., `ens4`), apply the same port rules to that zone:

```bash
firewall-cmd --permanent --zone=internal --add-port=6800-7568/tcp
firewall-cmd --reload
```

### ufw (Ubuntu)

```bash
# MON
ufw allow 3300/tcp
ufw allow 6789/tcp

# OSD, MGR, MDS (dynamic range)
ufw allow 6800:7568/tcp

# MGR Dashboard
ufw allow 8443/tcp

# MGR Prometheus exporter
ufw allow 9283/tcp

# RGW (if deploying object storage)
ufw allow 7480/tcp

ufw reload
ufw status numbered
```

For port reference, see the Port Requirements table in `skills/foundation/hardware-planning/README.md`.

---

## Prepare Disks

cephadm's `ceph orch apply osd --all-available-devices` will only consume disks that are clean — no partition table, no filesystem, no LVM signatures. Verify and wipe disks manually before bootstrap.

**List available block devices:**

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
```

Identify drives intended for OSDs. They should show no `FSTYPE` and no `MOUNTPOINT`. Drives with a filesystem, partition table, or LVM signature must be wiped.

**Wipe a disk (DESTRUCTIVE — all data on the device will be lost):**

> **WARNING:** `wipefs` and `sgdisk` permanently destroy all data and partition structures on the target device. Verify the device path multiple times before running. There is no undo.

```bash
# Wipe filesystem and partition-table signatures
wipefs --all --force /dev/sdX

# Zero the GPT and MBR structures
sgdisk --zap-all /dev/sdX

# Confirm the disk is clean
lsblk -o NAME,SIZE,TYPE,FSTYPE /dev/sdX
```

**Remove LVM signatures if present:**

```bash
# Check for LVM physical volumes
pvs
lvs
vgs

# Remove a volume group
vgremove vg_name

# Remove a physical volume
pvremove /dev/sdX

# Then wipe
wipefs --all --force /dev/sdX
sgdisk --zap-all /dev/sdX
```

**Verify the disk is clean before proceeding:**

```bash
file -s /dev/sdX            # should return "data" — no filesystem signature
parted /dev/sdX print       # should return "unrecognised disk label"
```

Repeat for each OSD disk on every node. Do not wipe the OS disk.

---

## Configure Kernel Parameters

Tune kernel parameters for Ceph OSD workloads. Apply on OSD nodes. Apply to MON/MGR nodes if they also run OSDs.

**Set parameters immediately and persist across reboots:**

```bash
cat > /etc/sysctl.d/90-ceph.conf << 'EOF'
# Reduce kernel swap tendency — OSD processes should not be paged out
vm.swappiness = 10

# Read-ahead for HDD OSDs — 2 MiB (4096 × 512 bytes) reduces seek overhead on large sequential reads
# Leave at default (128 KiB) for NVMe-only OSD nodes
# Set per-device after boot; this global value affects all block devices
# blockdev --setra 4096 /dev/sdX

# Maximum number of open file descriptors — each OSD opens many file handles
fs.file-max = 26234859

# inotify limits — cephadm and container orchestration watch many files
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 524288
EOF

sysctl --system
```

**For HDD OSDs, set per-device read-ahead at 2 MiB after boot:**

```bash
# Replace sdX, sdY, etc. with your HDD OSD device names
for dev in /dev/sdc /dev/sdd /dev/sde; do
  blockdev --setra 4096 ${dev}
done
```

To persist per-device read-ahead across reboots, create a udev rule:

```bash
cat > /etc/udev/rules.d/60-ceph-readahead.rules << 'EOF'
# Set 2 MiB read-ahead for HDD OSD devices
ACTION=="add|change", KERNEL=="sd[c-z]", ATTR{queue/rotational}=="1", \
  ATTR{bdi/read_ahead_kb}="2048"
EOF

udevadm control --reload-rules
```

**Increase open file limits for the ceph user:**

```bash
cat > /etc/security/limits.d/90-ceph.conf << 'EOF'
ceph soft nofile 1048576
ceph hard nofile 1048576
EOF
```

---

## SSH Key Distribution

cephadm uses SSH from the bootstrap host to add and manage all other cluster nodes. The cephadm-generated SSH key must be authorized on every node.

**Bootstrap will generate its own key.** However, if you want to pre-distribute keys (e.g., when adding nodes before or after bootstrap), follow this procedure.

**Generate an SSH key on the bootstrap host (if not already present):**

```bash
ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa -N ""
```

**Copy the public key to each additional host:**

```bash
ssh-copy-id -i /root/.ssh/id_rsa.pub root@<host2>
ssh-copy-id -i /root/.ssh/id_rsa.pub root@<host3>
ssh-copy-id -i /root/.ssh/id_rsa.pub root@<host4>
```

**Verify passwordless login works:**

```bash
ssh root@<host2> hostname
ssh root@<host3> hostname
```

**After `cephadm bootstrap` runs**, cephadm generates its own SSH key at `/etc/ceph/ceph.pub`. Distribute it to any hosts you want to add to the cluster:

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@<new-host>
```

Or using `ceph orch host add` with the `--addr` flag, cephadm will attempt SSH automatically if the key is already authorized.

**Non-root SSH user:** If using a non-root user, that user must have passwordless sudo. Pass `--ssh-user <user>` to `cephadm bootstrap`. The cephadm key will be added to `~<user>/.ssh/authorized_keys`.

---

## Verify Readiness

Run these checks on every node before bootstrapping. All should pass cleanly.

**1. Time synchronization:**

```bash
chronyc tracking | grep -E "Reference ID|System time|RMS offset"
# System time offset should be < 0.050 seconds
# Reference ID should not be 7F7F0101 (local clock fallback)
```

**2. Container runtime operational:**

```bash
# Podman
podman run --rm hello-world

# Docker
docker run --rm hello-world
```

**3. OSD disks are clean:**

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT | grep -v loop
# OSD devices should have no FSTYPE and no MOUNTPOINT

for dev in /dev/sdc /dev/sdd /dev/sde; do
  echo "=== ${dev} ==="
  file -s ${dev}
done
# All OSD devices should return "data"
```

**4. Network reachability between nodes:**

```bash
# From bootstrap host, ping all other nodes on both public and cluster networks
ping -c3 <host2-public-ip>
ping -c3 <host2-cluster-ip>
```

**5. SSH access from bootstrap to all other nodes:**

```bash
for host in host2 host3 host4; do
  ssh root@${host} "hostname && uptime" && echo "OK: ${host}" || echo "FAIL: ${host}"
done
```

**6. Python 3 and LVM2 present:**

```bash
python3 --version
lvmconfig --version 2>/dev/null || lvm version
```

**7. Hostname resolves correctly:**

```bash
hostname -f
# Must return the FQDN or short name that DNS/hosts resolves to this node's IP
getent hosts $(hostname)
```

**8. cephadm preflight check (after cephadm is installed):**

```bash
cephadm check-host
# All items should show OK or WARNING (not ERROR)
```

When all checks pass on all nodes, the cluster is ready for `cephadm bootstrap`.

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/cephadm/install.rst
  https://github.com/ceph/ceph/blob/main/doc/install/get-packages.rst
  https://github.com/ceph/ceph/blob/main/doc/start/os-recommendations.rst
-->
