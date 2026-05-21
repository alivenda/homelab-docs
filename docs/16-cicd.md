# Runbook 16: CI/CD with Forgejo and Woodpecker CI

End-to-end pipeline: push code, auto-build container images, deploy to k3s via GitOps.

| | |
|---|---|
| **Difficulty** | Intermediate–Advanced |
| **Time Estimate** | 2–3 hours |
| **Runs On** | k3s cluster (32 GB node) |
| **Depends On** | Runbook 5, 6, 10 |

## Step 1: OAuth2 App in Forgejo

**Site Admin → Applications → OAuth2 Applications.**

- **Redirect URI:** `https://ci.yourdomain.com/authorize`
- **Application Name:** `Woodpecker`

Save the **Client ID** and **Client Secret** to Vaultwarden.

!!! warning "Webhook delivery from Forgejo to Woodpecker"
    If Woodpecker runs on the same cluster as Forgejo, you may need to set `ALLOW_LOCALNETWORKS = true` in Forgejo's `app.ini` `[webhook]` section. Otherwise Forgejo refuses to call out to the cluster-internal Woodpecker URL.

## Step 2: Generate the agent secret

```bash
WOODPECKER_AGENT_SECRET=$(openssl rand -hex 32)
echo "Woodpecker agent secret: $WOODPECKER_AGENT_SECRET"   # save to Vaultwarden
```

## Step 3: Seal Woodpecker credentials

```bash
kubectl create namespace woodpecker

kubectl create secret generic woodpecker-secrets \
  --namespace woodpecker \
  --from-literal=forgejo-client-id="<CLIENT_ID_FROM_STEP_1>" \
  --from-literal=forgejo-client-secret="<CLIENT_SECRET_FROM_STEP_1>" \
  --from-literal=agent-secret="$WOODPECKER_AGENT_SECRET" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > woodpecker-secrets-sealed.yaml

# Commit to homelab-manifests/apps/woodpecker/
```

## Step 4: Deploy Woodpecker via Helm

`values.yaml`:

```yaml
server:
  env:
    WOODPECKER_HOST: https://ci.yourdomain.com
    WOODPECKER_FORGEJO: "true"
    WOODPECKER_FORGEJO_URL: https://git.yourdomain.com
    WOODPECKER_OPEN: "false"
    WOODPECKER_ADMIN: yourforgejousername
  envFrom:
    - secretRef:
        name: woodpecker-secrets   # provides FORGEJO_CLIENT, FORGEJO_SECRET, AGENT_SECRET
  persistentVolume:
    enabled: true
    storageClass: nfs-storage
  service:
    type: ClusterIP

agent:
  env:
    WOODPECKER_BACKEND: kubernetes
  envFrom:
    - secretRef:
        name: woodpecker-secrets
  replicaCount: 2

nodeSelector:
  storage: large
```

Map the secret keys to the env var names Woodpecker expects. For Helm charts that don't honor `envFrom` directly on a per-key basis, use `existingSecret` or `extraEnv` to wire `WOODPECKER_FORGEJO_CLIENT` ← `forgejo-client-id`, etc. — check the chart docs.

Install:

```bash
helm repo add woodpecker https://woodpecker-ci.org/
helm repo update
helm install woodpecker woodpecker/woodpecker \
  --namespace woodpecker \
  --values values.yaml
```

!!! warning "FORGEJO vs GITEA env var names"
    Modern Woodpecker (>= 2.x) ships a dedicated Forgejo provider (`WOODPECKER_FORGEJO_*`). Older versions only had `WOODPECKER_GITEA_*` which still works against Forgejo (same API surface). If your chart version is older, swap `FORGEJO` → `GITEA` in the env-var names.

## Step 5: IngressRoute

Standard IngressRoute for `ci.yourdomain.com`. Same shape as [Vaultwarden Step 3](11-vaultwarden.md#step-3-ingressroute) — change the service name to `woodpecker-server` and port to `8000`.

## Step 6: Why kubernetes backend (not docker)

- Each pipeline step becomes its own pod, scheduled by k3s. Resource limits and node selectors work.
- **No `/var/run/docker.sock` mount in the agent.** The original compose mount was a privileged escape risk. The kubernetes backend doesn't need it — image builds happen in dedicated build pods.
- Cross-architecture builds: pin steps to amd64 or arm64 via `nodeSelector` on the build pod if you ever add x86 nodes.

!!! warning "Pipeline images must be ARM64"
    All pipeline images must be ARM64-compatible. Most official images publish arm64 builds — verify third-party plugins. See [Reality of ARM64 Homelabs](00-prerequisites.md#reality-of-arm64-homelabs) for debugging strategy.

## Sample Pipeline (build → push → bump manifest)

The kubernetes backend doesn't have a docker socket, so use [kaniko](https://github.com/GoogleContainerTools/kaniko) to build OCI images without Docker:

```yaml
# In your app repo: .woodpecker.yml
when:
  - event: push
    branch: main

steps:
  build-and-push:
    image: gcr.io/kaniko-project/executor:latest
    commands:
      - /kaniko/executor
          --context=$CI_WORKSPACE
          --dockerfile=Dockerfile
          --destination=git.yourdomain.com/youruser/myapp:${CI_COMMIT_SHA}
          --destination=git.yourdomain.com/youruser/myapp:latest

  update-manifest:
    image: alpine/git:latest
    commands:
      - git config user.email ci@yourdomain.com
      - git config user.name "Woodpecker CI"
      - git clone https://${FORGEJO_TOKEN}@git.yourdomain.com/youruser/homelab-manifests.git
      - cd homelab-manifests
      - |
        sed -i "s|image: .*myapp:.*|image: git.yourdomain.com/youruser/myapp:${CI_COMMIT_SHA}|" \
          apps/myapp/deployment.yaml
      - git commit -am "ci(myapp): bump to ${CI_COMMIT_SHA}"
      - git push
    secrets: [ forgejo_token ]
```

!!! note "Why kaniko over docker-in-docker"
    Kaniko builds OCI images entirely in user space — no daemon, no privileged container, no socket mount. It's the standard pattern for image builds in any kubernetes-native CI system. For Forgejo registry auth, mount the credentials in `/kaniko/.docker/config.json` via a Secret (omitted from the sample for brevity).

!!! tip "Forgejo's built-in container registry"
    Enable Forgejo's container registry under `[packages]` in `app.ini` so pipelines push/pull images without Docker Hub.

## Renovate: keep third-party images current

Your CI pipeline above builds your own images. Third-party images (postgres, redis, paperless-ngx, immich, etc.) need a different update strategy. [Renovate](https://docs.renovatebot.com/) watches your manifests for outdated image tags and opens PRs to bump them. Self-host it as a Woodpecker pipeline, or pull from the official runner.

```json
// homelab-manifests/renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "kubernetes": {
    "fileMatch": ["apps/.*/.*\\.ya?ml$"]
  },
  "helm-values": {
    "fileMatch": ["apps/.*/values\\.ya?ml$"]
  },
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": false,
      "labels": ["renovate"]
    }
  ]
}
```

Run Renovate as a scheduled Woodpecker pipeline in the `homelab-manifests` repo:

```yaml
# homelab-manifests/.woodpecker/renovate.yml
when:
  - event: cron
    cron: renovate

steps:
  renovate:
    image: ghcr.io/renovatebot/renovate:latest
    environment:
      RENOVATE_PLATFORM: gitea         # Forgejo speaks Gitea API
      RENOVATE_ENDPOINT: https://git.yourdomain.com
      RENOVATE_TOKEN:
        from_secret: renovate_token
      RENOVATE_AUTODISCOVER: "true"
      LOG_LEVEL: info
    secrets: [ renovate_token ]
```

In Woodpecker, schedule the cron `renovate` to run nightly. Renovate scans `homelab-manifests`, opens PRs for outdated image tags, and you review/merge them. ArgoCD reconciles after the merge.

!!! tip
    Renovate also handles helm chart versions, terraform provider versions, GitHub Actions, and dozens of other ecosystems. If you set `RENOVATE_AUTODISCOVER=true`, it walks every repo the token has access to. Scope the token to `homelab-manifests` and `homelab-terraform` if you want it limited.

## Container registry: Forgejo built-in vs Harbor

Forgejo's container registry is sufficient for a homelab — basic OCI storage and access control. As soon as you want any of: image scanning (CVE detection), retention policies, image signing (cosign / sigstore), or replication, [Harbor](https://goharbor.io/) is the answer:

```bash
helm repo add harbor https://helm.goharbor.io
helm install harbor harbor/harbor \
  --namespace harbor --create-namespace \
  --set expose.type=clusterIP \
  --set externalURL=https://harbor.yourdomain.com \
  --set persistence.persistentVolumeClaim.registry.storageClass=nfs-storage \
  --set persistence.persistentVolumeClaim.registry.size=50Gi \
  --set trivy.enabled=true

# Then add a Traefik IngressRoute for harbor.yourdomain.com.
```

!!! tip
    For a 4-node CM4 cluster, Harbor is overkill. The recommendation: ship with Forgejo's registry. If your cluster grows or you start hosting images for other people, migrate to Harbor. "I deployed Harbor with Trivy scanning" is also a stronger resume line than "I used the built-in Forgejo registry."

## Verification

- [ ] `https://ci.yourdomain.com` loads. "Login with Forgejo" redirects to `git.yourdomain.com` and back successfully.
- [ ] Push a test commit to a Forgejo repo containing a `.woodpecker.yml`. Pipeline runs:

    ```text
    # In Woodpecker UI, the build step shows green
    # In Forgejo registry under Packages, a new image tag appears
    ```

- [ ] Manifest-bump step pushes a commit to `homelab-manifests`:

    ```bash
    git -C ~/homelab/homelab-manifests log --oneline -1
    # Expected: 'ci(myapp): bump to <sha>'
    ```

- [ ] ArgoCD reconciles the new image. In ArgoCD UI, the Application status briefly shows `OutOfSync` then returns to `Synced`. The pod is now running the new tag.
