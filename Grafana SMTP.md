# Grafana SMTP Alerts with MailHog - Kubernetes Setup

## Overview

This guide describes how to deploy **MailHog** in Kubernetes using **custom Deployment and Service manifests**, and how to configure **Grafana** to send alert emails through MailHog. Ideal for development/test environments.

---

## 1️⃣ Deploy MailHog

### 1.1 MailHog Deployment

Create a file named `mailhog-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mailhog
  labels:
    app: mailhog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mailhog
  template:
    metadata:
      labels:
        app: mailhog
    spec:
      containers:
      - name: mailhog
        image: mailhog/mailhog:latest
        ports:
        - containerPort: 1025  # SMTP
        - containerPort: 8025  # Web UI
```

### 1.2 MailHog Service

Create a file named `mailhog-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mailhog
spec:
  type: NodePort  # Expose UI externally
  selector:
    app: mailhog
  ports:
  - name: smtp
    port: 1025
    targetPort: 1025
    nodePort: 31025
  - name: ui
    port: 8025
    targetPort: 8025
    nodePort: 30025
```

### 1.3 Apply manifests

```bash
kubectl apply -f mailhog-deployment.yaml
kubectl apply -f mailhog-service.yaml
```

### 1.4 Verify service

```bash
kubectl get svc mailhog
```

Expected ports:

* SMTP: 1025 → NodePort 31025
* UI: 8025 → NodePort 30025

---

## 2️⃣ Configure Grafana SMTP

### 2.1 Grafana values.yaml

Edit `grafana-values.yaml` with MailHog SMTP settings:

```yaml
grafana.ini:
  smtp:
    enabled: true
    host: mailhog:1025  # Kubernetes service name
    user: ""             # Not needed for MailHog
    password: ""         # Not needed for MailHog
    from_address: alerts@grafana.local
    from_name: Grafana Alerts
    skip_verify: true      # Ignore TLS verification
```

### 2.2 Upgrade Grafana

```bash
helm upgrade grafana grafana/grafana \
  -n observability \
  -f grafana-values.yaml
```

---

## 3️⃣ Test Grafana Email Alerts

1. Go to **Grafana → Alerting → Contact points**
2. Add **Email** contact point
3. Set **recipient email** to anything (e.g., `test@example.com`)
4. Click **Send Test**

> The email will appear in MailHog UI.

---

## 4️⃣ Access MailHog UI

Open browser at:

```
http://<NODE-IP>:30025
```

* View all received alert emails
* Inspect subject, sender, and message body

---

## ✅ Summary

| Component   | Dev Setup                              |
| ----------- | -------------------------------------- |
| MailHog     | Deployment + NodePort Service          |
| SMTP config | Grafana → MailHog:1025, no credentials |
| UI          | NodePort 30025                         |
| Test email  | Any dummy address                      |

> Safe for testing alerts, no real emails sent.
