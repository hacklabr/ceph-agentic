# Hardware Planning

Plan Ceph hardware before deploying. Wrong choices here — RAM too thin, no SSD for MON, HDD without WAL offload — cause performance and stability problems that are expensive to fix after the fact.

---

## Minimum Cluster Sizes

A Ceph cluster requires an odd number of MON daemons for Paxos quorum. Quorum requires a majority: `floor(n/2) + 1`.

| Configuration | MON Count | Failure Tolerance | Use Case |
|---------------|-----------|------------------|----------|
| Minimum viable | 3 | 1 MON failure | Dev/test only |
| Small production | 3 | 1 MON failure | Clusters up to ~50 OSDs |
| Production standard | 5 | 2 MON failures | Clusters 50–500 OSDs |
| Large / multi-site | 5–7 | 2–3 MON failures | 500+ OSDs or stretch clusters |

**Minimum OSD count:** 3 OSDs across 3 separate hosts to satisfy the default 3-replica pool with `host` failure domain. 9 OSDs (3 per host) is the practical minimum for a production deployment.

Distribute Ceph daemons of a given type on hosts configured for that type. Do not co-locate OSD workloads with compute (OpenStack Nova, Kubernetes worker nodes, etc.).

---

## OSD Node Sizing

### CPU

OSD nodes must run RADOS, compute CRUSH data placement, replicate data, and maintain cluster map copies. The useful metric is **IOPS per core**, not cores per OSD.

| Media Type | Minimum Threads per OSD | Recommended Threads per OSD |
|------------|-------------------------|-----------------------------|
| HDD | 1 | 3 |
| SSD (SATA/SAS) | 2 | 4 |
| NVMe | 4 | 6 |

"Threads" means logical CPU threads (hyperthreads when HT is enabled). HT is almost always beneficial for Ceph OSD workloads. NVMe OSDs in production clusters typically utilize 5–6 cores per OSD; in isolated single-OSD benchmarks they can saturate up to 14 cores.

ARM processors may require more cores per OSD than x86 for equivalent performance.

MON and MGR daemons have modest CPU requirements. Run non-Ceph CPU-intensive processes on separate hosts to avoid resource contention.

### RAM

BlueStore manages its own memory independently of the kernel page cache. The key parameter is `osd_memory_target` (default: 4 GiB per OSD).

**OSD RAM guidelines:**

| `osd_memory_target` | Effect |
|--------------------|--------|
| Below 2 GiB | Not recommended — likely OOM or severe performance degradation |
| 2–4 GiB | Functional but degraded — metadata may require disk reads during IO |
| 4 GiB | Default — suitable for typical mixed workloads |
| 6 GiB+ | Recommended for HDD OSDs to mitigate slow requests |
| Higher | Beneficial for many small objects or datasets ≥256 GiB per OSD; especially effective with NVMe |

**Server RAM formula:** Total server RAM must exceed `(number_of_OSDs × osd_memory_target × 2)`. The 2× multiplier accounts for OS overhead, other co-located daemons, and recovery spikes.

Example: A 1U server with 10 OSDs at 4 GiB target needs at least 80 GiB — provision 128 GiB.

Budget at least 20% extra headroom above calculated targets to prevent OSDs from hitting OOM during temporary spikes or delayed kernel page reclaim.

Enable `osd_memory_target_autotune` on nodes that may host non-OSD daemons under varying load. Do not configure swap for OSD hosts — a crashing OSD is preferable to one that slows to a crawl.

### Disks

**One OSD per physical drive.** Never provision multiple OSDs on a single HDD. PCIe Gen 4+ NVMe drives larger than 30 TB may benefit from being split into two or more OSDs.

**Minimum OSD drive size:** 1 TiB. Drives smaller than 100 GiB are not effective. Drives smaller than 1 TiB spend a disproportionate fraction of capacity on metadata.

**Separate the OS drive from OSD drives.** Co-locating the OS and OSD data on the same drive causes slow OSD issues.

| Media | Use Case | Notes |
|-------|----------|-------|
| HDD (SATA/SAS) | Capacity-optimized bulk storage | Best under ~8 TB per drive; larger drives face SATA interface bottlenecks |
| SATA/SAS SSD | Performance OSDs on moderate budgets | Enterprise-class only; 1 DWPD rating sufficient for most OSD workloads |
| NVMe SSD | High-performance OSDs | No HBA needed; dramatically reduces cost gap vs HDD at system level |
| NVMe SSD (QLC) | High-density at lower $/GB | Closing cost gap with HDD; lower power consumption |

**HDD notes:**
- Drives above 8 TB are best suited for large sequential or read-mostly workloads due to SATA interface saturation
- Dense chassis with 24–100 drives use SAS expanders that can become bottlenecks
- SAS/SATA SSDs are disappearing from vendor roadmaps; plan NVMe-native chassis for long-term flexibility

**Disk controllers:** Use IT-mode (JBOD) HBAs, not RAID-mode. RAID SoC and cache add latency and cost. For NVMe, no HBA is needed at all — eliminating HBA cost partially offsets the NVMe price premium.

**Write caches:** Disable the volatile write cache on HDD OSDs to improve IOPS and reduce commit latency. BlueStore opens devices with `O_DIRECT` and issues `fsync()` frequently; enterprise drives with power loss protection (PLP) handle this correctly.

**WAL/DB offload onto fast media:**

Placing the BlueStore RocksDB metadata (block.db) and write-ahead log (block.wal) on a fast device while keeping object data on HDD materially improves HDD OSD performance.

| Fast Media Type | HDD OSDs per Device |
|-----------------|---------------------|
| SATA/SAS SSD | 4–5 HDD OSDs per DB/WAL SSD |
| NVMe SSD | Up to 15 HDD OSDs per DB/WAL NVMe |

If fast media is limited, prioritize a DB device over WAL-only — the DB device provides greater benefit.

---

## MON/MGR Node Sizing

MON daemons require stable, low-latency storage. MON database performance directly affects cluster recovery speed and stability.

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4+ cores |
| RAM (small cluster, ≤50 OSDs) | 32 GiB | 64 GiB |
| RAM (medium cluster, 50–300 OSDs) | 64 GiB | 64–128 GiB |
| RAM (large cluster, 300+ OSDs) | 128 GiB | 128+ GiB |
| Storage per MON daemon | 100 GiB | 256 GiB+ SSD |
| Storage media | SSD strongly urged | Enterprise NVMe |

MON and MGR memory usage scales with cluster size. Provision for peak usage at boot time, during topology changes, and during recovery — not just steady-state.

**Co-location rules:**
- Small clusters (≤50 OSDs): MON + MGR can co-reside on the same 3 nodes as OSD daemons, provided those nodes have sufficient CPU and RAM
- Medium clusters (50–300 OSDs): Dedicate separate MON/MGR nodes
- Large clusters (300+ OSDs): Dedicated management nodes; consider running RGW on MON/MGR nodes if resources allow

MGR daemons: deploy 1 active + 1 standby minimum. 1 MGR per MON node is a common pattern. MGR port 8443 (dashboard), 9283 (Prometheus).

Provision enterprise-class SSDs for MON storage. The MON RocksDB database is write-intensive during cluster changes. A 1 DWPD rating is sufficient; drives with PLP are required.

When provisioning a single SSD for both OS boot and MON/MGR purposes, minimum 256 GiB; 960 GiB recommended. Enterprise-class drives only — consumer SSDs are not suitable for production MON nodes.

---

## Network Requirements

### Topology

Provision two networks where possible:

| Network | Traffic | Interface |
|---------|---------|-----------|
| **Public (front-side)** | Client reads/writes + MON communication | 10 GbE minimum; 25 GbE for production |
| **Cluster (back-side)** | OSD replication, recovery, heartbeat | 10 GbE minimum; 25 GbE for production |

A single public network functions correctly in many deployments, especially at 25 GbE or faster. A dedicated cluster network reduces recovery impact on client traffic and is recommended for workloads with high replication traffic.

Cluster network must not be reachable from the public network or internet.

### Bandwidth

| Workload | Minimum | Recommended |
|----------|---------|-------------|
| Small cluster, HDD OSDs | 1 GbE | 10 GbE |
| Production cluster, HDD OSDs | 10 GbE | 25 GbE |
| Dense nodes or NVMe OSDs | 25 GbE | 100 GbE |
| Aggregate OSD throughput near NIC saturation | Bond multiple links | 25–100 GbE per node |

Dense or NVMe nodes can saturate 10 GbE or 25 GbE interfaces under replication load. Use active/active link bonding across redundant switches. Bonding hash policy must distribute traffic across links (typically layer 3+4 hashing; consult your network team for LACP policy).

Top-of-rack switches need fast uplinks to core/spine, typically 40 GbE minimum.

Recovery throughput context:
- 1 TiB replicated across 1 GbE = 3 hours
- 1 TiB replicated across 10 GbE = 20 minutes
- 10 TiB replicated across 1 GbE = 30 hours
- 10 TiB replicated across 10 GbE = 3 hours

Faster recovery means shorter windows of elevated risk when components fail.

### Port Requirements

| Daemon | Ports | Protocol | Notes |
|--------|-------|----------|-------|
| MON | 3300 | TCP | msgr2 (default since Nautilus) |
| MON | 6789 | TCP | msgr1 (legacy clients) |
| OSD | 6800–7568 | TCP | Dynamic; each OSD uses up to 4 ports (client, replication, 2× heartbeat) |
| MGR | 6800–7568 | TCP | Dynamic; first available port from 6800 |
| MGR Dashboard | 8443 | TCP | HTTPS |
| MGR Prometheus | 9283 | TCP | Metrics endpoint |
| MDS | 6800–7568 | TCP | Dynamic |
| RGW | 80 | TCP | HTTP (cephadm default); legacy standalone default is 7480 |
| RGW | 443 | TCP | HTTPS (when TLS configured) |

Open the full `6800:7568` range for OSD, MGR, and MDS on both public and cluster network interfaces. When a daemon fails and restarts without releasing its port, the restarted daemon binds to the next available port in the range.

---

## Node Role Planning

### Small Cluster (3 nodes, up to ~30 OSDs)

| Role | Count | CPU | RAM | Disks | Network |
|------|-------|-----|-----|-------|---------|
| MON + MGR + OSD | 3 | 2× 10-core | 128 GiB | 1× OS SSD + 10× HDD + 1× NVMe (WAL/DB) | 2× 10 GbE bonded |

Co-locate MON, MGR, and OSD daemons on the same 3 nodes. Dedicated OS drive is mandatory. NVMe for WAL/DB offload of HDD OSDs strongly recommended.

### Medium Cluster (5–10 nodes, 30–200 OSDs)

| Role | Count | CPU | RAM | Disks | Network |
|------|-------|-----|-----|-------|---------|
| MON + MGR | 3–5 | 2× 8-core | 128 GiB | 2× 960 GiB SSD (OS + MON store) | 2× 25 GbE bonded |
| OSD | 5–10 | 2× 16-core | 256–512 GiB | 1× OS SSD + 12–24× HDD + 2× NVMe (WAL/DB) | 2× 25 GbE bonded |

Dedicated management nodes. OSD nodes sized for ~24 drives per node.

### Large Cluster (10+ nodes, 200+ OSDs)

| Role | Count | CPU | RAM | Disks | Network |
|------|-------|-----|-----|-------|---------|
| MON + MGR | 5 | 2× 16-core | 256 GiB | 2× 1.6 TiB NVMe (OS + MON store) | 2× 25 GbE bonded |
| OSD (HDD) | 10+ | 2× 24-core | 512 GiB | 1× OS SSD + 24× HDD + 2× NVMe (WAL/DB) | 2× 100 GbE bonded |
| OSD (NVMe) | 10+ | 2× 32-core | 512 GiB | 1× OS SSD + 8–12× NVMe | 2× 100 GbE bonded |
| RGW (optional) | 2+ | 2× 8-core | 64 GiB | 2× 480 GiB SSD | 2× 25 GbE bonded |
| MDS (CephFS only) | 2+ | 2× 8-core (high GHz) | 64 GiB | 2× 480 GiB SSD | 2× 25 GbE bonded |

MDS is CPU-frequency sensitive (single-threaded metadata operations). Prioritize clock speed over core count.

---

## Disk Layout Examples

### Example A: Standard Capacity OSD Node (HDD with NVMe WAL/DB)

**Target:** 12 HDD OSDs per node, NVMe for WAL/DB offload.

```
/dev/sda              240 GiB SATA SSD   → OS (mirrored pair with /dev/sdb)
/dev/sdb              240 GiB SATA SSD   → OS mirror
/dev/sdc–/dev/sdn     12× 8 TiB HDD      → OSD data (osd.0–osd.11)
/dev/nvme0n1          2× 960 GiB NVMe    → WAL/DB for osd.0–osd.5
/dev/nvme1n1          2× 960 GiB NVMe    → WAL/DB for osd.6–osd.11
```

- Total raw capacity per node: 96 TiB HDD
- NVMe DB partition per HDD OSD: ~160 GiB (960 GiB ÷ 6 OSDs, with headroom)
- Use IT-mode SAS HBA for HDD backplane; NVMe connects directly to PCIe

### Example B: All-NVMe High-Performance OSD Node

**Target:** 8 NVMe OSDs per node, no WAL/DB separation needed.

```
/dev/sda              480 GiB SATA SSD   → OS (mirrored pair)
/dev/sdb              480 GiB SATA SSD   → OS mirror
/dev/nvme0n1–nvme7n1  8× 3.84 TiB NVMe  → OSD data (osd.0–osd.7)
```

- Total raw capacity per node: 30.72 TiB NVMe
- No HBA required — NVMe connects to PCIe directly
- BlueStore colocates WAL+DB on each NVMe drive
- Provision 6 CPU threads per OSD minimum (48 threads per node)
- Provision 256–512 GiB RAM (32 GiB per OSD × 8 + OS overhead)

### Example C: MON/MGR Dedicated Node

**Target:** Management-only node; no OSD workloads.

```
/dev/nvme0n1          960 GiB NVMe       → OS + MON store (partition: 200 GiB OS, 700 GiB MON)
/dev/nvme1n1          960 GiB NVMe       → MGR and log storage
```

- 2 NVMe drives, no HDD
- 128 GiB RAM for clusters up to 300 OSDs; 256 GiB for larger
- 2× 25 GbE bonded — MON traffic is modest but low-latency requirement is high
- No HBA; no expander; minimal complexity

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/start/hardware-recommendations.rst
  https://github.com/ceph/ceph/blob/main/doc/rados/configuration/network-config-ref.rst
-->
