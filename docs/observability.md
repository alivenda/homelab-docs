# Observability

Metrics, logs, dashboards, and alerts for the whole homelab.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours |
| **Runs On** | k3s (cluster), namespace `monitoring` |
| **Depends On** | Kubernetes (k3s), Traefik, Terraform (DNS). The alerting hookup (Step 4) returns here after ntfy. |

## The shape of the stack

Three pipelines, three ArgoCD Applications, one namespace:

```text
metrics  node-exporter (DaemonSet) + kube-state-metrics + kubelet/apiserver
           ──scrape──> Prometheus ──> Grafana dashboards
logs     /var/log/pods on every node
           ──tail──> Alloy (DaemonSet) ──push──> Loki <──query── Grafana Explore
alerts   Prometheus rules ──> Alertmanager ──webhook──> ntfy ──> your phone
```

- **`kube-prometheus-stack`** (Prometheus + Alertmanager + Grafana + the Prometheus
  Operator and its CRDs) from `prometheus-community/helm-charts`, plus a sibling
  raw-manifests Application for the sealed secrets, HTTPRoute, and alert rules.
- **`loki`** — the log store, from `grafana-community/helm-charts`.
- **`alloy`** — the log collector, from `grafana/helm-charts`.

!!! warning "The Grafana chart repos split — check you're pulling from the right one"
    `grafana.github.io/helm-charts` is winding down: the **loki** chart is frozen
    there (at 7.0.0) and its maintained lineage moved to
    [`grafana-community.github.io/helm-charts`](https://github.com/grafana-community/helm-charts)
    (same chart, renumbered at 17.x). **Promtail is EOL since March 2026** — do not
    deploy it; [Grafana Alloy](https://grafana.com/docs/alloy/latest/) is its
    successor as the log collector. Alloy itself, being an active Grafana product,
    **stayed** in `grafana.github.io/helm-charts`. Three components, two repos —
    easy to get wrong.

## Before you start: fix node clock sync

DietPi's default time sync is a **oneshot** systemd-timesyncd run, so the kernel's
"clock synchronised" flag is unset almost all the time and the CM4s (which have no
RTC) free-run between pokes. Prometheus notices: the stack's default
`NodeClockNotSynchronising` alert fires on **every node, permanently**. The fix is a
real NTP daemon, codified in `homelab-ansible` (`site.yml`, `--tags time`): set
`CONFIG_NTP_MODE=0` in `/boot/dietpi.txt` (DietPi's documented "custom" mode for
running your own daemon) and install `chrony` — apt removes systemd-timesyncd in the
same transaction, so no second daemon fights over the clock. Run that play before
deploying the stack and the alert never fires.

## Step 1: kube-prometheus-stack

### The Applications

Two ArgoCD Applications in `bootstrap/kube-prometheus-stack.yaml`, exactly the
chart-plus-manifests split Forgejo and Woodpecker use:

```yaml
# bootstrap/kube-prometheus-stack.yaml (chart Application — abridged)
spec:
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 85.3.3          # pin a version; Renovate bumps it
      helm:
        releaseName: kube-prometheus-stack
        valueFiles:
          - $values/infrastructure/kube-prometheus-stack/values.yaml
    - repoURL: https://github.com/<you>/homelab-manifests.git
      targetRevision: HEAD
      ref: values
  destination:
    namespace: monitoring
  syncPolicy:
    automated:
      prune: false       # see below
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true   # the CRDs exceed kubectl's 262 KiB annotation limit
```

Two settings here are load-bearing:

- **`prune: false`** — pruning would drop the Prometheus Operator CRDs
  (ServiceMonitor, PodMonitor, PrometheusRule, …) and silently break every scrape
  config that depends on them. Bump the chart deliberately.
- **`ServerSideApply=true`** — the stack's CRDs are too large for the
  client-side-apply annotation; without SSA the sync fails outright.

The second Application (`kube-prometheus-stack-manifests`) points at
`infrastructure/kube-prometheus-stack/manifests/` — the sealed Grafana admin
secret, the Grafana HTTPRoute, the alert rules and the sealed ntfy token from
Step 4. Give it `CreateNamespace=true` too: the SealedSecret carries
`namespace: monitoring`, so without it the manifests app fails until the chart app
happens to create the namespace first. (And per the Forgejo/Woodpecker pattern, **no**
`ServerSideApply` on the manifests app — the HTTPRoute trap.)

### Grafana — stateless on purpose

```yaml
# infrastructure/kube-prometheus-stack/values.yaml (Grafana section)
grafana:
  admin:
    existingSecret: grafana-admin   # SealedSecret, below
  persistence:
    enabled: false
  grafana.ini:
    server:
      root_url: https://grafana.yourdomain.com
  additionalDataSources:
    - name: Loki
      type: loki
      uid: loki
      access: proxy
      url: http://loki.monitoring.svc.cluster.local:3100
```

`persistence: enabled: false` is deliberate, not an oversight. Grafana's own state
store (`grafana.db`) is **SQLite**, which `nfs-storage` corrupts (see
[Storage & Data Architecture](storage-architecture.md)) — but rather than move the
PVC to `local-path`, keep no state at all: dashboards and datasources are
provisioned by the chart and its sidecar, alerting lives in
PrometheusRule + Alertmanager, and login comes from the sealed admin secret.
Nothing in Grafana is worth backing up; a rebuilt pod re-provisions everything from
git. The one consequence: **dashboards built in the UI are a scratchpad** — export
the JSON and commit it, or it's gone on the next pod restart.

`root_url` matters behind a proxy: Grafana bakes it into redirects, rendered links,
and the OIDC callback — wrong or unset breaks login flows through Traefik.

The Loki datasource is added by hand because Loki is a separate chart; Prometheus
and Alertmanager datasources are provisioned automatically. It points at the
service port directly — Loki's nginx gateway is disabled (Step 2).

Seal the admin login ([pattern from Kubernetes Step 13](kubernetes.md#step-13-encrypt-your-first-secret)):

```bash
GRAFANA_ADMIN_PW=$(openssl rand -base64 24)
echo "Grafana admin password: $GRAFANA_ADMIN_PW"   # save to your password manager

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
```

### Prometheus — TSDB on the SSD, loss-tolerant

```yaml
# values.yaml (Prometheus section)
prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: 18GiB
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          resources:
            requests:
              storage: 20Gi
    nodeSelector:
      kubernetes.io/hostname: topaz
    resources:
      requests:
        cpu: 200m
        memory: 1Gi
      limits:
        memory: 2Gi
```

- **Why `local-path`, not `nfs-storage`:** Prometheus's TSDB on NFS is explicitly
  unsupported upstream (non-POSIX semantics can corrupt blocks). The `local-path`
  class plus the topaz pin lands the volume on the SATA SSD via the provisioner's
  `nodePathMap` — the right disk, POSIX-correct.
- **Loss-tolerant by design:** metrics history is *not* backed up. A rebuild starts
  an empty TSDB and that's fine — dashboards refill within the retention window.
- **`retentionSize` below the PVC size:** Prometheus deletes its oldest blocks past
  18 GiB, and the 2 GiB of slack absorbs WAL + compaction churn so the volume never
  actually fills.
- **The resource limit protects the node:** this is the heaviest pod in the
  cluster (~1 Gi steady-state at this size). The 2 Gi cap keeps a label-cardinality
  blow-up from OOMing topaz — which also serves NFS for everything else.

!!! warning "ARM RAM pressure"
    With 8 GB per node this stack is the heaviest single workload in the cluster.
    If topaz struggles, drop `retention` first, then consider disabling
    Alertmanager. Reference the
    [Resource Budget table in Prerequisites](prerequisites.md#resource-budget-expectations)
    for steady-state numbers.

### Turn off the scrapes k3s can't serve

```yaml
# values.yaml
kubeControllerManager:
  enabled: false
kubeScheduler:
  enabled: false
kubeProxy:
  enabled: false
kubeEtcd:
  enabled: false
```

k3s runs the control-plane components (and etcd) **inside the single k3s process**
— the standalone pods these scrape jobs select never exist on this cluster. The
chart gates each component's default alert rules on the same flag, so this also
removes `KubeControllerManagerDown` / `KubeSchedulerDown` / `KubeProxyDown` —
which would otherwise be permanent false positives. Node, kubelet, and apiserver
scraping are separate components and stay on.

## Step 2: Loki

One Application (`bootstrap/loki.yaml`), chart `loki` from
**`grafana-community.github.io/helm-charts`** (see the repo-split warning above),
same namespace as the stack so Grafana reaches it without cross-namespace DNS.
`prune: false` again — pruning would drop the StatefulSet mid-flight.

??? example "infrastructure/loki/values.yaml"

    ```yaml
    deploymentMode: Monolithic   # the chart's current name for SingleBinary

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
      compactor:
        retention_enabled: true
        delete_request_store: filesystem   # mandatory once retention is on
      limits_config:
        retention_period: 14d

    singleBinary:
      replicas: 1
      persistence:
        enabled: true
        storageClass: local-path
        size: 10Gi
      nodeSelector:
        kubernetes.io/hostname: topaz

    # Scaled-out components don't run in Monolithic mode — zero them out.
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

    gateway:
      enabled: false
    lokiCanary:
      enabled: false
    test:
      enabled: false
    ```

The decisions, briefly:

- **Monolithic mode** is the right shape for a homelab: one pod, one PVC, no object
  store. A few hundred MB of logs a day does not need microservices.
- **Retention must be explicit.** Loki keeps logs *forever* by default — without
  the compactor block the PVC just fills. The compactor runs inside the single
  binary and enforces the 14-day window.
- **Same storage treatment as Prometheus:** Loki's TSDB index + WAL have the same
  non-POSIX trouble on NFS, so `local-path` + the topaz pin. Logs are
  loss-tolerant; nothing here is backed up.
- **Gateway, canary, and helm-test off:** single tenant, single binary — clients
  hit `loki:3100` directly, the nginx gateway is a pointless hop, and a per-node
  canary DaemonSet is too much footprint for four CM4s.

## Step 3: Alloy — the log collector

One Application (`bootstrap/alloy.yaml`), chart `alloy` from
**`grafana.github.io/helm-charts`** (it stayed put), `prune: false` so a sync never
rolls the DaemonSet across every node at once. Alloy tails `/var/log/pods` on each
node and pushes to Loki:

??? example "infrastructure/alloy/values.yaml"

    ```yaml
    alloy:
      configMap:
        content: |
          // Discover pods on THIS node only — each DaemonSet pod tails its own
          // node's files. K8S_NODE_NAME is injected by the chart (downward API).
          discovery.kubernetes "pods" {
            role = "pod"
            selectors {
              role  = "pod"
              field = "spec.nodeName=" + sys.env("K8S_NODE_NAME")
            }
          }

          discovery.relabel "pods" {
            targets = discovery.kubernetes.pods.targets

            rule {
              source_labels = ["__meta_kubernetes_namespace"]
              target_label  = "namespace"
            }
            rule {
              source_labels = ["__meta_kubernetes_pod_name"]
              target_label  = "pod"
            }
            rule {
              source_labels = ["__meta_kubernetes_pod_container_name"]
              target_label  = "container"
            }
            rule {
              source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
              target_label  = "app"
            }
            rule {
              source_labels = ["__meta_kubernetes_pod_node_name"]
              target_label  = "node"
            }
            // Map each container to its log file on the host:
            // /var/log/pods/<namespace>_<pod>_<uid>/<container>/*.log
            rule {
              source_labels = ["__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
              separator     = "/"
              replacement   = "/var/log/pods/*$1/*.log"
              target_label  = "__path__"
            }
          }

          local.file_match "pods" {
            path_targets = discovery.relabel.pods.output
          }

          loki.source.file "pods" {
            targets    = local.file_match.pods.targets
            forward_to = [loki.process.pods.receiver]
          }

          loki.process "pods" {
            // k3s uses containerd → CRI log format; strip the wrapper.
            stage.cri {}
            forward_to = [loki.write.loki.receiver]
          }

          loki.write "loki" {
            endpoint {
              url = "http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push"
            }
          }

      mounts:
        varlog: true
        extra:
          - name: positions
            mountPath: /var/lib/alloy

      securityContext:
        runAsUser: 0          # pod log files on the host are root-owned

      storagePath: /var/lib/alloy

    controller:
      type: daemonset         # the config above only works one-collector-per-node
      volumes:
        extra:
          - name: positions
            hostPath:
              path: /var/lib/alloy
              type: DirectoryOrCreate
    ```

What's deliberate here:

- **File tailing, not the kubelet API:** `loki.source.file` over `/var/log/pods`
  is the GA pipeline; `loki.source.kubernetes` (kubelet API) is still
  public-preview.
- **`stage.cri`** strips the `timestamp stream flags` wrapper containerd puts on
  every line.
- **Positions on a hostPath:** Alloy records how far it has read under
  `storagePath`. The chart default is an emptyDir — every pod restart would
  re-ingest every log file from the beginning. The `/var/lib/alloy` hostPath keeps
  positions across restarts (the same trick the old promtail chart used with
  `/run/promtail`).

## Step 4: Alerting → ntfy

!!! note "Come back to this step after ntfy"
    Alertmanager publishes to the ntfy server, which doesn't exist until ntfy. The
    stack runs fine without it — alerts just have nowhere to go yet.

Alertmanager is configured in the same kube-prometheus-stack `values.yaml`. The
routing: everything lands on ntfy (topic `alerts`) except the `Watchdog`
heartbeat, which is *meant* to fire forever and goes to a null receiver.

??? example "values.yaml (Alertmanager section)"

    ```yaml
    alertmanager:
      config:
        route:
          group_by: ['namespace', 'alertname']
          group_wait: 30s
          group_interval: 5m
          repeat_interval: 12h
          receiver: ntfy
          routes:
            - receiver: 'null'
              matchers:
                - alertname = "Watchdog"
        receivers:
          - name: 'null'
          - name: ntfy
            webhook_configs:
              - url: http://ntfy.ntfy.svc.cluster.local/alerts?template=alertmanager
                send_resolved: true
                http_config:
                  authorization:
                    type: Bearer
                    credentials_file: /etc/alertmanager/secrets/alertmanager-ntfy-token/token
      alertmanagerSpec:
        secrets:
          - alertmanager-ntfy-token   # mounted at /etc/alertmanager/secrets/<name>/
        storage:
          volumeClaimTemplate:
            spec:
              storageClassName: nfs-storage
              resources:
                requests:
                  storage: 2Gi
    ```

The details that matter:

- **In-cluster URL on purpose** — alert delivery keeps working when ingress or DNS
  is down, exactly when you need alerts most. `?template=alertmanager` is ntfy's
  built-in formatter for Alertmanager webhook payloads (firing/resolved).
- **The token never appears in git:** it rides in a SealedSecret and Alertmanager
  reads it from the mounted file (`credentials_file`), not inline config. Seal the
  `alertmanager` publisher token you provisioned in ntfy:

    ```bash
    kubectl create secret generic alertmanager-ntfy-token \
      --namespace monitoring \
      --from-literal=token=tk_... \
      --dry-run=client -o yaml \
      | kubeseal --controller-name=sealed-secrets-controller \
                 --controller-namespace=sealed-secrets \
                 --format yaml \
      > infrastructure/kube-prometheus-stack/manifests/alertmanager-ntfy-token-sealed.yaml
    ```

- **Helm merges this config shallowly:** the chart's default `inhibit_rules`
  (critical mutes warning, etc.) survive, but `route` and `receivers` replace the
  defaults wholesale — which is why both are stated in full.
- **Alertmanager's PVC stays on `nfs-storage`** — accepted exception: its state is
  plain snapshot files (silences, notification log), not SQLite or a TSDB, and
  losing it only risks a duplicate notification or an expired silence firing once.

### Custom alert rules

The chart's default rules already cover node down, PVC pressure, crash loops, and
scrape-target down. What they *can't* know about is your backup jobs and
certificates — a `PrometheusRule` in the manifests Application fills the gap
(deploy it after Backups, which provides the velero metrics):

??? example "manifests/alert-rules.yaml (abridged)"

    ```yaml
    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: homelab-alerts
      namespace: monitoring
      labels:
        release: kube-prometheus-stack   # REQUIRED — see warning below
    spec:
      groups:
        - name: homelab.velero
          rules:
            - alert: VeleroBackupFailed
              # 1 = last run Completed; 0 covers Failed and PartiallyFailed.
              expr: velero_backup_last_status{schedule!=""} == 0
              for: 15m
              labels:
                severity: critical
            - alert: VeleroBackupOverdue
              # Daily schedule → >26h without a success means a missed/failing run.
              # The > 0 guard skips the gauge's uninitialized state after a restart.
              expr: >-
                velero_backup_last_successful_timestamp{schedule!=""} > 0
                and time() - velero_backup_last_successful_timestamp{schedule!=""} > 26 * 3600
              for: 15m
              labels:
                severity: warning
        - name: homelab.cert-manager
          rules:
            - alert: CertManagerCertNotReady
              # Renewal hiccups self-heal in minutes; 1h of not-Ready is a stuck
              # issuer/challenge that needs a human.
              expr: max by(namespace, name) (certmanager_certificate_ready_status{condition="False"}) == 1
              for: 1h
              labels:
                severity: warning
    ```

!!! warning "Rules without the release label silently never load"
    kube-prometheus-stack's default `ruleSelector` only loads PrometheusRules
    carrying its Helm release label (`ruleSelectorNilUsesHelmValues`) — the same
    gotcha as ServiceMonitors. Omit `release: kube-prometheus-stack` and the rule
    applies cleanly, shows up in `kubectl get prometheusrules`, and is never
    evaluated.

### Black-box probes (blackbox-exporter)

Prometheus only scrapes `/metrics` — it can't dial an arbitrary TCP port or exercise a
public URL end-to-end. A small **blackbox-exporter** Application
(`infrastructure/blackbox-exporter`, raw manifests) fills both gaps with declarative
`Probe` CRDs feeding the same rule → Alertmanager → ntfy pipeline:

- **NAS Postgres reachability** — a bare TCP connect to the shared database server
  every 30s. Five apps share that server, so `NasPostgresDown` (critical) turns
  "everything just broke" into one root-cause alert. The probe never authenticates —
  no DB credentials touch the monitoring path.
- **Public endpoints** — an HTTPS GET of the published `*.yourdomain.com` URLs every
  60s, exercising the full external path (DNS → Traefik → TLS → Authelia → app).
  `PublicEndpointDown` and `PublicEndpointCertExpiringSoon` (both warning) catch broken
  routes and the served wildcard cert nearing expiry. These probes replaced Uptime Kuma
  — see the [App Catalog retirement note](apps-catalog.md#uptime-kuma).

Like the custom rules above, `Probe` objects are only selected if they carry the
`release: kube-prometheus-stack` label.

## Step 5: DNS + HTTPRoute

1. Add `grafana` to the service list in the Cloudflare [Terraform](terraform.md) module
   and apply — one A record per service, no wildcard.
2. HTTPRoute in the manifests Application, same shape as every other service:

```yaml
# manifests/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: monitoring
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - grafana.yourdomain.com
  rules:
    - backendRefs:
        - name: kube-prometheus-stack-grafana
          port: 80
```

Prometheus, Alertmanager, and Loki get **no** route — they have no auth story of
their own and nothing on the outside needs them. Reach them with
`kubectl port-forward` when debugging.

## Grafana dashboards

Import from the Grafana UI (Dashboards → Import → paste the ID):

| ID | Name |
|---|---|
| 1860 | Node Exporter Full |
| 13502 | Kubernetes cluster monitoring |
| 15489 | Loki log dashboard |

Remember: Grafana is stateless here. The UI is a scratchpad — anything you build
and want to keep, export as JSON and provision it from git, or re-import after the
next pod restart.

!!! tip
    "I built monitoring and alerting for a multi-node cluster" is materially
    stronger than "I installed Grafana" on a resume.

## Verification

- [ ] All pods Running:

    ```bash
    kubectl get pods -n monitoring
    # Expected: prometheus-*, alertmanager-*, grafana, operator,
    # kube-state-metrics, node-exporter ×4, loki-0, alloy ×4,
    # blackbox-exporter — all Running
    ```

- [ ] Grafana reachable at `https://grafana.yourdomain.com`; login `admin` /
  password from the SealedSecret.
- [ ] Dashboard 1860 (Node Exporter Full) shows all 4 nodes with live CPU /
  memory / disk metrics.
- [ ] Logs flow: Grafana Explore → Loki datasource → `{namespace="argocd"}`
  returns recent lines.
- [ ] No false positives: the Alerting page shows **only `Watchdog`** firing — no
  `KubeControllerManagerDown` / `KubeSchedulerDown` / `KubeProxyDown` (the k3s
  scrape disables) and no `NodeClockNotSynchronising` (chrony).
- [ ] *(After ntfy)* A test alert reaches your phone:

    ```bash
    kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093 &
    curl -XPOST http://localhost:9093/api/v2/alerts \
      -H "Content-Type: application/json" \
      -d '[{"labels":{"alertname":"TestAlert","severity":"warning"}}]'
    # → an ntfy notification on the `alerts` topic within ~30s (group_wait)
    ```
