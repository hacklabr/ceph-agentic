# Kubernetes CSI Integration

This guide covers exposing RBD and CephFS storage to Kubernetes workloads via the Rook-Ceph CSI drivers. It assumes a healthy Rook-Ceph cluster is already deployed.

**When to use this guide:** you want Pods to consume Ceph storage through PersistentVolumes and StorageClasses.

---

## CSI Driver Architecture

The Rook operator deploys two CSI drivers:

- **RBD CSI** — block volumes, `ReadWriteOnce` (RWO) by default, `ReadWriteMany` (RWX) with block volume mode.
- **CephFS CSI** — shared filesystem, `ReadWriteMany` (RWX) by default.

Each driver has:

- A controller plugin (provisioning, snapshot, resize)
- A node plugin (mount/attach on worker nodes)

---

## RBD StorageClass

Create a `CephBlockPool` and a StorageClass:

```yaml
---
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

Apply and test:

```bash
kubectl apply -f rbd-pool.yaml
kubectl get storageclass
```

### PVC example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  storageClassName: rook-ceph-block
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

---

## CephFS StorageClass

Create a `CephFilesystem` and StorageClass:

```yaml
---
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data0
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-data0
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

### PVC example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  storageClassName: rook-cephfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

---

## Verify PVC and Pod Usage

```bash
# PVC should be Bound
kubectl get pvc

# Pod should be Running and volume mounted
kubectl get pod
kubectl exec <pod> -- df -h
```

---

## Snapshots and Volume Expansion

### VolumeSnapshotClass for RBD

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
driver: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
```

### Expand a PVC

```bash
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

Ensure `allowVolumeExpansion: true` is set in the StorageClass.

---

## Troubleshooting

- **PVC stuck Pending:** check `kubectl describe pvc` and the CSI provisioner logs.
- **Pod stuck ContainerCreating:** check node plugin logs and node events.
- **Mount failures:** ensure `ceph-common` or required CSI node utilities are present on worker nodes (Rook CSI containers bundle these).

---

<!-- Sources:
  https://rook.io/docs/rook/latest/Storage-Configuration/Block-Storage-RBD/rbd/
  https://rook.io/docs/rook/latest/Storage-Configuration/Shared-Filesystem-CephFS/cephfs/
-->
