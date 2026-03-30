# Day 57 – Resource Requests, Limits, and Probes

---

## Requests vs Limits

**Requests** — the guaranteed minimum. The scheduler uses this to decide which node a pod lands on. If a node doesn't have enough free CPU/memory to satisfy the request, the pod doesn't go there.

**Limits** — the maximum allowed. The kubelet enforces this at runtime. Exceed the CPU limit and you get throttled. Exceed the memory limit and the container is killed immediately.

| | Requests | Limits |
|---|---|---|
| Used by | Scheduler (placement) | kubelet (runtime enforcement) |
| CPU over limit | Throttled | Throttled |
| Memory over limit | — | OOMKilled (exit 137) |

CPU is in millicores: `100m` = 0.1 of one CPU core. Memory is in mebibytes: `128Mi`.

---

## Task 1 – Resource Requests and Limits

**File:** `resources-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resources-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi
```

```bash
kubectl apply -f resources-pod.yaml
kubectl describe pod resources-pod
# Requests:  cpu: 100m, memory: 128Mi
# Limits:    cpu: 250m, memory: 256Mi
# QoS Class: Burstable
```

**QoS Classes:**

| Class | Condition |
|-------|-----------|
| `Guaranteed` | requests == limits for all containers |
| `Burstable` | requests < limits (at least one container) |
| `BestEffort` | no requests or limits set at all |

Under memory pressure, Kubernetes evicts `BestEffort` pods first, then `Burstable`, then `Guaranteed`.

---

## Task 2 – OOMKilled: Exceeding Memory Limits

**File:** `oom-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      limits:
        memory: 100Mi
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
```

```bash
kubectl apply -f oom-pod.yaml
kubectl describe pod oom-pod
# State:       Terminated
# Reason:      OOMKilled
# Exit Code:   137
```

Exit code `137` = `128 + SIGKILL`. Memory is incompressible — there's no "slow it down" for memory. The moment usage exceeds the limit, the kernel kills the process instantly. CPU is compressible — it just gets throttled (slowed down, not killed).

---

## Task 3 – Pending Pod: Requesting Too Much

**File:** `pending-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      requests:
        cpu: "100"
        memory: 128Gi
```

```bash
kubectl apply -f pending-pod.yaml
kubectl get pod pending-pod
# STATUS: Pending  (forever)

kubectl describe pod pending-pod
# Events:
# Warning  FailedScheduling  0/1 nodes are available:
#          1 Insufficient cpu, 1 Insufficient memory.
```

The scheduler runs but finds no node with 100 CPUs and 128Gi free. The pod stays `Pending` indefinitely — it never even tries to start. This is the scheduler telling you exactly what's wrong.

---

## Task 4 – Liveness Probe

A liveness probe detects stuck/broken containers. Failure triggers a **restart**.

**File:** `liveness-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 600"]
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

```bash
kubectl apply -f liveness-pod.yaml
kubectl get pod liveness-pod -w
# 0/1 Running (0 restarts)
# ... 30s later, /tmp/healthy deleted ...
# 0/1 Running (1 restart)  ← 3 consecutive failures triggered restart
```

Timeline: pod starts → file exists → probe passes → 30s → file deleted → probe fails 3× in 15s → container restarted → file recreated → probe passes again.

> **Screenshot:** `kubectl get pod liveness-pod` showing RESTARTS > 0

---

## Task 5 – Readiness Probe

A readiness probe controls traffic routing. Failure **removes the pod from Service endpoints** but does NOT restart the container.

**File:** `readiness-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
  labels:
    app: readiness-test
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

```bash
kubectl apply -f readiness-pod.yaml
kubectl expose pod readiness-pod --port=80 --name=readiness-svc
kubectl get endpoints readiness-svc
# ENDPOINTS: 10.244.0.x:80  ← pod IP listed

# Break the probe
kubectl exec readiness-pod -- rm /usr/share/nginx/html/index.html

# Wait 15s
kubectl get pod readiness-pod
# READY: 0/1  ← not ready, but NOT restarted

kubectl get endpoints readiness-svc
# ENDPOINTS: <none>  ← removed from load balancer
```

The container is still running — nginx process is alive. But traffic stops reaching it because the readiness probe failed. This is how rolling updates work safely: a new pod only receives traffic after its readiness probe passes.

---

## Task 6 – Startup Probe

A startup probe gives slow-starting containers a protected window. While the startup probe is running, liveness and readiness probes are **disabled** — preventing premature restarts during initialization.

**File:** `startup-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "sleep 20 && touch /tmp/started && sleep 600"]
    startupProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 5
      failureThreshold: 12
    livenessProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 10
      failureThreshold: 3
```

```bash
kubectl apply -f startup-pod.yaml
kubectl get pod startup-pod -w
# Running — startup probe runs every 5s for up to 60s (5 × 12)
# After 20s, /tmp/started exists → startup probe passes
# Liveness probe takes over — file still there → keeps passing
```

`failureThreshold: 2` instead of `12` would give only 10 seconds (5s × 2) before the startup probe fails and kills the container — before the app has time to initialize. The pod would be stuck in a restart loop forever.

**Probe comparison:**

| Probe | On failure | Typical use |
|-------|-----------|-------------|
| `livenessProbe` | Restart container | Detect stuck/deadlocked app |
| `readinessProbe` | Remove from endpoints | App not ready to serve traffic |
| `startupProbe` | Kill container | Give slow apps time to initialize |

---

## Task 7 – Clean Up

```bash
kubectl delete pod resources-pod oom-pod pending-pod liveness-pod readiness-pod startup-pod
kubectl delete service readiness-svc
kubectl get pods  # empty
```