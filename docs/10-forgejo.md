# Runbook 10: Forgejo

Self-hosted Git server.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30 minutes |
| **Runs On** | k3s (pin to 32 GB node) |
| **Depends On** | Runbook 6 |

Deploy Forgejo via the official Helm chart instead of docker-compose. This keeps the Git server inside the cluster's GitOps lifecycle: persistent volumes go through the NFS provisioner, the workload is visible to ArgoCD, and an IngressRoute (Runbook 6) gives you HTTPS without exposing NodePorts.

## Step 1: Seal the admin and DB credentials

Don't pass passwords via `--set` — they end up in shell history and `ps` output. Seal them first, then reference the Secret from `values.yaml`.

```bash
# Generate
FORGEJO_ADMIN_PW=$(openssl rand -base64 24)
FORGEJO_DB_PW=$(openssl rand -base64 24)

echo "Forgejo admin password: $FORGEJO_ADMIN_PW"   # save to Vaultwarden
echo "Forgejo DB password: $FORGEJO_DB_PW"         # save to Vaultwarden

# Create the namespace
kubectl create namespace forgejo

# Build a plain Secret manifest, then seal it
kubectl create secret generic forgejo-credentials \
  --namespace forgejo \
  --from-literal=admin-password="$FORGEJO_ADMIN_PW" \
  --from-literal=db-password="$FORGEJO_DB_PW" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > forgejo-credentials-sealed.yaml

# Commit to homelab-manifests/apps/forgejo/forgejo-credentials-sealed.yaml
```

## Step 2: Install via Helm

`values.yaml` (commit to `homelab-manifests/apps/forgejo/values.yaml`):

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
    username: forgejo_admin
    existingSecret: forgejo-credentials   # references our SealedSecret
    email: admin@yourdomain.com
  config:
    server:
      DOMAIN: git.yourdomain.com
      ROOT_URL: https://git.yourdomain.com/
      SSH_DOMAIN: git.yourdomain.com
postgresql:
  enabled: true
  auth:
    existingSecret: forgejo-credentials
    secretKeys:
      adminPasswordKey: db-password
      userPasswordKey: db-password
  global:
    storageClass: nfs-storage
nodeSelector:
  storage: large
```

Install:

```bash
helm repo add forgejo https://code.forgejo.org/forgejo-helm
helm repo update
helm install forgejo forgejo/forgejo \
  --namespace forgejo \
  --values values.yaml
```

!!! warning "Chart key naming"
    The Forgejo Helm chart inherits Gitea's chart-internal key naming (the `gitea.` prefix), since Forgejo forked it. This is not a bug — check chart docs at [code.forgejo.org/forgejo-helm](https://code.forgejo.org/forgejo-helm/forgejo-helm) for any value name changes if you upgrade the chart later.

## Step 3: GitOps-managed install (recommended)

The `helm install` above is a bootstrap. Commit the equivalent ArgoCD `Application` manifest to `homelab-manifests/apps/forgejo/application.yaml` so ArgoCD reconciles future changes from Git:

```yaml
# homelab-manifests/apps/forgejo/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: forgejo
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://code.forgejo.org/forgejo-helm
      chart: forgejo
      targetRevision: 9.0.0   # pin a version, don't track latest
      helm:
        valueFiles:
          - $values/apps/forgejo/values.yaml
    - repoURL: https://github.com/yourusername/homelab-manifests.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: forgejo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

This pattern uses ArgoCD's [multi-source Application](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#multiple-sources-for-an-application) so the Helm chart and your `values.yaml` can live in different repos.

## Step 4: IngressRoute

```yaml
# homelab-manifests/apps/forgejo/ingressroute.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: forgejo
  namespace: forgejo
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`git.yourdomain.com`)
      kind: Rule
      services:
        - name: forgejo-http
          port: 3000
  tls:
    certResolver: cloudflare
    domains:
      - main: git.yourdomain.com
---
# Optional: TCP route for git-over-SSH
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: forgejo-ssh
  namespace: forgejo
spec:
  entryPoints: [ssh]   # add this entrypoint to Traefik values.yaml if you want SSH on 2222
  routes:
    - match: HostSNI(`*`)
      services:
        - name: forgejo-ssh
          port: 22
```

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
    # Expected: forgejo-0 + forgejo-postgresql-0 all 1/1 Running
    ```

- [ ] Web UI reachable at `https://git.yourdomain.com` — login as `forgejo_admin` works.
- [ ] Push a test repo from your laptop:

    ```bash
    git remote add forgejo https://git.yourdomain.com/forgejo_admin/test.git
    git push -u forgejo main
    # Expected: push succeeds, repo visible in UI
    ```

- [ ] If you committed the ArgoCD Application manifest: `kubectl get application -n argocd` lists `forgejo` as `Synced/Healthy`.
