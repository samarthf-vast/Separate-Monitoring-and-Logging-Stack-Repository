# Monitoring & Logging Stack

A standalone Kubernetes monitoring repository for the  application. It deploys **Prometheus**, **Grafana**, **Loki**, **Grafana Alloy**, **cAdvisor**, and **Node Exporter** into a dedicated `-monitoring` namespace and is fully automated via GitHub Actions CI/CD.

---

## Stack Overview

| Tool | Role | Port |
|---|---|---|
| Prometheus | Metrics scraping & storage | 9090 |
| Grafana | Dashboards, alerts, email notifications | 3000 |
| Loki | Log aggregation & storage | 3100 |
| Grafana Alloy | Log collector (DaemonSet — ships pod logs to Loki) | — |
| cAdvisor | Container-level resource metrics | 8080 |
| Node Exporter | Host/node-level metrics (CPU, RAM, disk) | 9100 |

---

## Repository Structure

```
monitoring/
├── .github/
│   └── workflows/
│       └── monitoring-deploy.yml   # CI/CD pipeline
├── k8s/
│   ├── namespace.yml               # ''-app + ''-monitoring namespaces
│   ├── configmaps/                 # All configuration files
│   │   ├── prometheus-config.yml
│   │   ├── loki-config.yml
│   │   ├── alloy-config.yml
│   │   ├── grafana-datasources.yml
│   │   ├── grafana-dashboards.yml
│   │   ├── grafana-dashboards-config.yml
│   │   ├── grafana-dashboard-node.yml
│   │   ├── grafana-alerting.yml
│   │   ├── grafana-email-template.yml
│   │   ├── grafana-user-setup.yml
│   │   └── nginx-config.yml
│   └── monitoring/                 # Kubernetes resource manifests
│       ├── prometheus-deployment.yml
│       ├── prometheus-service.yml
│       ├── prometheus-pvc.yml
│       ├── prometheus-rbac.yml
│       ├── grafana-deployment.yml
│       ├── grafana-service.yml
│       ├── grafana-pvc.yml
│       ├── loki-deployment.yml
│       ├── loki-service.yml
│       ├── loki-pvc.yml
│       ├── alloy-daemonset.yml
│       ├── alloy-rbac.yml
│       ├── cadvisor-daemonset.yml
│       ├── cadvisor-service.yml
│       ├── node-exporter-daemonset.yml
│       ├── node-exporter-service.yml
│       └── ingress.yml
└── README.md
```

---

## Prerequisites

- Kubernetes cluster with a self-hosted GitHub Actions runner registered
- `kubectl` configured and pointing to the target cluster
- NGINX Ingress Controller installed in the cluster
- DNS records (or `/etc/hosts` entries) for:
  - `grafana.''app.com`
  - `prometheus.''app.com`

---

## Step-by-Step Setup

### Step 1 — Add GitHub Actions Secrets

In your GitHub repository go to **Settings → Secrets and variables → Actions** and add:

| Secret name | Description |
|---|---|
| `GRAFANA_ADMIN_USER` | Grafana admin username |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin password |
| `GRAFANA_SMTP_PASSWORD` | Gmail app password for alert emails |

---

### Step 2 — Create Namespaces

```bash
kubectl apply -f k8s/namespace.yml
```

This creates two namespaces:
- `''-app` — for the '' application pods
- `''-monitoring` — for all monitoring tools

---

### Step 3 — Create Grafana Secrets

```bash
kubectl create secret generic grafana-secrets \
  --from-literal=GF_SECURITY_ADMIN_USER=<your-user> \
  --from-literal=GF_SECURITY_ADMIN_PASSWORD=<your-password> \
  --from-literal=GF_SMTP_PASSWORD=<your-smtp-password> \
  --namespace=''-monitoring \
  --dry-run=client -o yaml | kubectl apply -f -
```

> The CI/CD pipeline creates this secret automatically from GitHub Actions secrets on every deploy.

---

### Step 4 — Apply ConfigMaps

```bash
kubectl apply --server-side -f k8s/configmaps/
```

`--server-side` is required because some ConfigMaps (e.g. `grafana-dashboards`) exceed the 262 KB client-side annotation limit.

ConfigMaps include:
- Prometheus scrape targets (backend app, node-exporter, cAdvisor)
- Loki storage and schema configuration
- Alloy pipeline (discovers all pods → ships logs to Loki)
- Grafana datasources, pre-built dashboards, alerting rules, and email templates

---

### Step 5 — Apply Monitoring Manifests

```bash
kubectl apply --server-side -f k8s/monitoring/
```

Deploys all components: RBAC, DaemonSets, Deployments, Services, PVCs, and Ingress.

---

### Step 6 — Restart Deployments

```bash
kubectl rollout restart deployment/prometheus -n ''-monitoring
kubectl rollout restart deployment/grafana    -n ''-monitoring
kubectl rollout restart deployment/loki       -n ''-monitoring
```

Restarts pick up any updated ConfigMap values mounted as volumes.

---

### Step 7 — Verify Everything is Running

```bash
kubectl get pods -n ''-monitoring
```

Wait for all pods to show `Running`:

```
NAME                             READY   STATUS    RESTARTS
prometheus-xxxxx                 1/1     Running   0
grafana-xxxxx                    1/1     Running   0
loki-xxxxx                       1/1     Running   0
alloy-xxxxx (per node)           1/1     Running   0
cadvisor-xxxxx (per node)        1/1     Running   0
node-exporter-xxxxx (per node)   1/1     Running   0
```

---

## Accessing the Stack

| Service | URL |
|---|---|
| Grafana | http://grafana.''app.com |
| Prometheus | http://prometheus.''app.com |

Log in to Grafana with the admin credentials from Step 1. Dashboards for **Container Monitoring**, **Logs**, and **Node Exporter** are provisioned automatically.

---

## CI/CD Pipeline

The pipeline in [.github/workflows/monitoring-deploy.yml](.github/workflows/monitoring-deploy.yml) triggers automatically on every push to `master` **only when monitoring files change** (`k8s/monitoring/**`, `k8s/configmaps/**`, `k8s/namespace.yml`).

**Pipeline order (sequential):**

```
1. Apply Namespace
2. Create / update Grafana Secrets
3. Apply ConfigMaps
4. Apply Monitoring Manifests
5. Restart Prometheus, Grafana, Loki
6. Wait for rollout health checks
7. Verify all pods are running
```

The self-hosted runner must have `kubectl` access to the target cluster.

---

## Prometheus Scrape Targets

| Job | Target | What it collects |
|---|---|---|
| `backend` | `backend.''-app.svc.cluster.local:5000` | '' app custom metrics |
| `node` | `node-exporter:9100` | Host CPU, memory, disk, network |
| `cadvisor` | `cadvisor:8080` | Per-container resource usage |

---

## Grafana Alerting

Alerts are configured via provisioned alerting rules and sent by email using Gmail SMTP (`smtp.gmail.com:587`). The email template is customized via `grafana-email-template.yml`.
