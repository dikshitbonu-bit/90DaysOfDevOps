# Day 54 – Kubernetes ConfigMaps and Secrets

---

## What ConfigMaps and Secrets Are

**ConfigMap** — stores non-sensitive configuration as plain key-value pairs or full config files. Decouples config from the container image so you don't rebuild on every value change.

**Secret** — stores sensitive data (passwords, tokens, keys) as base64-encoded values. Same mechanism as ConfigMap but with RBAC separation, tmpfs storage on nodes (never written to disk), and optional encryption at rest.

Rule of thumb: ConfigMap for anything you'd commit to git. Secret for anything you wouldn't.

---

## Task 1 – ConfigMap from Literals

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080

kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

Data is stored as plain text — no encoding, no encryption. Anyone with read access to the namespace can see it.

---

## Task 2 – ConfigMap from a File

**File:** `default.conf`

```nginx
server {
    listen 80;
    location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
    }
}
```

```bash
kubectl create configmap nginx-config --from-file=default.conf=default.conf
kubectl get configmap nginx-config -o yaml
```

The key name (`default.conf`) becomes the filename when mounted into a Pod. The full file contents become the value.

---

## Task 3 – Use ConfigMaps in a Pod

**Inject all keys as environment variables — `envfrom-pod.yaml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo APP_ENV=$APP_ENV && echo APP_PORT=$APP_PORT && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
```

**Mount config file as a volume — `nginx-config-pod.yaml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-config-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-conf
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: nginx-conf
    configMap:
      name: nginx-config
```

```bash
kubectl apply -f envfrom-pod.yaml
kubectl apply -f nginx-config-pod.yaml

kubectl logs envfrom-pod
kubectl exec nginx-config-pod -- curl -s http://localhost/health
# Output: healthy
```

**Environment variables vs volume mounts:**

| | `envFrom` / `env` | Volume mount |
|---|---|---|
| Use for | Simple key-value settings | Full config files |
| Updates | Set at pod startup — never update live | Auto-update within ~60s |
| Access in container | `$ENV_VAR` | File at mount path |

---

## Task 4 – Create a Secret

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cureP@ssw0rd

kubectl get secret db-credentials -o yaml
```

Values are base64-encoded in the output:

```bash
# Decode manually
echo '<base64-value>' | base64 --decode

# Or in one command
kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

**Why base64 is encoding, not encryption:**

base64 is a reversible encoding format — anyone can decode it instantly with a single command. It exists to allow binary data to be stored as text in YAML, not to protect it. The real security benefits of Secrets come from:
- **RBAC** — you can grant read access to ConfigMaps but not Secrets
- **tmpfs** — Secret volumes are stored in memory on the node, never written to disk
- **Encryption at rest** — with `EncryptionConfiguration`, etcd stores Secrets encrypted (not enabled by default)

---

## Task 5 – Use Secrets in a Pod

**File:** `secret-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo DB_USER=$DB_USER && cat /etc/db-credentials/DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER
    volumeMounts:
    - name: db-creds
      mountPath: /etc/db-credentials
      readOnly: true
  volumes:
  - name: db-creds
    secret:
      secretName: db-credentials
```

```bash
kubectl apply -f secret-pod.yaml
kubectl logs secret-pod
```

Each Secret key becomes a **file** at the mount path. The file contents are the **decoded plaintext value** — not base64. Kubernetes decodes automatically on mount.

---

## Task 6 – ConfigMap Live Update via Volume

**Create ConfigMap:**

```bash
kubectl create configmap live-config --from-literal=message=hello
```

**File:** `live-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: live-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "while true; do cat /etc/config/message; echo; sleep 5; done"]
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: live-config
```

```bash
kubectl apply -f live-pod.yaml
kubectl logs -f live-pod
# Output: hello (repeating every 5s)

# Update the ConfigMap
kubectl patch configmap live-config --type merge -p '{"data":{"message":"world"}}'

# Wait 30-60 seconds — logs change to:
# Output: world
```

Volume-mounted ConfigMaps update automatically because the kubelet syncs ConfigMap data to the mounted volume on a ~60s cycle. Environment variables do **not** update — they are injected once at container startup and frozen for the life of the pod.

---

## Task 7 – Clean Up

```bash
kubectl delete pod envfrom-pod nginx-config-pod secret-pod live-pod
kubectl delete configmap app-config nginx-config live-config
kubectl delete secret db-credentials

kubectl get pods
kubectl get configmaps
kubectl get secrets
```