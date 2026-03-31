# Network Topology — Flat vs Separated Networks

## Single vs separated network comparison

| Aspect | Single public network | Public + cluster network |
|---|---|---|
| NICs required per node | 1 (or 1 bonded pair) | 2 (or 2 bonded pairs) |
| Switch ports required | 1× per node | 2× per node |
| Replication traffic on client network | Yes | No — isolated to cluster network |
| Client-facing latency during recovery | Degraded (shares bandwidth) | Stable (replication on separate path) |
| Security | Replication traffic exposed to clients | Cluster network can be firewalled from clients |
| Configuration complexity | Minimal | Moderate |
| Suitable link speed | 25 GbE or faster per node | 10 GbE acceptable if replication is separated |
| Recommended for | Dev/test, small clusters, high-speed fabric | Production clusters, 10 GbE nodes, high client load |

## When separation matters

OSD replication generates roughly `(replication_factor - 1) × write_throughput` of internal traffic.
At `size=3` and 1 GB/s client writes, OSDs move ~2 GB/s of replication data among themselves.
On a single 10 GbE link this crowds out client traffic and creates latency spikes during recovery.

Heartbeat traffic (4 ports per OSD) also adds background overhead that grows with cluster size.

## Recommendation by cluster size

| Cluster size | Link speed | Recommendation |
|---|---|---|
| ≤ 10 nodes, test/dev | Any | Single network — simplicity outweighs benefit |
| ≤ 10 nodes, production | 25 GbE+ | Single network is acceptable; monitor utilisation |
| ≤ 10 nodes, production | 10 GbE | Separate networks — replication will saturate single link |
| 10–50 nodes | 25 GbE+ | Separated networks recommended |
| 10–50 nodes | 10 GbE | Separated networks required |
| > 50 nodes | Any | Separated networks required; consider bonding |

For resilience, always bond NICs active/active and connect to redundant switches.
Verify bond hash policy with your network team (layer 3+4 hashing, policy 2+3 or 3+4).

## Configuration commands

### Single public network

Add to `/etc/ceph/ceph.conf` `[global]` section:

```ini
[global]
public_network = 10.0.0.0/24
```

Or via the central config store (cephadm-managed clusters):

```bash
ceph config set global public_network 10.0.0.0/24
```

### Public + cluster (separated) network

Add to `/etc/ceph/ceph.conf`:

```ini
[global]
public_network  = 10.0.0.0/24
cluster_network = 192.168.0.0/24
```

Or via central config:

```bash
ceph config set global public_network  10.0.0.0/24
ceph config set global cluster_network 192.168.0.0/24
```

The cluster network must **not** be reachable from the public network or the internet.

### Per-OSD address override (mixed or single-NIC hosts)

```ini
[osd.0]
public_addr  = 10.0.0.10
cluster_addr = 192.168.0.10
```

### Firewall rules (iptables — replace variables with real values)

```bash
# Monitors — public network only, ports 3300 and 6789
iptables -A INPUT -i eth0 -p tcp -s 10.0.0.0/24 --dport 6789 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp -s 10.0.0.0/24 --dport 3300 -j ACCEPT

# OSDs — public network (client traffic)
iptables -A INPUT -i eth0 -m multiport -p tcp -s 10.0.0.0/24 --dports 6800:7568 -j ACCEPT

# OSDs — cluster network (replication + heartbeat)
iptables -A INPUT -i eth1 -m multiport -p tcp -s 192.168.0.0/24 --dports 6800:7568 -j ACCEPT
```

### Verify network assignment

```bash
# Confirm which addresses OSDs are using
ceph osd dump | grep -E "^osd\.[0-9]+ " | head -20

# Show public and cluster addresses per OSD
ceph osd find 0

# Check that replication is using cluster network (look for cluster_addr)
ceph osd metadata 0 | jq '{public_addr, cluster_addr}'
```

## Migration path (single → separated network)

Migrating an existing cluster to a separated network requires adding a cluster address
to each OSD without interrupting client traffic. Ceph daemons bind dynamically, so a
rolling restart is sufficient.

```bash
# Step 1 — Verify the new cluster-side interface is up on all hosts
# (confirm IP assignment on eth1 / bond1 before proceeding)

# Step 2 — Add cluster_network to ceph.conf or central config
ceph config set global cluster_network 192.168.0.0/24

# Step 3 — Rolling OSD restart to pick up new binding
# Repeat for each host; allow recovery to complete between hosts
ceph orch host maintenance enter node01
# Wait for cluster to stabilise:
watch ceph status
# Restart OSDs on this host:
ceph orch daemon restart osd   # targets OSDs scheduled on node01
ceph orch host maintenance exit node01

# Confirm OSDs picked up cluster_addr:
ceph osd metadata <id> | jq '.cluster_addr'

# Step 4 — Repeat for each node; verify replication traffic moves to cluster network
# Use iftop or bmon on both interfaces to confirm traffic separation
iftop -i eth0   # should show client traffic only
iftop -i eth1   # should show OSD-to-OSD replication traffic
```

Note: Monitor daemons always operate on the public network only.
No restart or reconfiguration of monitors is required when adding a cluster network.

<!-- source: https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/ — reviewed 2026-03-29 -->
