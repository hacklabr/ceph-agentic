# Kubernetes Prerequisites for Rook

This guide covers the Kubernetes cluster prerequisites required before deploying Rook-Ceph. Rook runs Ceph as containers inside Kubernetes, so the node, network, and storage configuration must be prepared in advance.

**When to use this guide:** before installing Rook on a new Kubernetes cluster or before expanding a Rook-Ceph deployment.

---

## Node Requirements

### Minimum cluster size

| Environment | Worker nodes | MONs | OSDs |
|---|---|---|---|
| Lab / test | 3 | 3 | 3 |
| Production | 5+ | 3 or 5 | 6+ |

MONs need stable identities and persistent storage. OSDs need raw, unformatted block devices or partitions.

### Per-node requirements

- Linux kernel `5.4` or newer (Ubuntu 20.04+, RHEL 8+, Rocky/Alma 8+).
- `lvm2` package installed on every node that will run OSDs.
- `ceph-common` and `rook-ceph` CSI driver prerequisites satisfied by Rook.
- Container runtime: containerd (recommended), CRI-O, or Docker.
- Sufficient resources:
  - OSD: 4–8 GB RAM per daemon, 1 CPU core per daemon, plus raw block devices.
  - MON: 2–4 GB RAM, 1 CPU core.
  - MGR: 2–4 GB RAM, 1 CPU core.

### Labels and taints

Label nodes that will host Ceph daemons:

```bash
kubectl label node <node> ceph-role=storage
```

Use taints/tolerations if you want dedicated storage nodes:

```bash
kubectl taint node <node> dedicated=storage:NoSchedule
```

Then configure Rook CRs with matching tolerations.

---

## Storage Requirements

### Devices for OSDs

Rook OSDs require raw, unformatted block devices or partitions. Rook will refuse to consume devices that have:

- A mounted filesystem
- An LVM signature
- A partition table (unless explicitly configured)
- Existing Ceph BlueStore metadata

Identify candidate devices:

```bash
lsblk -f
```

Expected: the device shows `FSTYPE` and `MOUNTPOINT` empty.

### Device types

- **HDD** — cheapest capacity, acceptable for most workloads.
- **SSD** — better IOPS and latency for RBD metadata pools and CephFS metadata.
- **NVMe** — best performance for high-IOPS RBD pools or DB/WAL devices.

### DB/WAL devices

For mixed HDD+NVMe clusters, use NVMe for BlueStore DB/WAL to reduce metadata latency:

```yaml
storage:
  config:
    metadataDevice: nvme0n1
```

---

## Network Requirements

### CNI

Any Kubernetes CNI is supported, but ensure:

- Pod-to-pod connectivity across all nodes works.
- Network policies allow Rook components to communicate.
- Host network is available for Rook OSDs if using host networking mode.

### Rook Network Modes

Rook supports two network modes:

| Mode | Description | Use when |
|---|---|---|
| `host` | Ceph daemons use host networking (default for many Rook deployments) | Simpler, direct node-to-node performance |
| `multus` | Ceph uses dedicated CNI networks (public and cluster) via Multus | You need isolated replication traffic |

Host networking is the simplest starting point. Multus is recommended for production clusters that want to isolate client and replication traffic.

---

## RBAC and Security

Rook requires a ServiceAccount, ClusterRole, and ClusterRoleBinding for the operator. These are created by the Rook deployment manifests. Do not reduce permissions unless you understand the impact.

### PSP / SCC

On clusters with Pod Security Policies or OpenShift SCCs, Rook needs privileged pods for OSDs and the ability to mount host devices. Use the provided Rook SCCs or a custom policy that allows:

- `privileged: true`
- `hostPath` volumes
- `hostNetwork` (if using host networking)
- `allowHostDirVolumePlugin: true`
- `allowHostPID: true` (for some Rook tooling)

---

## Kubernetes Version Compatibility

| Rook version | Kubernetes version | Ceph versions |
|---|---|---|
| 1.16+ | 1.28+ | Reef, Squid, Tentacle |

Check the Rook release notes for the exact Ceph image tags supported.

---

## Toolbox Pod

Rook provides a toolbox pod preconfigured with the Ceph CLI and admin keyring. Always use it for cluster commands instead of running `ceph` from your workstation.

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -it -- ceph status
```

The toolbox is deployed from the Rook examples repository:

```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.16/deploy/examples/toolbox.yaml
```

---

## Quick Validation

Before deploying Rook, verify:

```bash
# Nodes are ready
kubectl get nodes

# Candidate devices exist and are unformatted
kubectl get nodes -o json | jq '.items[].status.addresses[] | select(.type=="InternalIP") | .address'

# LVM is installed on storage nodes
kubectl get nodes -l ceph-role=storage -o name | xargs -I {} kubectl debug {} -it -- nsenter --target 1 -- bash -c "which lvm || echo 'lvm2 missing'"
```

---

<!-- Sources:
  https://rook.io/docs/rook/latest/Getting-Started/Prerequisites/prerequisites/
  https://rook.io/docs/rook/latest/Storage-Configuration/Cluster-design-considerations/considerations/
-->
