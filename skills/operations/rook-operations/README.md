# Operate Rook-Ceph on Kubernetes

This guide covers day-2 operations for a Ceph cluster managed by Rook on Kubernetes: scaling, upgrades, resource management, and common configuration changes.

**When to use this guide:** the cluster is already deployed and you need to change its topology, upgrade it, or tune its resources.

---

## Scaling OSDs

Rook creates OSDs based on the `storage` section of the `CephCluster` CR. There are two common patterns.

### Add devices to existing nodes

Edit the CephCluster CR and add devices:

```yaml
spec:
  storage:
    nodes:
      - name: k8s-worker-01
        devices:
          - name: sdb
          - name: sdc   # new device
```

Apply the change:

```bash
kubectl apply -f ceph-cluster.yaml
```

Rook will create a new OSD pod for the new device. Watch progress:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-osd -w
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
```

### Add a new node

Add the node to the CR:

```yaml
spec:
  storage:
    nodes:
      - name: k8s-worker-01
        devices:
          - name: sdb
      - name: k8s-worker-04   # new node
        devices:
          - name: sdb
```

Ensure the new node has:

- `lvm2` installed
- An unformatted device
- Sufficient CPU/RAM

### Use device sets (recommended for homogeneous production clusters)

Device sets are the declarative equivalent of OSD specs. Example for 3 replicas of 1 TB OSDs:

```yaml
spec:
  storage:
    storageClassDeviceSets:
      - name: set1
        count: 3
        portable: false
        tuneDeviceClass: true
        tuneFastPathClass: false
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              resources:
                requests:
                  storage: 1Ti
              storageClassName: local-block
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

To scale, increase `count` and apply.

---

## Remove OSDs

OSD removal in Rook is more involved than with cephadm because you must both purge Ceph state and remove the Kubernetes resources.

### Safe removal procedure

1. **Identify the OSD to remove:**

   ```bash
   kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree
   ```

2. **Set noout to prevent premature rebalancing:**

   ```bash
   kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd set noout
   ```

3. **Mark the OSD out:**

   ```bash
   kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd out osd.<id>
   ```

4. **Wait for data migration to complete:**

   ```bash
   kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
   ```

   All PGs must be `active+clean` before proceeding.

5. **Remove the OSD from the CephCluster CR:**

   Remove the device or reduce the device set count. Apply the change.

6. **Purge the OSD:**

   Rook provides a cleanup job. Create a job to zap the device:

   ```bash
   kubectl apply -f - <<EOF
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: rook-ceph-osd-remove-<id>
     namespace: rook-ceph
   spec:
     template:
       spec:
         containers:
           - name: osd-removal
             image: rook/ceph:master
             command:
               - "ceph"
               - "osd"
               - "purge"
               - "<id>"
               - "--yes-i-really-really-mean-it"
             volumeMounts:
               - mountPath: /var/lib/rook
                 name: rook-data
         volumes:
           - name: rook-data
             hostPath:
               path: /var/lib/rook
               type: Directory
         restartPolicy: Never
   EOF
   ```

   > **WARNING:** This is a destructive operation. Verify the OSD is out and data has migrated before purging.

7. **Unset noout:**

   ```bash
   kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd unset noout
   ```

---

## Upgrade Rook

### Upgrade the Rook Operator

```bash
helm upgrade --namespace rook-ceph rook-ceph rook-release/rook-ceph
```

Wait for the operator to reconcile and for all pods to return to Ready.

### Upgrade Ceph version

Change the image in the CephCluster CR:

```yaml
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.2.5
```

Apply and monitor:

```bash
kubectl apply -f ceph-cluster.yaml
kubectl -n rook-ceph get cephcluster rook-ceph -w
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph versions
```

Ceph upgrades inside Rook follow the same order as cephadm: MGR → MON → OSD → clients.

---

## Resource Limits and Tolerations

### OSD resource limits

```yaml
spec:
  resources:
    osd:
      limits:
        memory: "8Gi"
        cpu: "2000m"
      requests:
        memory: "4Gi"
        cpu: "1000m"
```

### Tolerations for dedicated storage nodes

```yaml
spec:
  placement:
    all:
      tolerations:
        - key: dedicated
          operator: Equal
          value: storage
          effect: NoSchedule
```

---

## Common Configuration Changes

### Enable the Ceph dashboard

```yaml
spec:
  dashboard:
    enabled: true
    ssl: true
```

Access via port-forward:

```bash
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 8443:8443
```

### Enable monitoring (Prometheus/Grafana)

```yaml
spec:
  monitoring:
    enabled: true
```

Requires the Prometheus Operator and ServiceMonitor CRDs.

---

<!-- Sources:
  https://rook.io/docs/rook/latest/Storage-Configuration/Advanced/ceph-configuration-on-pvc/
  https://rook.io/docs/rook/latest/Upgrade/rook-upgrade/
-->
