# Runbook 8: Prometheus + Grafana + Loki

Full observability for your homelab.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours |
| **Runs On** | k3s cluster |
| **Depends On** | Runbook 5, Runbook 6 |

## Install kube-prometheus-stack

Use a `values.yaml` for chart configuration so you can commit it to `homelab-manifests/apps/monitoring/`:

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
  > monitoring/grafana-admin-sealed.yaml

# Commit grafana-admin-sealed.yaml to homelab-manifests/apps/monitoring/
```

Install:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml
```

!!! warning "ARM RAM pressure"
    With 8 GB RAM per node, this stack can be memory-hungry. Reduce retention or disable AlertManager if you hit OOM issues. Reference the [Resource Budget table in R0](00-prerequisites.md#resource-budget-expectations) for steady-state numbers.

## Add Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.storageClassName=nfs-storage \
  --set loki.persistence.size=10Gi
```

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
    # Expected: prometheus-server, grafana, loki, promtail all Running
    ```
