# Deploy Rook on Kubernetes

This guide covers deploying a Rook-Ceph cluster on Kubernetes. Rook manages Ceph as Kubernetes custom resources (CRs), automating daemon lifecycle, service discovery, and storage provisioning.

**When to use this guide:** you are installing a new Rook-Ceph cluster or redeploying after a teardown.

---

## Architecture Overview

Rook components:

- **Rook Operator** — watches `CephCluster`, `CephBlockPool`, `CephFilesystem`, `CephObjectStore`, and other CRs; reconciles desired state.
- **CephCluster CR** — defines cluster topology, versions, storage devices, network, and daemons.
- **MON pods** — run a Ceph monitor quorum (usually 3).
- **MGR pods** — active/standby Ceph managers (usually 2).
- **OSD pods** — one per device or device set.
- **CSI driver pods** — RBD and CephFS provisioners, node plugins, and snapshot controllers.

---

## Install the Rook Operator

### Using Helm (recommended)

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update

helm install --create-namespace --namespace rook-ceph rook-ceph \
  rook-release/rook-ceph
```

Wait for the operator pod:

```bash
kubectl -n rook-ceph wait --for=condition=ready pod -l app=rook-ceph-operator --timeout=120s
```

### Using plain YAML

```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.16/deploy/examples/common.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.16/deploy/examples/operator.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.16/deploy/examples/csi/nfs/rbac.yaml
```

---

## Deploy the CephCluster CR

Create a minimal cluster spec. This example assumes every storage node has an unformatted device named `sdb`:

```yaml
# ceph-cluster.yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.2.5
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 2
  dashboard:
    enabled: true
  monitoring:
    enabled: false
  storage:
    useAllNodes: false
    nodes:
      - name: k8s-worker-01
        devices:
          - name: sdb
      - name: k8s-worker-02
        devices:
          - name: sdb
      - name: k8s-worker-03
        devices:
          - name: sdb
```

Apply and wait:

```bash
kubectl apply -f ceph-cluster.yaml
kubectl -n rook-ceph get cephcluster rook-ceph -w
```

Wait for `phase: Ready` and `health: HEALTH_OK`.

### Using all available devices

For homogeneous lab clusters where every storage node has only unformatted data disks:

```yaml
spec:
  storage:
    useAllNodes: true
    useAllDevices: true
```

> **WARNING:** `useAllDevices: true` consumes every unformatted disk on nodes labeled or selected for storage. Verify device inventory before enabling this.

---

## Configure Ceph Networking

### Host networking (default)

Ceph daemons use the host network stack. No extra CNI configuration is needed, but ensure host ports are not conflicting.

### Multus (isolated public and cluster networks)

If you have Multus installed and want isolated networks, add:

```yaml
spec:
  network:
    provider: multus
    selectors:
      public: public-conf
      cluster: cluster-conf
```

Create the NetworkAttachmentDefinitions `public-conf` and `cluster-conf` first.

---

## Verify Deployment

```bash
# Operator and cluster status
kubectl -n rook-ceph get cephcluster

# All pods should be Running
kubectl -n rook-ceph get pod

# Ceph health via toolbox
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
```

Expected output:

```
  cluster:
    id:     ...
    health: HEALTH_OK
  services:
    mon: 3 daemons, quorum a,b,c
    mgr: a(active, since ...), standbys: b
    osd: 3 osds: 3 up, 3 in
```

---

## Deploy the Toolbox

The toolbox provides a `ceph` CLI with the correct keyring and config:

```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.16/deploy/examples/toolbox.yaml
kubectl -n rook-ceph exec deploy/rook-ceph-tools -it -- ceph status
```

---

## Deploy CSI Drivers

The Rook Operator usually deploys the CSI drivers automatically. To configure them, see `skills/operations/k8s-csi`.

---

## Export Cluster State

Store the CephCluster CR and related resources in Git:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph -o yaml > rook-ceph-cluster.yaml
kubectl -n rook-ceph get configmap rook-ceph-config -o yaml > rook-ceph-config.yaml
```

Redact secrets before committing.

---

## Common Pitfalls

- **OSD pods do not start:** verify the device is unformatted and `lvm2` is installed.
- **MON pods stuck Pending:** verify nodes have enough CPU/RAM and no taints blocking Rook unless tolerations are set.
- **Ceph health stays HEALTH_WARN:** check the toolbox `ceph health detail` output.
- **Rook version mismatch:** ensure the Rook Operator version supports the Ceph image version in the CephCluster CR.

---

<!-- Sources:
  https://rook.io/docs/rook/latest/Getting-Started/quickstart/
  https://rook.io/docs/rook/latest/Storage-Configuration/Ceph-ceph-cluster-crd/
-->
