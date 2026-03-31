# Compatibility Matrix

All data sourced from the official Ceph OS Recommendations page and release notes (Reef 18.2, Squid 19.2, Tentacle 20.2). See the reference comment at the bottom for source links.

---

## Ceph Version x Operating System

Support tiers:

- **A** — Packages provided; comprehensive testing performed.
- **B** — Packages provided; basic testing performed.
- **C** — Packages provided; no testing performed.
- **D** — Client-only packages from an external site; not maintained or tested by the core Ceph team.
- **—** — Not supported / no packages provided.

| Operating System | Reef 18.2.z | Squid 19.2.z | Tentacle 20.2.z |
|------------------|:-----------:|:------------:|:---------------:|
| CentOS 9 Stream  | A           | A            | A               |
| Ubuntu 22.04 LTS | A           | A            | A               |
| Ubuntu 20.04 LTS | A           | —            | —               |
| Debian 12        | C           | C            | C               |
| MS Windows       | D           | D            | D               |

> **Rocky Linux / RHEL 9** are binary-compatible with CentOS 9 Stream. Ceph packages built for CentOS 9 install and run on Rocky Linux 9 and RHEL 9 without modification. No separate tier rating exists in upstream docs; treat them as equivalent to the CentOS 9 row above.

> **Ubuntu 20.04** support was dropped after Reef. Do not run Squid or Tentacle on Ubuntu 20.04.

### Container Hosts

These operating systems are tested as container hosts for Ceph's official cephadm-managed deployments:

| Container Host   | Reef 18.2.z | Squid 19.2.z | Tentacle 20.2.z |
|------------------|:-----------:|:------------:|:---------------:|
| CentOS 9 Stream  | H           | H            | H               |
| Ubuntu 22.04 LTS | H           | H            | H               |

**H** — Ceph tests this distribution as a container host.

---

## Ceph Version x Linux Kernel

Kernel requirements apply primarily to the **kernel client** (RBD `map` and CephFS kernel mount). Userspace clients (FUSE, librados, librbd, ceph-fuse) have no hard kernel minimum but benefit from more recent kernels for performance.

| Ceph Version    | Minimum Kernel (RBD client) | Recommended Kernel | Notes |
|-----------------|-----------------------------|--------------------|-------|
| Reef (18.2.z)   | 4.19 (LTS)                  | 5.15+ LTS or distro LTS | RBD image features require ≥ 5.3 or CentOS 8.2 kernel |
| Squid (19.2.z)  | 4.19 (LTS)                  | 5.15+ LTS or distro LTS | Same RBD feature floor as Reef |
| Tentacle (20.2.z) | 4.19 (LTS)                | 6.1+ LTS or distro LTS  | Container images based on CentOS 9; older kernels may have thread-creation incompatibilities |

Key guidance from official docs:

- Use a kernel from the "stable" or "longterm maintenance" series at [kernel.org](https://www.kernel.org) or your distro vendor.
- For RBD block device features (encryption, cloning, migration), kernel **5.3** is the functional floor. The CentOS 8.2 kernel (4.18-based with backports) meets this floor.
- For CephFS kernel mounts, consult the [Mounting CephFS using Kernel Driver](https://docs.ceph.com/en/latest/cephfs/mount-using-kernel-driver/#which-kernel-version) page for per-feature kernel requirements.
- Container images for Reef ≥ 18.2.4 are based on CentOS 9. Running these images on very old host kernels (e.g., Ubuntu 18.04 kernel) can cause failures due to differences in thread creation.

---

## Container Images

Official Ceph container images are hosted on **quay.io** (primary) and mirrored on Docker Hub.

| Ceph Version    | Primary Registry  | Image Path           | Stable Tag Pattern     | Example |
|-----------------|-------------------|----------------------|------------------------|---------|
| Reef (18.2.z)   | quay.io           | `quay.io/ceph/ceph`  | `v<major>.<minor>.<patch>` | `quay.io/ceph/ceph:v18.2.8` |
| Squid (19.2.z)  | quay.io           | `quay.io/ceph/ceph`  | `v<major>.<minor>.<patch>` | `quay.io/ceph/ceph:v19.2.3` |
| Tentacle (20.2.z) | quay.io         | `quay.io/ceph/ceph`  | `v<major>.<minor>.<patch>` | `quay.io/ceph/ceph:v20.2.0` |

Additional component images:

| Component  | Registry Path                  | Notes |
|------------|-------------------------------|-------|
| Grafana    | `quay.io/ceph/grafana`        | Dashboard monitoring; loaded into container at runtime (Squid+) |
| Pre-release / CI | `quay.ceph.io/ceph-ci/ceph` | Testing images only; not for production |

Usage with cephadm:

```bash
# Target a specific point release
ceph orch upgrade start --image quay.io/ceph/ceph:v19.2.3

# Check upgrade status
ceph orch upgrade status
```

> Always pin to an explicit `vX.Y.Z` tag in production. Floating `latest` or `stable` tags are not provided by the official registry.

---

## Client Compatibility

Ceph supports **rolling upgrades** and maintains backward compatibility with clients from the **two most recent stable release series**.

| Cluster Version | Minimum Compatible Client | Supported Client Versions | Notes |
|-----------------|--------------------------|--------------------------|-------|
| Reef (18.2.z)   | Pacific (16.2.z)          | Pacific, Quincy, Reef    | Upgrade cluster from Pacific (16.2.z) or Quincy (17.2.z) before upgrading to Reef |
| Squid (19.2.z)  | Quincy (17.2.z)           | Quincy, Reef, Squid      | Upgrade cluster from Quincy (17.2.z) or Reef (18.2.z) before upgrading to Squid |
| Tentacle (20.2.z) | Reef (18.2.z)           | Reef, Squid, Tentacle    | Must upgrade to Reef or Squid before upgrading to Tentacle |

> **Important:** Ceph does not support skipping two or more release series in a single upgrade. Example: upgrading from Pacific directly to Squid is unsupported. You must pass through Quincy or Reef first.

> **Pacific to Reef warning:** A bug was discovered in Reef v18.2.8 QA that makes direct Pacific-to-Reef upgrades potentially unreliable. Upgrading via Quincy (17.2.z) is now the recommended path when coming from Pacific.

### OSD Compatibility Gate

After a full cluster upgrade, lock in the new release to prevent old-version OSDs from rejoining:

```bash
# After all OSDs have been upgraded to Reef
ceph osd require-osd-release reef

# After all OSDs have been upgraded to Squid
ceph osd require-osd-release squid
```

---

## Python Requirements

Ceph's management plane (cephadm, mgr modules, ceph-volume) requires Python 3. Specific version floors per distribution:

| Ceph Version    | Minimum Python | Typical Distro Python  | Notes |
|-----------------|---------------|------------------------|-------|
| Reef (18.2.z)   | Python 3.6    | CentOS 9: 3.9 / Ubuntu 22.04: 3.10 | ceph-volume uses `importlib` from stdlib (available 3.8+); validated against 3.11 |
| Squid (19.2.z)  | Python 3.6    | CentOS 9: 3.9 / Ubuntu 22.04: 3.10 | Required Python dependencies available in EPEL9 for CentOS/RHEL 9 |
| Tentacle (20.2.z) | Python 3.8  | CentOS 9: 3.9 / Ubuntu 22.04: 3.10 | Python `datetime` APIs now return timezone-aware objects; code calling these APIs must handle `tzinfo` |

> Python 2 has been unsupported since the Nautilus release (14.2.z). Do not install Ceph on hosts where Python 2 is the system default without an explicit Python 3 alternative.

> The `restful` and `zabbix` mgr modules (deprecated since 2020) were removed in Tentacle. Any automation that invokes these modules must be updated.

---

<!-- Sources (verified March 2026):
  - OS Recommendations: https://github.com/ceph/ceph/blob/main/doc/start/os-recommendations.rst
  - Reef release notes:  https://github.com/ceph/ceph/blob/main/doc/releases/reef.rst
  - Squid release notes: https://github.com/ceph/ceph/blob/main/doc/releases/squid.rst
  - Tentacle release notes: https://github.com/ceph/ceph/blob/main/doc/releases/tentacle.rst
  - General release cycle: https://github.com/ceph/ceph/blob/main/doc/releases/general.rst
-->
