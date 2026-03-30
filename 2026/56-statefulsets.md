# Day 56 – Kubernetes StatefulSets

---

## What StatefulSets Are and When to Use Them

A StatefulSet is the Kubernetes workload for stateful applications — databases, message queues, distributed systems — where each replica needs a stable identity, predictable startup order, and its own dedicated storage. Deployments treat all pods as interchangeable. StatefulSets treat each pod as unique.

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| Pod names | Random (`app-xyz-abc`) | Stable, ordered (`app-0`, `app-1`, `app-2`) |
| Startup order | All at once, parallel | Ordered: `pod-0` → `pod-1` → `pod-2` |
| Storage | Shared PVC (or none) | Each pod gets its own PVC via `volumeClaimTemplates` |
| Network identity | No stable hostname | Stable DNS per pod via Headless Service |
| Use case | Web servers, APIs, stateless apps | MySQL, PostgreSQL, Kafka, Zookeeper, Elasticsearch |

---

## Task 1 – Why Random Names Break Databases

```bash
# Create a 3-replica Deployment
kubectl create deployment web-demo --image=nginx --replicas=3
kubectl get pods
# web-demo-7d6f-abc   Running
# web-demo-7d6f-xyz   Running
# web-demo-7d6f-def   Running

# Delete one pod
kubectl delete pod web-demo-7d6f-abc
kubectl get pods
# Replacement: web-demo-7d6f-NEW  ← completely different name
```

In a database cluster (e.g. MySQL with primary + replicas), other nodes connect to specific peers by hostname. If `mysql-primary` gets a random new name on restart, every replica loses its connection reference. The cluster breaks. Stable identity is not a nice-to-have — it's a hard requirement.

```bash
kubectl delete deployment web-demo
```

---

## Task 2 – Headless Service

**File:** `headless-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f headless-service.yaml
kubectl get service web-headless
# NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# web-headless   ClusterIP   None         <none>        80/TCP
```

`clusterIP: None` tells Kubernetes not to assign a virtual IP. Instead of load-balancing to one IP, DNS returns the individual Pod IPs directly. This gives each StatefulSet pod its own DNS record — required for stable per-pod addressing.

---

## Task 3 – Create a StatefulSet

**File:** `statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: web-data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 100Mi
```

```bash
kubectl apply -f statefulset.yaml
kubectl get pods -l app=web -w
# web-0   Pending → ContainerCreating → Running
# web-1   Pending → ContainerCreating → Running  (only after web-0 is Ready)
# web-2   Pending → ContainerCreating → Running  (only after web-1 is Ready)

kubectl get pvc
# web-data-web-0   Bound
# web-data-web-1   Bound
# web-data-web-2   Bound
```

PVC naming pattern: `<template-name>-<statefulset-name>-<ordinal>`
So `volumeClaimTemplates.name: web-data` + StatefulSet name `web` + ordinal `0` = `web-data-web-0`.

Each pod owns its own PVC. Kubernetes never mixes them up.

> **Screenshot:** `kubectl get pods` and `kubectl get pvc`

---

## Task 4 – Stable Network Identity (DNS)

Each pod gets a DNS record: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

```bash
kubectl run dns-test --image=busybox:latest --rm -it --restart=Never -- sh

# Inside:
nslookup web-0.web-headless.default.svc.cluster.local
nslookup web-1.web-headless.default.svc.cluster.local
nslookup web-2.web-headless.default.svc.cluster.local
exit
```

```bash
# Verify IPs match
kubectl get pods -o wide
# web-0   10.244.0.5
# web-1   10.244.0.6
# web-2   10.244.0.7
```

The `nslookup` IP for `web-0.web-headless...` returns exactly `10.244.0.5` — matching the pod IP. This is how database nodes find each other — stable DNS names that survive pod deletion and recreation.

---

## Task 5 – Stable Storage: Data Survives Pod Deletion

```bash
# Write unique data to each pod
kubectl exec web-0 -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- sh -c "echo 'Data from web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- sh -c "echo 'Data from web-2' > /usr/share/nginx/html/index.html"

# Verify before deletion
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# Data from web-0

# Delete web-0
kubectl delete pod web-0

# Wait for it to come back (gets same name: web-0)
kubectl get pods -w

# Verify data survived
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# Data from web-0  ← still there
```

The new `web-0` pod reconnects to the same `web-data-web-0` PVC. The data is on the PVC, not inside the container — so it survives completely.

---

## Task 6 – Ordered Scaling

```bash
# Scale up — pods create in order
kubectl scale statefulset web --replicas=5
# web-3 created → web-4 created (after web-3 is Ready)

kubectl get pvc
# 5 PVCs: web-data-web-0 through web-data-web-4

# Scale down — pods terminate in reverse order
kubectl scale statefulset web --replicas=3
# web-4 terminates → web-3 terminates (after web-4 is gone)

kubectl get pvc
# Still 5 PVCs — web-data-web-3 and web-data-web-4 are retained
```

Kubernetes intentionally keeps PVCs on scale-down. If you scale back up to 5, `web-3` and `web-4` reconnect to their original PVCs with the original data intact.

---

## Task 7 – Clean Up

```bash
kubectl delete statefulset web
kubectl delete service web-headless

# PVCs are NOT auto-deleted — check them
kubectl get pvc
# web-data-web-0 through web-data-web-4 — all still exist

# Delete manually
kubectl delete pvc web-data-web-0 web-data-web-1 web-data-web-2 web-data-web-3 web-data-web-4

kubectl get pvc  # empty
```

PVCs surviving StatefulSet deletion is a deliberate safety feature — you can't accidentally lose your database storage by deleting the StatefulSet. You must explicitly delete PVCs to release the storage.

---

## How It All Fits Together

```
StatefulSet
  ├── Headless Service  → gives each pod a stable DNS name
  ├── Ordered pods      → web-0, web-1, web-2 (never random)
  └── volumeClaimTemplates → web-data-web-0, web-data-web-1, web-data-web-2
                             (each pod owns its own PVC permanently)
```