# Loki + Grafana + Fluent Bit – Dev Installation Guide

This guide explains how to set up **Loki (logs backend)**, **Grafana (visualization)**, and **Fluent Bit (log forwarder)** on Kubernetes using **Helm**.

The setup is optimized for **learning, labs, and dev environments** (KodeKloud / local clusters / self‑managed K8s).

---

## Architecture Overview

```
Application Pods
      │
      ▼
Fluent Bit (DaemonSet)
      │
      ▼
Loki (SingleBinary)
      │
      ▼
Grafana (Explore / Dashboards / Alerts)
```

---

## Prerequisites

* Kubernetes cluster (kind / k3s / kubeadm / cloud)
* Helm v3 installed
* kubectl configured

---

## Step 1: Add Helm Repositories

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

---

## Step 2: Storage Class (IMPORTANT)

### Option A: Cluster WITHOUT default storage (kind / kubeadm)

Use **local-path-provisioner**:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Option B: Cluster WITH storage already (OpenEBS / EBS / GCE PD)

⚠️ **Skip this step** if your cluster already has:

* OpenEBS
* AWS EBS
* Azure Disk
* GCE Persistent Disk

👉 Just replace `storageClass: local-path` in values files with your existing storage class.

---

## Step 3: Install Loki (SingleBinary – Dev Mode)

### loki-dev-values.yaml

```yaml
deploymentMode: SingleBinary

test:
  enabled: false

loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  storage:
    type: filesystem
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin

singleBinary:
  replicas: 1

read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0

# Disable extra components (dev optimization)
gateway:
  enabled: false

lokiCanary:
  enabled: false

chunksCache:
  enabled: false

resultsCache:
  enabled: false

persistence:
  enabled: true
  size: 10Gi
  storageClass: local-path
```

### Install Loki

```bash
helm install loki grafana/loki \
  -n observability --create-namespace \
  -f loki-dev-values.yaml
```

### Verify

```bash
kubectl get pods -n observability
```

Expected:

* `loki-0` → Running (2/2 containers)

---

## Step 4: Install Grafana

### grafana-dev-values.yaml

```yaml
adminUser: admin
adminPassword: admin

persistence:
  enabled: true
  size: 5Gi
  storageClass: local-path

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki.observability.svc.cluster.local:3100
        isDefault: true
```

### Install Grafana

```bash
helm install grafana grafana/grafana \
  -n observability \
  -f grafana-dev-values.yaml
```

### Access Grafana

```bash
kubectl port-forward svc/grafana 3000:80 -n observability
```

Login:

* **User:** admin
* **Password:** admin

---

## Step 5: Install Fluent Bit (Log Forwarder)

Fluent Bit runs as a **DaemonSet** and tails container logs from each node.

### Fluent Bit Values (Important Sections)

#### Input – Tail Container Logs

```ini
[INPUT]
    Name tail
    Path /var/log/containers/*_dummy_*.log
    multiline.parser docker, cri
    Tag kube.*
    Mem_Buf_Limit 5MB
    Skip_Long_Lines On
```

✔ Reads logs only from `dummy` app pods
✔ Uses CRI/Docker multiline parsing

---

#### Filter – Kubernetes Metadata

```ini
[FILTER]
    Name kubernetes
    Match kube.*
    Merge_Log On
    Keep_Log Off
    K8S-Logging.Parser On
    K8S-Logging.Exclude On
```

✔ Adds namespace, pod, container info
✔ Removes duplicate log fields

---

#### Output – Send Logs to Loki

```ini
[OUTPUT]
    Name loki
    Match kube.*
    host loki
    port 3100
    tenant_id ""
    labels job=fluent-bit, container=$kubernetes['container_name'], namespace=$kubernetes['namespace_name'], pod=$kubernetes['pod_name'], stream=$stream, image=$kubernetes['container_image'], source=$kubernetes['host']
    remove_keys kubernetes,stream
    auto_kubernetes_labels off
    line_format json
```

### Key Settings Explained

| Setting                      | Meaning                     |
| ---------------------------- | --------------------------- |
| `host loki`                  | Uses Kubernetes service DNS |
| `auto_kubernetes_labels off` | Prevents label explosion    |
| `line_format json`           | Structured logs for LogQL   |
| `remove_keys`                | Reduces log payload size    |

---

### Install Fluent Bit

```bash
helm install fluent-bit fluent/fluent-bit \
  -n observability \
  -f fluentbit-values.yaml
```

---

## Step 6: Verify End-to-End Flow

1. Generate logs (dummy / event-simulator app)
2. Open **Grafana → Explore**
3. Select **Loki** datasource
4. Run query:

```logql
{job="fluent-bit"}
```

You should see logs 🎉

---

## Notes on Production vs Dev

| Area        | Dev          | Production        |
| ----------- | ------------ | ----------------- |
| Loki mode   | SingleBinary | Distributed       |
| Replication | 1            | ≥3                |
| Storage     | filesystem   | S3 / GCS / Azure  |
| Labels      | Controlled   | Strict governance |
| Alerts      | Optional     | Mandatory         |

---

## Summary

✔ Loki stores logs
✔ Fluent Bit ships logs
✔ Grafana visualizes & alerts
✔ Storage class depends on cluster

This setup is **perfect for learning, demos, and labs**.

---

Happy Observability 🚀
