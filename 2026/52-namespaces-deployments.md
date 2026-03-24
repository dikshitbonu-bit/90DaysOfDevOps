# Day 52 – Kubernetes Namespaces and Deployments

---

## What Namespaces Are and Why You Use Them

A namespace is a virtual cluster inside your physical cluster. It lets you isolate resources so that `dev`, `staging`, and `production` workloads live in the same cluster but cannot see or interfere with each other. Teams, environments, and applications get their own namespace — resources in different namespaces can have the same name without conflict.

---

## Task 1 – Default Namespaces

```bash
kubectl get namespaces
kubectl get pods -n kube-system
```

| Namespace | Purpose |
|-----------|---------|
| `default` | Where resources go if you don't specify a namespace |
| `kube-system` | Kubernetes internal components — API server, scheduler, etcd, coredns |
| `kube-public` | Publicly readable resources, rarely used |
| `kube-node-lease` | Node heartbeat tracking — keeps the API server from flooding with node health requests |

---

## Task 2 – Create and Use Custom Namespaces

```bash
# Imperative
kubectl create namespace dev
kubectl create namespace staging
```

**Declarative — namespace.yaml:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```bash
kubectl apply -f namespace.yaml
kubectl get namespaces

# Run pods in specific namespaces
kubectl run nginx-dev --image=nginx:latest -n dev
kubectl run nginx-staging --image=nginx:latest -n staging

# See all pods across all namespaces
kubectl get pods -A
```

`kubectl get pods` with no flags only shows the `default` namespace. You must pass `-n <namespace>` or `-A` to see everything.

---

## Task 3 – Deployment Manifest

**File:** `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
```

**What each section does:**

| Section | Purpose |
|---------|---------|
| `apiVersion: apps/v1` | Deployments live in the `apps` API group, not core `v1` |
| `replicas: 3` | Kubernetes maintains exactly 3 running pods at all times |
| `selector.matchLabels` | How the Deployment finds the pods it owns — must match `template.metadata.labels` exactly |
| `template` | The pod blueprint — the Deployment clones this to create each replica |

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments -n dev
kubectl get pods -n dev
```

**Deployment columns explained:**

| Column | Meaning |
|--------|---------|
| `READY` | How many replicas are ready vs desired (e.g. `3/3`) |
| `UP-TO-DATE` | How many replicas have the latest spec applied |
| `AVAILABLE` | How many replicas are available to serve traffic |



---

## Task 4 – Self-Healing

```bash
kubectl get pods -n dev

# Delete one pod
kubectl delete pod <pod-name> -n dev

# Watch immediately
kubectl get pods -n dev
```

The Deployment controller detects only 2 of 3 desired replicas exist and creates a replacement within seconds. The new pod gets a **different name** — it is a brand new pod, not the same one resurrected. This is the fundamental difference from a standalone Pod, which stays deleted forever.

---

## Task 5 – Scaling

**Imperative:**
```bash
kubectl scale deployment nginx-deployment --replicas=5 -n dev
kubectl get pods -n dev

kubectl scale deployment nginx-deployment --replicas=2 -n dev
kubectl get pods -n dev
```

**Declarative** — edit `replicas: 4` in the YAML, then:
```bash
kubectl apply -f nginx-deployment.yaml
```

When scaling down from 5 to 2, Kubernetes **terminates 3 pods** — it picks pods to remove, sends them a `SIGTERM`, waits for graceful shutdown, then removes them. The cluster converges to exactly 2 replicas.

---

## Task 6 – Rolling Update and Rollback

**Trigger a rolling update:**
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev
kubectl rollout status deployment/nginx-deployment -n dev
```

Kubernetes replaces pods **one by one** — a new pod must be healthy before the next old one is terminated. Zero downtime. Behind the scenes it creates a new ReplicaSet for the new version and gradually scales it up while scaling the old one down.

**Check history:**
```bash
kubectl rollout history deployment/nginx-deployment -n dev
```

**Roll back to previous version:**
```bash
kubectl rollout undo deployment/nginx-deployment -n dev
kubectl rollout status deployment/nginx-deployment -n dev

# Verify image rolled back
kubectl describe deployment nginx-deployment -n dev | grep Image
```

After rollback the image returns to `nginx:1.24`. Kubernetes keeps the old ReplicaSet around specifically to enable this — it just scales it back up.

---

## Task 7 – Clean Up

```bash
kubectl delete deployment nginx-deployment -n dev
kubectl delete pod nginx-dev -n dev
kubectl delete pod nginx-staging -n staging
kubectl delete namespace dev staging production

# Verify
kubectl get namespaces
kubectl get pods -A
```

Deleting a namespace deletes **everything inside it** — pods, deployments, services, configmaps. There is no recovery. In production, namespace deletion requires explicit access controls for exactly this reason.

---

## Standalone Pod vs Deployment Pod — Summary

| | Standalone Pod | Deployment Pod |
|---|---|---|
| Deleted? | Gone forever | Replaced automatically |
| Crashed? | Stays dead | Restarted by controller |
| Scaling | Manual, one by one | Single `--replicas` flag |
| Updates | Delete and recreate | Rolling update, zero downtime |
| Production use | Never | Always |