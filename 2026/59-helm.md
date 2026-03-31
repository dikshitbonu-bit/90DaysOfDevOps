# Day 59 – Helm: Kubernetes Package Manager

---

## What Helm Is

Helm is the package manager for Kubernetes — like `apt` for Ubuntu or `npm` for Node.js. Instead of managing a dozen individual YAML files for one application, Helm bundles them into a single installable unit called a **chart**.

**Three core concepts:**

| Concept | What it is | Analogy |
|---------|-----------|---------|
| **Chart** | A package of Kubernetes manifest templates | A `.deb` or npm package |
| **Release** | A specific installation of a chart in your cluster | An installed instance of a package |
| **Repository** | A collection of charts | npm registry / apt repo |

You can install the same chart multiple times with different configs — each installation is a separate release with its own name.

---

## Task 1 – Install Helm

```bash
# Linux (curl script)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
helm env
```

---

## Task 2 – Add a Repository and Search

```bash
# Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search
helm search repo nginx
helm search repo bitnami
helm search repo bitnami | wc -l   # count all charts
```

`helm repo update` is the equivalent of `apt update` — syncs the latest chart index from the remote repo.

---

## Task 3 – Install a Chart

```bash
helm install my-nginx bitnami/nginx

# See everything that was created
kubectl get all

# Inspect the release
helm list
helm status my-nginx
helm get manifest my-nginx   # shows actual rendered YAML that was applied
```

One command created a Deployment, a Service, a ServiceAccount, and a ConfigMap. Without Helm you'd write and manage all of those separately.


---

## Task 4 – Customize with Values

```bash
# See all configurable defaults
helm show values bitnami/nginx

# Install with inline overrides
helm install my-nginx-custom bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort

# Check what overrides were applied
helm get values my-nginx-custom
```

**File:** `custom-values.yaml`

```yaml
replicaCount: 3

service:
  type: NodePort

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

```bash
# Install from values file
helm install my-nginx-values bitnami/nginx -f custom-values.yaml

# Verify overrides
helm get values my-nginx-values
kubectl get pods -l app.kubernetes.io/instance=my-nginx-values
# 3 pods running
kubectl get svc my-nginx-values
# TYPE: NodePort
```

`--set` is for quick one-off overrides. `-f values.yaml` is for version-controlled, repeatable config — the production pattern.

---

## Task 5 – Upgrade and Rollback

```bash
# Upgrade to 5 replicas
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Check revision history
helm history my-nginx
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         deployed    Upgrade complete

# Rollback to revision 1
helm rollback my-nginx 1

# Check history again
helm history my-nginx
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         superseded  Upgrade complete
# 3         deployed    Rollback to 1   ← new revision, not overwrite
```

Rollback creates **revision 3** — it never overwrites history. Same concept as `kubectl rollout undo` but at the full application stack level: every resource in the chart gets rolled back together atomically.

---

## Task 6 – Create Your Own Chart

```bash
# Scaffold a chart
helm create my-app
```

**Chart structure:**

```
my-app/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default values for templates
└── templates/
    ├── deployment.yaml # Templated Deployment manifest
    ├── service.yaml    # Templated Service manifest
    ├── _helpers.tpl    # Reusable template snippets
    └── hpa.yaml        # Optional HPA template
```

**How Go templating works — `templates/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}  # e.g. my-release-my-app
spec:
  replicas: {{ .Values.replicaCount }}          # from values.yaml
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Template variables:
- `{{ .Values.key }}` — from `values.yaml` or `--set` / `-f` overrides
- `{{ .Chart.Name }}` — from `Chart.yaml`
- `{{ .Release.Name }}` — the name you gave when running `helm install`

**Edit `values.yaml`:**

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"

service:
  type: ClusterIP
  port: 80
```

```bash
# Validate
helm lint my-app

# Preview rendered YAML without installing
helm template my-release ./my-app

# Install
helm install my-release ./my-app
kubectl get pods   # 3 pods running

# Upgrade replicas
helm upgrade my-release ./my-app --set replicaCount=5
kubectl get pods   # 5 pods running
```


---

## Task 7 – Clean Up

```bash
# Uninstall each release
helm uninstall my-nginx
helm uninstall my-nginx-custom
helm uninstall my-nginx-values
helm uninstall my-release

# Remove chart directory and values file
rm -rf my-app custom-values.yaml

# Verify
helm list   # zero releases
kubectl get all   # only default kubernetes service remains
```

`helm uninstall` removes all Kubernetes resources the chart created in one command. Pass `--keep-history` to retain revision history for auditing without keeping the live resources.

---

## Helm vs Raw YAML: Summary

| | Raw YAML | Helm |
|---|---|---|
| Manage one app | 5-15 separate files | 1 chart, 1 command |
| Customize per environment | Copy and edit files | `values.yaml` per env |
| Upgrade | `kubectl apply` each file | `helm upgrade` |
| Rollback | Manual or `kubectl rollout undo` | `helm rollback` (whole stack) |
| Share with team | Git raw YAML | Helm repo |
| Production standard | Simple apps | Complex multi-resource apps |