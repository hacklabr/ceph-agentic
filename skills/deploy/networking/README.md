# Cluster Networking

Ceph makes no use of request routing or proxy daemons — clients connect directly to OSDs, and OSDs replicate data directly to each other. Network design therefore has an outsized impact on cluster performance and resilience. A single public network works for many deployments (especially at 25 GbE or faster), but a dedicated cluster network offloads replication and heartbeat traffic from the client-facing path.

**Prerequisites:** Host interfaces must be up and configured before deploying any Ceph daemons. See `skills/foundation/host-preparation/README.md`.

---

## Network Architecture

Ceph supports two distinct networks:

| Network | Also called | Carries | Used by |
|---|---|---|---|
| **Public network** | front-side, client | Client I/O, MON traffic, OSD → client responses | MONs, MGRs, MDSs, OSDs (client-facing port) |
| **Cluster network** | back-side, replication, private | OSD heartbeat, object replication, recovery | OSDs only |

If no cluster network is declared, all traffic flows over the public network. This is acceptable but means replication and recovery storms compete directly with client I/O.

```
                           +-------------+
                           | Ceph Client |
                           +----*--*-----+
                                |  ^
                        Request |  : Response
                                v  |
 /------------------------------*--*-----------------------------------------\
 |                          Public Network  (10.0.0.0/24)                    |
 \---*--*-----------*--*-----------*--*---------*--*---------*--*------------/
     ^  ^           ^  ^           ^  ^         ^  ^         ^  ^
     |  |           |  |           |  |         |  |         |  |
  +--*--*--+    +---*--*---+   +---*--*---+  +--*--*---+  +--*--*---+
  |  MON   |    |  MON/MGR |   |   OSD    |  |   OSD   |  |   OSD   |
  +--------+    +----------+   +---*--*---+  +--*--*---+  +--*--*---+
                                   ^  ^          ^  ^          ^  ^
                                   |  |          |  |          |  |
 /------------------------------------*----------*--*----------*--*-----------\
 |                          Cluster Network  (10.0.1.0/24)                   |
 \---------------------------------------------------------------------------/
```

The cluster network should **not** be reachable from the public network or the internet.

---

## Configure Networks

### Set Public Network

Add `public_network` to the `[global]` section of `/etc/ceph/ceph.conf`. Use CIDR notation. Multiple subnets are allowed, separated by commas — all subnets must be routable to each other.

```ini
[global]
fsid = a3f8c1d2-7e4b-4a1f-8c9d-2b5e3f7a1c4d
mon_host = 10.0.0.10,10.0.0.11,10.0.0.12
public_network = 10.0.0.0/24
```

To pin individual daemons to specific addresses on the public network:

```ini
[osd.0]
public_addr = 10.0.0.20
```

### Set Cluster Network

Add `cluster_network` alongside `public_network`. OSDs will bind their replication port to an address on this subnet automatically.

```ini
[global]
fsid = a3f8c1d2-7e4b-4a1f-8c9d-2b5e3f7a1c4d
mon_host = 10.0.0.10,10.0.0.11,10.0.0.12
public_network  = 10.0.0.0/24
cluster_network = 10.0.1.0/24
```

To override the cluster address for a specific OSD (useful for single-NIC nodes in a dual-network cluster):

```ini
[osd.0]
public_addr  = 10.0.0.20
cluster_addr = 10.0.1.20
```

> **Single-NIC note:** If an OSD host has only one interface but the cluster has two networks, set `public_addr` in that OSD's config section and ensure the two networks can route to each other. This is a last resort — it reintroduces the security concern the cluster network was meant to solve.

Apply configuration changes with ceph config assimilate and regenerate a minimal config:

```bash
ceph config assimilate-conf < /etc/ceph/ceph.conf
ceph config generate-minimal-config > /etc/ceph/ceph.conf
```

Daemons bind dynamically — restart individual daemons (not the whole cluster) to pick up network changes:

```bash
ceph orch daemon restart osd.0
```

### Verify

Confirm OSDs report addresses on both networks:

```bash
ceph osd dump | grep "^osd\." | head -5
```

Expected output shows two addresses per OSD — one on the public network, one on the cluster network:

```
osd.0 up   in  weight 1 up_from 10 up_thru 42 down_at 0 last_clean_interval [0,0) \
  [v2:10.0.0.20:6800/1234,v1:10.0.0.20:6801/1234] \
  [v2:10.0.1.20:6800/1234,v1:10.0.1.20:6801/1234] \
  exists,up
```

Confirm MONs are listening on both v2 and v1 ports:

```bash
ceph mon dump
```

```
epoch 3
fsid a3f8c1d2-7e4b-4a1f-8c9d-2b5e3f7a1c4d
0: [v2:10.0.0.10:3300/0,v1:10.0.0.10:6789/0] mon.ceph01
1: [v2:10.0.0.11:3300/0,v1:10.0.0.11:6789/0] mon.ceph02
2: [v2:10.0.0.12:3300/0,v1:10.0.0.12:6789/0] mon.ceph03
```

---

## Messenger v2 (msgr2)

msgr2 is the second major revision of Ceph's wire protocol, default since Nautilus (14.x). It provides two connection modes and two port bindings per daemon.

### Ports

| Port | Protocol | Usage |
|---|---|---|
| `3300` | v2 (IANA-assigned, `0xce4`) | All new clients and daemons (Nautilus+) |
| `6789` | v1 (legacy) | Backward compatibility with pre-Nautilus clients |

MONs bind both ports simultaneously. Connecting peers negotiate the highest mutual version and use v2 if both sides support it.

### Connection Modes

| Mode | Auth | Encryption | Integrity | Use when |
|---|---|---|---|---|
| `crc` | cephx mutual auth | None | crc32c | Trusted internal network, maximum throughput |
| `secure` | cephx mutual auth | AES-GCM | Cryptographic | Untrusted network segments, compliance requirements |

AES-GCM is hardware-accelerated on modern CPUs (typically faster than SHA-256) — the throughput penalty for `secure` mode is small on recent hardware.

### Enable msgr2 on Monitors

msgr2 is enabled by default when monitors bind to port `6789`. If your cluster predates Nautilus or monitors use non-standard ports, explicitly enable it:

```bash
# Standard case — monitors on port 6789
ceph mon enable-msgr2

# Non-standard case — monitor mon.a is on port 1111, add v2 on 1112
ceph mon set-addrs a [v2:10.0.0.10:1112,v1:10.0.0.10:1111]
```

### Set Connection Mode Globally

```bash
# Use secure mode for all cluster-internal connections
ceph config set global ms_cluster_mode secure

# Use secure mode for all client-facing connections
ceph config set global ms_service_mode secure

# Use secure mode for client connections to monitors specifically
ceph config set global ms_mon_client_mode secure
```

Valid values: `crc`, `secure`, `prefer-crc`, `prefer-secure`.

### Verify msgr2 Status

```bash
ceph --admin-daemon /var/run/ceph/ceph-mon.ceph01.asok config show | grep ms_
```

Check which protocol version a live connection is using:

```bash
ceph daemon mon.ceph01 sessions
```

### Compression (msgr2 v2 only)

For multi-AZ or cloud deployments where inter-AZ bandwidth is expensive, enable compression on OSD replication traffic:

```bash
ceph config set osd ms_osd_compress_mode force
ceph config set osd ms_osd_compression_algorithm snappy
ceph config set osd ms_osd_compress_min_size 1024
```

`snappy` gives the best compression throughput ratio. Use `zstd` when ratio matters more than CPU overhead.

---

## Network Bonding

Bond at least two links per host for redundancy and throughput. The official recommendation is active/active bonding (LACP, mode 4) or a layer-3 multipath strategy.

**Hash policy matters for LACP:** Consult your network team. The wrong policy causes all flows to hash to the same member link. Use `layer3+4` (policy `2+3` in some notation) for TCP/UDP traffic. Verify with `bmon` or `iftop` that both member links carry traffic under load.

### netplan (Ubuntu 20.04+)

Create `/etc/netplan/01-ceph-bond.yaml` on each Ceph node:

```yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: false
    enp4s0:
      dhcp4: false

  bonds:
    bond0:
      interfaces: [enp3s0, enp4s0]
      parameters:
        mode: 802.3ad           # LACP active/active
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
      addresses: [10.0.0.10/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.1]

  vlans:
    bond0.100:                  # cluster network VLAN — see VLAN section
      id: 100
      link: bond0
      addresses: [10.0.1.10/24]
```

Apply without rebooting:

```bash
netplan apply
```

### nmcli (RHEL / Rocky / AlmaLinux)

```bash
# Create the bond
nmcli con add type bond \
  con-name bond0 \
  ifname bond0 \
  bond.options "mode=active-backup,miimon=100"

# For LACP instead of active-backup:
# bond.options "mode=802.3ad,miimon=100,xmit_hash_policy=layer3+4,lacp_rate=fast"

# Add member interfaces
nmcli con add type ethernet \
  con-name bond0-slave0 \
  ifname enp3s0 \
  master bond0

nmcli con add type ethernet \
  con-name bond0-slave1 \
  ifname enp4s0 \
  master bond0

# Assign address to bond
nmcli con mod bond0 \
  ipv4.addresses 10.0.0.10/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.method manual

nmcli con up bond0
```

Verify bond member status:

```bash
cat /proc/net/bonding/bond0
```

Look for `MII Status: up` on all slave interfaces. An interface showing `MII Status: down` is physically disconnected or has a port error on the switch.

---

## MTU / Jumbo Frames

Set MTU 9000 on the cluster network to reduce CPU overhead for large OSD replication writes. The public network typically stays at 1500 unless all clients also support jumbo frames end-to-end.

**The entire path must agree:** NIC, bond, switch port, and switch uplinks must all be configured to 9000. A single 1500 MTU hop silently fragments or drops oversized frames.

### Set MTU with netplan

```yaml
network:
  version: 2
  bonds:
    bond0:
      mtu: 9000
      interfaces: [enp3s0, enp4s0]
      parameters:
        mode: 802.3ad
        transmit-hash-policy: layer3+4
      addresses: [10.0.1.10/24]   # cluster network address
```

For the underlying member interfaces, netplan propagates MTU automatically. Apply:

```bash
netplan apply
ip link show bond0 | grep mtu
```

### Set MTU with nmcli

```bash
nmcli con mod bond0 ethernet.mtu 9000
nmcli con mod bond0-slave0 ethernet.mtu 9000
nmcli con mod bond0-slave1 ethernet.mtu 9000
nmcli con up bond0
```

### Verify End-to-End MTU

Use `ping` with the "do not fragment" flag and a payload size of 8972 bytes (9000 MTU − 28 bytes IP/ICMP overhead):

```bash
# Linux
ping -M do -s 8972 10.0.1.11

# If the path supports 9000 MTU, this succeeds.
# If any hop is misconfigured, you get:
#   ping: local error: message too long, mtu=1500
```

Run from each OSD host to every other OSD host on the cluster network. A failure between any pair means a misconfigured switch port or interface.

Check current MTU on all interfaces:

```bash
ip link show | grep -E "^[0-9]+:|mtu"
```

---

## VLAN Configuration

Use VLANs to carry public and cluster traffic on the same physical bond without a second set of NICs. This is common in environments with structured cabling constraints or where the cluster network traffic volume does not justify dedicated links.

VLAN tagging happens at the bond level. The switch port must be configured as a trunk carrying both VLANs.

### netplan VLAN example

Public network on untagged bond0, cluster network on VLAN 100:

```yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: false
    enp4s0:
      dhcp4: false

  bonds:
    bond0:
      interfaces: [enp3s0, enp4s0]
      parameters:
        mode: 802.3ad
        transmit-hash-policy: layer3+4
        mii-monitor-interval: 100
      mtu: 9000
      addresses: [10.0.0.10/24]   # public network

  vlans:
    bond0.100:
      id: 100
      link: bond0
      mtu: 9000
      addresses: [10.0.1.10/24]   # cluster network
```

### nmcli VLAN example

```bash
# Create VLAN 100 on top of bond0
nmcli con add type vlan \
  con-name bond0.100 \
  ifname bond0.100 \
  dev bond0 \
  id 100 \
  ipv4.addresses 10.0.1.10/24 \
  ipv4.method manual \
  ethernet.mtu 9000

nmcli con up bond0.100
```

Set `cluster_network = 10.0.1.0/24` in `ceph.conf` to direct OSD replication traffic onto the VLAN interface.

---

## Firewall Rules

Ceph daemons bind to dynamic ports in the range `6800:7568` in addition to the fixed MON ports. Open the full range on both public and cluster network interfaces.

### Port reference

| Daemon | Network | Port(s) | Protocol |
|---|---|---|---|
| MON | Public | 3300 (v2), 6789 (v1) | TCP |
| MGR / Dashboard | Public | 8443 (dashboard), 9283 (Prometheus) | TCP |
| MGR / other modules | Public | 6800–7568 | TCP |
| MDS | Public | 6800–7568 | TCP |
| OSD (client-facing) | Public | 6800–7568 | TCP |
| OSD (replication) | Cluster | 6800–7568 | TCP |
| OSD (heartbeat) | Both | 6800–7568 | TCP |

Each OSD uses up to four ports: client+monitor, data replication, and two heartbeat ports (one per network). The dynamic range handles daemon restarts that pick up a new port within the window.

### iptables

```bash
# Public network interface — MON ports
iptables -A INPUT -i eth0 -p tcp -s 10.0.0.0/24 --dport 6789 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp -s 10.0.0.0/24 --dport 3300 -j ACCEPT

# Public network interface — daemon range (MGR, MDS, OSD client-facing)
iptables -A INPUT -i eth0 -m multiport -p tcp -s 10.0.0.0/24 --dports 6800:7568 -j ACCEPT

# Dashboard and Prometheus (restrict to management subnet)
iptables -A INPUT -i eth0 -p tcp -s 10.0.0.0/24 --dport 8443 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp -s 10.0.0.0/24 --dport 9283 -j ACCEPT

# Cluster network interface — OSD replication and heartbeat
iptables -A INPUT -i eth1 -m multiport -p tcp -s 10.0.1.0/24 --dports 6800:7568 -j ACCEPT

# Save rules (RHEL/Rocky)
iptables-save > /etc/sysconfig/iptables

# Save rules (Debian/Ubuntu)
iptables-save > /etc/iptables/rules.v4
```

### firewalld

```bash
# Add Ceph services to the public zone
firewall-cmd --zone=public --add-port=3300/tcp --permanent
firewall-cmd --zone=public --add-port=6789/tcp --permanent
firewall-cmd --zone=public --add-port=6800-7568/tcp --permanent
firewall-cmd --zone=public --add-port=8443/tcp --permanent
firewall-cmd --zone=public --add-port=9283/tcp --permanent

# Assign the cluster network interface to an internal zone
firewall-cmd --zone=internal --add-interface=bond0.100 --permanent
firewall-cmd --zone=internal --add-port=6800-7568/tcp --permanent

firewall-cmd --reload
firewall-cmd --list-all --zone=public
firewall-cmd --list-all --zone=internal
```

> **Container note:** Docker and Podman insert their own iptables rules. Reload firewalld rules serially: set maintenance mode, stop container services, apply rule changes, restart container services, exit maintenance mode. Reloading rules while containers are running can break in-flight connections.

---

## Troubleshoot Network Issues

### Decision tree: slow operations

```
Slow ops or high op latency?
│
├─ Check overall cluster network bandwidth
│    ceph osd perf
│    # Look for high commit_latency_ms (> 50ms on HDD, > 5ms on NVMe = investigate)
│
├─ Check for MTU mismatch
│    ping -M do -s 8972 <osd-cluster-ip>
│    # Failure → find the misconfigured switch port or NIC
│    ip link show | grep mtu
│
├─ Check bond member utilization
│    bmon -p bond0
│    iftop -i bond0
│    # One member at 100%, other idle → wrong transmit hash policy
│    cat /proc/net/bonding/bond0
│
├─ Check msgr2 errors
│    ceph daemon osd.0 perf dump | grep ms_
│    # ms_connection_rejected → auth or version mismatch
│    # ms_bad_message → MTU or corruption
│
├─ Check OSD network interface errors
│    ethtool -S enp3s0 | grep -E "error|drop|miss"
│    # Non-zero rx_errors → bad cable, SFP, or switch port
│
└─ Check OSD heartbeat timeouts
     ceph health detail | grep OSD_SLOW_PING
     # Threshold: osd_heartbeat_grace = 20s (default)
     # Adjust: ceph config set osd osd_heartbeat_grace 30
```

### Useful diagnostic commands

Check which network each OSD is using for replication:

```bash
ceph osd dump | grep "^osd\." | awk '{print $1, $17, $18}'
```

Show live messenger statistics for a daemon:

```bash
ceph daemon osd.0 perf dump | python3 -m json.tool | grep -A2 '"ms_'
```

Measure raw TCP throughput between two OSD hosts on the cluster network (requires `iperf3` installed):

```bash
# On the destination OSD host (10.0.1.11):
iperf3 -s

# On the source OSD host:
iperf3 -c 10.0.1.11 -t 30 -P 4
# -P 4 uses 4 parallel streams to saturate a bond
```

Expected throughput: ~940 Mbps per 1 GbE link, ~9.4 Gbps per 10 GbE link. Significantly lower throughput points to a duplex mismatch, bond hash imbalance, or congestion.

Check for dropped packets on the cluster network interface:

```bash
watch -n 2 'ip -s link show bond0.100'
```

Non-zero `dropped` in the RX column on a VLAN interface often means the switch is not tagging frames with that VLAN ID, or the trunk port is misconfigured.

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/rados/configuration/network-config-ref.rst
  https://github.com/ceph/ceph/blob/main/doc/rados/configuration/msgr2.rst
-->
