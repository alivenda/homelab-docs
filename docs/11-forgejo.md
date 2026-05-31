# Runbook 11: Forgejo

Self-hosted Git server.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30 minutes |
| **Runs On** | k3s (pin to 32 GB node) |
| **Depends On** | Runbook 6 |

Deploy Forgejo via the official Helm chart instead of docker-compose. This keeps the Git server inside the cluster's GitOps lifecycle: persistent volumes go through the NFS provisioner, the workload is visible to ArgoCD, and an HTTPRoute (Runbook 6) gives you HTTPS without exposing NodePorts.

## Step 1: Seal the admin credentials

Don't pass the admin password via `--set` — it ends up in shell history and `ps` output. Seal it first, then reference the Secret from `values.yaml`.

```bash
# Generate
FORGEJO_ADMIN_PW=$(openssl rand -base64 24)

echo "Forgejo admin password: $FORGEJO_ADMIN_PW"   # save to Vaultwarden

# Create the namespace
kubectl create namespace forgejo

# Build a plain Secret manifest, then seal it
kubectl create secret generic forgejo-admin \
  --namespace forgejo \
  --from-literal=username=forgejo_admin \
  --from-literal=password="$FORGEJO_ADMIN_PW" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > forgejo-admin-sealed.yaml

# Commit to homelab-manifests/infrastructure/forgejo/manifests/forgejo-admin-sealed.yaml
```

!!! note "No separate DB secret"
    The chart removed its bundled PostgreSQL sub-chart in v14 — for a homelab single-instance deployment, Forgejo's built-in SQLite is the natural fit and needs no separate credential. If you scale Forgejo or want to share Postgres across services, deploy [bitnami/postgresql](https://artifacthub.io/packages/helm/bitnami/postgresql) separately and point `gitea.config.database` at it.

## Step 2: Install via Helm

`values.yaml` (commit to `homelab-manifests/infrastructure/forgejo/values.yaml`):

```yaml
persistence:
  enabled: true
  storageClass: nfs-storage
  size: 20Gi
service:
  http:
    type: ClusterIP
gitea:
  admin:
    existingSecret: forgejo-admin   # references our SealedSecret
    email: admin@yourdomain.com
  config:
    server:
      DOMAIN: git.yourdomain.com
      ROOT_URL: https://git.yourdomain.com/
      SSH_DOMAIN: git.yourdomain.com
    # SQLite is the chart default; no `database:` block needed for single-instance.
nodeSelector:
  storage: large
```

Install (the chart is published as an OCI artifact — no `helm repo add` needed):

```bash
helm upgrade --install forgejo oci://code.forgejo.org/forgejo-helm/forgejo \
  --version <X.Y.Z> \
  --namespace forgejo --create-namespace \
  --values values.yaml
```

Pin `--version` to a current release listed on [code.forgejo.org/forgejo-helm](https://code.forgejo.org/forgejo-helm/forgejo-helm).

!!! warning "Chart key naming"
    The Forgejo Helm chart inherits Gitea's chart-internal key naming (the `gitea.` prefix), since Forgejo forked it. This is not a bug — check the chart README at [code.forgejo.org/forgejo-helm](https://code.forgejo.org/forgejo-helm/forgejo-helm) for any value name changes if you upgrade the chart later.

## Step 3: GitOps-managed install (recommended)

The CLI install above is a bootstrap. Commit the equivalent ArgoCD `Application` manifest to `homelab-manifests/bootstrap/forgejo.yaml` so ArgoCD reconciles future changes from Git:

```yaml
# homelab-manifests/bootstrap/forgejo.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: forgejo
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: code.forgejo.org/forgejo-helm   # OCI registry, no scheme prefix
      chart: forgejo
      targetRevision: 17.1.0   # pin a version, don't track latest
      helm:
        valueFiles:
          - $values/infrastructure/forgejo/values.yaml
    - repoURL: https://github.com/yourusername/homelab-manifests.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: forgejo
  syncPolicy:
    automated:
      prune: false   # don't orphan the SQLite StatefulSet/PVC if the Application is deleted
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

This pattern uses ArgoCD's [multi-source Application](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#multiple-sources-for-an-application) so the Helm chart and your `values.yaml` can live in different repos.

## Step 4: HTTPRoute (+ SSH)

```yaml
# homelab-manifests/infrastructure/forgejo/manifests/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: forgejo
  namespace: forgejo
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - git.yourdomain.com
  rules:
    - backendRefs:
        - name: forgejo-http
          port: 3000
```

Git-over-SSH is TCP, which the HTTP Gateway can't carry. Expose it with a MetalLB `LoadBalancer` Service instead — simplest is the chart's own `service.ssh.type: LoadBalancer` (MetalLB assigns an IP from the pool). A standalone Service works too (match the selector to your Forgejo pods):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: forgejo-ssh
  namespace: forgejo
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: forgejo
  ports:
    - port: 22
      targetPort: 22
```

!!! note "TCPRoute alternative"
    The Gateway API can route TCP via a `TCPRoute`, but that needs the Gateway API **experimental** channel plus a dedicated TCP listener on the Gateway — neither is configured here (the Gateway has only HTTP/HTTPS listeners). The LoadBalancer Service is the simpler path.

## Step 5: Initial setup

1. Complete the setup wizard at `https://git.yourdomain.com`.
2. Log in as `forgejo_admin` with the password from the SealedSecret.
3. Add your SSH key.
4. Create the `homelab-manifests` repository (or migrate from GitHub).

!!! tip
    If running Woodpecker CI on the same Docker host as Forgejo (NAS-side compose for some deployments), set `ALLOW_LOCALNETWORKS = true` in Forgejo's `app.ini` `[webhook]` section.

## Verification

- [ ] All Forgejo pods Running:

    ```bash
    kubectl get pods -n forgejo
    # Expected: forgejo-0 (1/1 Running). SQLite is in-pod, no separate DB pod.
    ```

- [ ] Web UI reachable at `https://git.yourdomain.com` — login as `forgejo_admin` works.
- [ ] Push a test repo from your machine:

    ```bash
    git remote add forgejo https://git.yourdomain.com/forgejo_admin/test.git
    git push -u forgejo main
    # Expected: push succeeds, repo visible in UI
    ```

- [ ] If you committed the ArgoCD Application manifest: `kubectl get application -n argocd` lists `forgejo` as `Synced/Healthy`.
