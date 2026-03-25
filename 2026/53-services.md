# Day 53 – Kubernetes Services

---


## Task 1 – Deploy the Application

**File:** `app-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f app-deployment.yaml
kubectl get pods -o wide
```

Note the Pod IPs — these are ephemeral. Every restart gives a new IP. That's exactly what Services fix.

---

## Task 2 – ClusterIP Service (Internal Only)

**File:** `clusterip-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

| Field | Purpose |
|-------|---------|
| `selector.app: web-app` | Routes traffic to all pods with this label |
| `port: 80` | Port the Service listens on |
| `targetPort: 80` | Port on the Pod to forward traffic to |

```bash
kubectl apply -f clusterip-service.yaml
kubectl get services

# Test from inside the cluster
kubectl run test-client --image=busybox:latest --rm -it --restart=Never -- sh
# Inside:
wget -qO- http://web-app-clusterip
exit
```

ClusterIP is reachable **only from inside the cluster**. External traffic cannot reach it directly.

> **Screenshot:** `kubectl get services` showing ClusterIP

---

## Task 3 – Kubernetes DNS

Every Service gets an automatic DNS entry:

```
<service-name>.<namespace>.svc.cluster.local
```

```bash
kubectl run dns-test --image=busybox:latest --rm -it --restart=Never -- sh
# Inside:
wget -qO- http://web-app-clusterip                             # short name (same namespace)
wget -qO- http://web-app-clusterip.default.svc.cluster.local  # full DNS name
nslookup web-app-clusterip
exit
```

`nslookup` returns the same CLUSTER-IP shown in `kubectl get services`. CoreDNS (running in `kube-system`) handles all DNS resolution inside the cluster. Short name works within the same namespace; full DNS name is needed cross-namespace.

---

## Task 4 – NodePort Service (External via Node)

**File:** `nodeport-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

Traffic flow: `<NodeIP>:30080` → Service → `Pod:80`

`nodePort` must be in range `30000–32767`. If omitted, Kubernetes picks one automatically.

```bash
kubectl apply -f nodeport-service.yaml
kubectl get services


# kind — get node IP first
kubectl get nodes -o wide
curl http://<node-internal-ip>:30080

```
---

## Task 5 – LoadBalancer Service (Cloud External Access)

**File:** `loadbalancer-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f loadbalancer-service.yaml
kubectl get services

# minikube only — simulates a cloud LB
minikube tunnel
```

On a local cluster, `EXTERNAL-IP` shows `<pending>` — there is no cloud provider to provision a real load balancer. In AWS/GCP/Azure, this automatically creates an ELB/NLB/ALB and assigns a public IP or hostname.

---

## Task 6 – Service Types Side by Side

```bash
kubectl get services -o wide
kubectl describe service web-app-loadbalancer
```

A LoadBalancer also has a ClusterIP and a NodePort assigned — each type **builds on the previous one**.

| Type | Accessible From | Use Case |
|------|----------------|----------|
| `ClusterIP` | Inside cluster only | Internal service-to-service communication |
| `NodePort` | Outside via `<NodeIP>:<NodePort>` | Dev/testing, direct node access |
| `LoadBalancer` | Outside via cloud LB public IP | Production traffic in cloud environments |

**What are Endpoints?**

Every Service maintains an Endpoints object — the actual list of Pod IPs currently backing it. When a Pod is added, removed, or crashes, the Endpoints list updates automatically.

```bash
kubectl get endpoints web-app-clusterip
```

If this list is empty, the Service selector doesn't match any running Pods — the most common cause of a Service routing to nothing.

---

## Task 7 – Clean Up

```bash
kubectl delete -f app-deployment.yaml
kubectl delete -f clusterip-service.yaml
kubectl delete -f nodeport-service.yaml
kubectl delete -f loadbalancer-service.yaml

kubectl get pods
kubectl get services
```

Only the built-in `kubernetes` service in `default` namespace should remain.