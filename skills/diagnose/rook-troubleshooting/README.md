# Troubleshooting Rook on Kubernetes

This guide provides symptom-based troubleshooting for Rook-Ceph clusters running on Kubernetes. Always start with the toolbox for `ceph` commands and use `kubectl` for Kubernetes-level issues.

**When to use this guide:** Rook pods are not running, Ceph health is not OK, PVCs are not provisioning, or workloads cannot access storage.

---

## Start Here

Run these commands in order:

```bash
# 1. Check Rook namespace pods
kubectl -n rook-ceph get pod

# 2. Check CephCluster status
kubectl -n rook-ceph get cephcluster rook-ceph -o yaml

# 3. Check Ceph health via toolbox
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status

# 4. Check recent events
kubectl -n rook-ceph get events --sort-by='.lastTimestamp'
```

---

## Rook Operator Issues

### Operator pod not running

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-operator
kubectl -n rook-ceph describe pod -l app=rook-ceph-operator
kubectl -n rook-ceph logs -l app=rook-ceph-operator --tail=100
```

Common causes:

- Missing RBAC (ClusterRole/ClusterRoleBinding).
- Incorrect `ROOK_CURRENT_NAMESPACE_ONLY` setting.
- Image pull failure.

### CephCluster CR not reconciling

Check the operator logs for errors parsing the CephCluster CR:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-operator --tail=200 | grep -i error
```

Validate the CR with `kubectl apply --dry-run=server`.

---

## MON Issues

### MON pods stuck in Pending

```bash
kubectl -n rook-ceph describe pod -l app=rook-ceph-mon
```

Common causes:

- Insufficient CPU/RAM on nodes.
- Taints without tolerations.
- PVC for MON data cannot be provisioned (if using PVC-based MONs).

### MON quorum lost

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph mon stat
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph quorum_status
```

If quorum is lost:

1. Do **not** delete MON pods arbitrarily.
2. Identify which MONs are running and healthy.
3. Check network connectivity between MON pods.
4. If a MON pod is unhealthy, scale the MON count down temporarily only if you understand the quorum math.

---

## OSD Issues

### OSD pods not created

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-operator --tail=200 | grep -i osd
```

Common causes:

- Device is not unformatted.
- `lvm2` is not installed on the node.
- The node name in the CephCluster CR does not match `kubectl get node`.

### OSD pod CrashLoopBackOff

```bash
kubectl -n rook-ceph logs <osd-pod-name> --tail=100
```

Look for:

- Disk errors (`EIO`, `fsck`, `corruption`).
- OOM kills (increase memory limits).
- Missing `ceph.conf` or keyring (operator issue).

### Check device eligibility on a node

```bash
kubectl debug node/<node-name> -it -- nsenter --target 1 -- bash
lsblk -f
lvs
pvs
```

A device eligible for Rook must have no filesystem, no partition table, and no LVM signature.

---

## CSI Issues

### PVC stuck Pending

```bash
kubectl describe pvc <pvc-name>
```

Check the CSI provisioner logs:

```bash
kubectl -n rook-ceph logs -l app=csi-rbdplugin-provisioner --tail=200
kubectl -n rook-ceph logs -l app=csi-cephfsplugin-provisioner --tail=200
```

Common causes:

- StorageClass `clusterID` or pool name mismatch.
- Missing CSI secrets.
- Ceph health not OK.

### Pod cannot mount volume

```bash
kubectl describe pod <pod-name>
```

Check the CSI node plugin on the worker node:

```bash
kubectl -n rook-ceph logs -l app=csi-rbdplugin --field-selector spec.nodeName=<node-name> --tail=200
```

Common causes:

- `ceph-common` utilities missing (Rook CSI containers should include them).
- Network policy blocking CSI socket.
- Stale volume attachments.

---

## Common Ceph Health Warnings in Rook

| Symptom | Likely cause | Action |
|---|---|---|
| `HEALTH_WARN` `MON_CLOCK_SKEW` | Node NTP drift | Sync time on all Kubernetes nodes. |
| `HEALTH_WARN` `OSD_DOWN` | OSD pod failing or node down | Check OSD pod logs and node status. |
| `HEALTH_WARN` `OSD_NEARFULL` | Cluster capacity low | Add devices or OSDs. |
| `HEALTH_ERR` `POOL_FULL` | Pool quota reached | Increase quota or delete data. |
| `HEALTH_WARN` `TOO_FEW_PGS` | PG count too low | Enable `pg_autoscaler` or increase `pg_num`. |

---

## Useful Commands

```bash
# Rook cluster status
kubectl -n rook-ceph get cephcluster rook-ceph

# All Rook CRs
kubectl -n rook-ceph get cephcluster,cephblockpool,cephfilesystem,cephobjectstore

# Toolbox shell
kubectl -n rook-ceph exec deploy/rook-ceph-tools -it -- bash

# Ceph health detail
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph health detail

# OSD tree
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree

# Operator logs
kubectl -n rook-ceph logs -l app=rook-ceph-operator --tail=500 -f
```

---

<!-- Sources:
  https://rook.io/docs/rook/latest/Troubleshooting/common-issues/
  https://rook.io/docs/rook/latest/Troubleshooting/ceph-toolbox/
-->
