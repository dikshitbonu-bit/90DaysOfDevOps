# Day 58 – Metrics Server and Horizontal Pod Autoscaler (HPA)

---

## What the Metrics Server Is and Why HPA Needs It

The Metrics Server is a lightweight in-cluster component that collects real-time CPU and memory usage from every kubelet every 15 seconds. Without it, `kubectl top` returns nothing and HPA has no data to act on — it just shows `<unknown>` in the TARGETS column and never scales.

HPA doesn't watch the containers directly. It reads from the Metrics Server API. The chain is:

```
kubelet (node agent) → Metrics Server → HPA controller → scales Deployment
```

---

## Task 1 – Install the Metrics Server

```bash
# Check if already running
kubectl get pods -n kube-system | grep metrics-server

# Kind
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Kind needs --kubelet-insecure-tls (local cluster only, never production)
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait 60s then verify
kubectl top nodes
kubectl top pods -A
```

---

## Task 2 – Explore kubectl top

```bash
kubectl top nodes
kubectl top pods -A
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

**`kubectl top` vs `kubectl describe pod`:**

| | `kubectl top` | `kubectl describe pod` |
|---|---|---|
| Shows | Actual live usage | Configured requests/limits |
| Source | Metrics Server (15s poll) | Pod spec |
| Example | `cpu: 32m` (what it's using) | `requests.cpu: 100m` (what it's promised) |

A pod can be using `32m` CPU while its request is `100m` — the scheduler reserved `100m` on the node regardless.

---

## Task 3 – Deployment with CPU Requests

**File:** `php-apache-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
```

```bash
kubectl apply -f php-apache-deployment.yaml
kubectl expose deployment php-apache --port=80
kubectl top pod -l app=php-apache
```

`resources.requests.cpu` is mandatory for HPA. Without it, HPA cannot calculate a utilization percentage — it doesn't know what 100% means. The most common HPA setup mistake is forgetting this field.

---

## Task 4 – Create HPA (Imperative)

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa
kubectl describe hpa php-apache
```

HPA formula:
```
desiredReplicas = ceil(currentReplicas × (currentUsage / targetUsage))
```

Example: 1 replica using 100% of its 200m request → `ceil(1 × (100% / 50%))` = 2 replicas.

```bash
kubectl get hpa
# NAME         REFERENCE               TARGETS    MINPODS  MAXPODS  REPLICAS
# php-apache   Deployment/php-apache   0%/50%     1        10       1
```

`TARGETS` shows `<unknown>` for the first ~30s until Metrics Server data arrives.

---

## Task 5 – Generate Load and Watch Autoscaling

```bash
# Start load generator
kubectl run load-generator --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

# Watch HPA in real time
kubectl get hpa php-apache --watch
# TARGETS: 0%/50% → 112%/50% → 250%/50% → replicas: 3 → 5 → 7
```

HPA checks every 15 seconds. Scale-up is fast — new pods appear within 30-60s. Scale-down has a 5-minute stabilization window to prevent flapping.

```bash
# Stop load
kubectl delete pod load-generator

# Pods scale back down (wait 5 min to observe, not required)
kubectl get hpa php-apache --watch
```

---

## Task 6 – HPA from YAML (autoscaling/v2)

**File:** `hpa-v2.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

```bash
kubectl delete hpa php-apache
kubectl apply -f hpa-v2.yaml
kubectl describe hpa php-apache
```

**What `behavior` controls:**

| Section | Setting | Effect |
|---------|---------|--------|
| `scaleUp.stabilizationWindowSeconds: 0` | No cooldown on scale-up | Scales up immediately when threshold crossed |
| `scaleUp.policies: 100% per 15s` | Max scale-up rate | Can double replicas every 15s |
| `scaleDown.stabilizationWindowSeconds: 300` | 5 min cooldown on scale-down | Prevents flapping when load fluctuates |
| `scaleDown.policies: 1 pod per 60s` | Slow scale-down rate | Removes one pod per minute |

**autoscaling/v1 vs v2:**

| | `autoscaling/v1` | `autoscaling/v2` |
|---|---|---|
| Metrics | CPU only | CPU + memory + custom metrics |
| Behavior control | None | Full scale-up/down policies |
| Multiple metrics | No | Yes (scale on whichever triggers first) |
| Production use | Legacy | Standard |

---

## Task 7 – Clean Up

```bash
kubectl delete hpa php-apache
kubectl delete service php-apache
kubectl delete deployment php-apache
kubectl delete pod load-generator --ignore-not-found

kubectl get hpa       # empty
kubectl get pods      # empty
# Metrics Server stays installed
```