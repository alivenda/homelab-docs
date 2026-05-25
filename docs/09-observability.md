# Runbook 9: Prometheus + Grafana + Loki

Full observability for your homelab.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours |
| **Runs On** | k3s cluster |
| **Depends On** | Runbook 5, Runbook 6 |

## Install kube-prometheus-stack

Use a `values.yaml` for chart configuration so you can commit it to `homelab-manifests/infrastructure/kube-prometheus-stack/`:

```yaml
# values.yaml
grafana:
  admin:
    existingSecret: grafana-admin   # see SealedSecret below
prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-storage
          resources:
            requests:
              storage: 20Gi
    nodeSelector:
      storage: large
```

Create a SealedSecret for the Grafana admin password ([pattern from R5 Step 13](05-kubernetes.md#step-13-encrypt-your-first-secret)):

```bash
# Generate and seal
GRAFANA_ADMIN_PW=$(openssl rand -base64 24)
echo "Grafana admin password: $GRAFANA_ADMIN_PW"   # save to Vaultwarden

kubectl create namespace monitoring   # so the seal targets the right ns
kubectl create secret generic grafana-admin \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$GRAFANA_ADMIN_PW" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > infrastructure/kube-prometheus-stack/manifests/grafana-admin-sealed.yaml

# Commit grafana-admin-sealed.yaml to homelab-manifests/infrastructure/kube-prometheus-stack/manifests/
```

Install:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --version <X.Y.Z> \
  --namespace monitoring --create-namespace \
  --values values.yaml
```

Pin `--version` to a current release listed on [prometheus-community/helm-charts](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack). Once GitOps-managed in `homelab-manifests/bootstrap/`, Renovate keeps the pin fresh.

!!! warning "ARM RAM pressure"
    With 8 GB RAM per node, this stack can be memory-hungry. Reduce retention or disable AlertManager if you hit OOM issues. Reference the [Resource Budget table in R0](00-prerequisites.md#resource-budget-expectations) for steady-state numbers.

## Add Loki

The `grafana/loki-stack` chart was deprecated. Use the maintained `grafana/loki` chart (monolithic mode is the right shape for a homelab) and install Promtail from its own chart.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Loki in single-binary mode — `loki-values.yaml`:

```yaml
deploymentMode: SingleBinary
loki:
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem
  auth_enabled: false
  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
singleBinary:
  replicas: 1
  persistence:
    enabled: true
    storageClass: nfs-storage
    size: 10Gi
# Disable the scaled-out components — they don't run in SingleBinary mode
read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0
chunksCache:
  enabled: false
resultsCache:
  enabled: false
```

```bash
helm upgrade --install loki grafana/loki \
  --version <X.Y.Z> \
  --namespace monitoring \
  --values loki-values.yaml
```

Promtail (separate chart now):

```bash
helm upgrade --install promtail grafana/promtail \
  --version <X.Y.Z> \
  --namespace monitoring \
  --set config.clients[0].url=http://loki:3100/loki/api/v1/push
```

Pin `--version` for both: see [grafana/helm-charts](https://github.com/grafana/helm-charts/tree/main/charts) for current `loki` and `promtail` releases.

## Grafana Dashboards

Import these from the Grafana UI (Dashboards → Import → paste the ID):

| ID | Name |
|---|---|
| 1860 | Node Exporter Full |
| 11315 | UniFi Poller |
| 13502 | Kubernetes cluster monitoring |
| 15489 | Loki log dashboard |

!!! tip
    "I built monitoring and alerting for a multi-node cluster" is materially stronger than "I installed Grafana" on a resume.

## Verification

- [ ] Grafana UI reachable at `https://grafana.yourdomain.com`. Initial login: `admin` / password from the SealedSecret.
- [ ] Dashboard 1860 (Node Exporter Full) loads and shows all 4 cubes with live CPU / memory / disk metrics.
- [ ] Loki receives logs from the cluster. In Grafana Explore, query `{namespace="argocd"}` — should return recent log lines.
- [ ] All pods Running:

    ```bash
    kubectl get pods -n monitoring
    # Expected: prometheus-*, grafana-*, loki-0, promtail-* all Running
    ```
