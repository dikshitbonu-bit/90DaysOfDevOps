# Day 55 – Persistent Volumes and Persistent Volume Claims

---

## Why Containers Need Persistent Storage

Containers are ephemeral by design. When a Pod dies or gets rescheduled, everything written inside the container filesystem is gone. For stateless apps (React frontend, nginx) that's fine. For databases, message queues, or anything that writes data — it's a catastrophe. PVs and PVCs decouple storage from the Pod lifecycle so data survives Pod deletions, restarts, and rescheduling.

---

## Task 1 – See the Problem: Data Lost on Pod Deletion

**File:** `emptydir-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo Timestamp: $(date) > /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: temp-storage
      mountPath: /data
  volumes:
  - name: temp-storage
    emptyDir: {}
```

```bash
kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-pod -- cat /data/message.txt
# Output: Timestamp: Mon Mar 30 10:00:00 UTC 2026

kubectl delete pod emptydir-pod
kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-pod -- cat /data/message.txt
# Output: Timestamp: Mon Mar 30 10:05:00 UTC 2026  ← different timestamp, old data gone
```

`emptyDir` lives as long as the Pod. When the Pod dies, the volume is deleted with it. The timestamp changes on every recreation — proving the data is gone.

---

## Task 2 – Create a PersistentVolume (Static Provisioning)

**File:** `pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/k8s-pv-data
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
# STATUS: Available
```

**Access modes:**

| Mode | Short | Meaning |
|------|-------|---------|
| `ReadWriteOnce` | RWO | Read-write by a single node |
| `ReadOnlyMany` | ROX | Read-only by many nodes |
| `ReadWriteMany` | RWX | Read-write by many nodes |

`hostPath` stores data on the node's local filesystem at `/tmp/k8s-pv-data`. Fine for local learning — in production you'd use cloud storage (AWS EBS, GCP PD, Azure Disk).

PVs are **cluster-wide** — not namespaced. Any namespace can claim them.

---

## Task 3 – Create a PersistentVolumeClaim

**File:** `pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
# STATUS: Bound   VOLUME: my-pv

kubectl get pv
# STATUS: Bound
```

Kubernetes matched the PVC to `my-pv` because the access mode matched and the PV capacity (1Gi) satisfies the PVC request (500Mi). The `VOLUME` column in `kubectl get pvc` shows `my-pv` — the exact PV it bound to.

PVC status flow: `Pending` → `Bound`
PV status flow: `Available` → `Bound` → `Released`

---

## Task 4 – Use the PVC in a Pod

**File:** `pvc-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo Pod-1 written at $(date) >> /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

```bash
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-pod -- cat /data/message.txt
# Pod-1 written at Mon Mar 30 10:00:00 UTC 2026

kubectl delete pod pvc-pod

# Edit command to write Pod-2, apply again
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-pod -- cat /data/message.txt
# Pod-1 written at Mon Mar 30 10:00:00 UTC 2026
# Pod-2 written at Mon Mar 30 10:05:00 UTC 2026
```

Both entries exist — data from the first Pod survived its deletion. The PVC held the reference to the PV, and the PV held the data on the host filesystem.

---

## Task 5 – StorageClasses and Dynamic Provisioning

```bash
kubectl get storageclass
kubectl describe storageclass
```

**What a StorageClass does:**

With static provisioning, an admin manually creates PVs and developers claim them. With dynamic provisioning, developers just create a PVC with a `storageClassName` — the StorageClass automatically provisions a PV from the cloud provider. No manual PV creation needed.

The default StorageClass in a kind cluster is typically `standard` with the `rancher.io/local-path` provisioner.

---

## Task 6 – Dynamic Provisioning

**File:** `dynamic-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: standard
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 200Mi
```

```bash
kubectl apply -f dynamic-pvc.yaml
kubectl get pv
# Now shows 2 PVs:
# my-pv        — manually created (static)
# pvc-xxxxxxx  — auto-created by StorageClass (dynamic)
```

**File:** `dynamic-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo Dynamic PV works > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: dyn-storage
      mountPath: /data
  volumes:
  - name: dyn-storage
    persistentVolumeClaim:
      claimName: dynamic-pvc
```

```bash
kubectl apply -f dynamic-pod.yaml
kubectl exec dynamic-pod -- cat /data/test.txt
# Dynamic PV works
```

---

## Task 7 – Clean Up

```bash
# Delete pods first
kubectl delete pod pvc-pod dynamic-pod

# Delete PVCs
kubectl delete pvc my-pvc dynamic-pvc

# Check what happened to the PVs
kubectl get pv
# my-pv       → STATUS: Released  (Retain policy — data kept, PV stays)
# pvc-xxxxxxx → gone              (Delete policy — PV auto-deleted with PVC)

# Manually delete the retained PV
kubectl delete pv my-pv

kubectl get pv  # empty
```

**Why the difference:**

| Reclaim Policy | What happens when PVC is deleted |
|----------------|----------------------------------|
| `Retain` | PV stays, status = `Released`. Data preserved. Admin must manually delete. |
| `Delete` | PV and underlying storage deleted automatically. |

Static PVs default to `Retain` — you set it explicitly. Dynamic PVs from a StorageClass default to `Delete` — the StorageClass defines the policy.

---

## Summary: PV vs PVC

| | PersistentVolume (PV) | PersistentVolumeClaim (PVC) |
|---|---|---|
| Created by | Admin (static) or StorageClass (dynamic) | Developer |
| Scope | Cluster-wide (not namespaced) | Namespaced |
| Represents | Actual storage resource | A request for storage |
| Analogy | A hotel room | A hotel booking |