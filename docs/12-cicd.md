# Runbook 12: CI/CD with Forgejo and Woodpecker CI

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
  # The chart pulls additional env vars from the named Secret. Make sure
  # the SealedSecret's keys match the env names exactly:
  #   WOODPECKER_FORGEJO_CLIENT, WOODPECKER_FORGEJO_SECRET, WOODPECKER_AGENT_SECRET
  extraSecretNamesForEnvFrom:
    - woodpecker-secrets
  persistentVolume:
    enabled: true
    storageClass: nfs-storage
  service:
    type: ClusterIP

agent:
  env:
    WOODPECKER_BACKEND: kubernetes
  extraSecretNamesForEnvFrom:
    - woodpecker-secrets
  replicaCount: 2

nodeSelector:
  storage: large
```

When you seal the credentials in Step 3, name the keys to match the Woodpecker env vars (`WOODPECKER_FORGEJO_CLIENT`, `WOODPECKER_FORGEJO_SECRET`, `WOODPECKER_AGENT_SECRET`) so `envFrom` wires them directly — no per-key mapping needed.

Install:

```bash
helm repo add woodpecker https://woodpecker-ci.org/
helm repo update
helm upgrade --install woodpecker woodpecker/woodpecker \
  --version <X.Y.Z> \
  --namespace woodpecker --create-namespace \
  --values values.yaml
```

Pin `--version` to a current release listed on [woodpecker-ci/helm](https://github.com/woodpecker-ci/helm).

!!! warning "FORGEJO vs GITEA env var names"
    Modern Woodpecker (>= 2.x) ships a dedicated Forgejo provider (`WOODPECKER_FORGEJO_*`). Older versions only had `WOODPECKER_GITEA_*` which still works against Forgejo (same API surface). If your chart version is older, swap `FORGEJO` → `GITEA` in the env-var names.

## Step 5: IngressRoute

Standard IngressRoute for `ci.yourdomain.com`. Same shape as [Vaultwarden Step 3](07-vaultwarden.md#step-3-ingressroute) — change the service name to `woodpecker-server` and port to `8000`.

## Step 6: Why kubernetes backend (not docker)

- Each pipeline step becomes its own pod, scheduled by k3s. Resource limits and node selectors work.
- **No `/var/run/docker.sock` mount in the agent.** The original compose mount was a privileged escape risk. The kubernetes backend doesn't need it — image builds happen in dedicated build pods.
- Cross-architecture builds: pin steps to amd64 or arm64 via `nodeSelector` on the build pod if you ever add x86 nodes.

!!! warning "Pipeline images must be ARM64"
    All pipeline images must be ARM64-compatible. Most official images publish arm64 builds — verify third-party plugins. See [Reality of ARM64 Homelabs](00-prerequisites.md#reality-of-arm64-homelabs) for debugging strategy.

## Sample Pipeline (build → push → bump manifest)

The kubernetes backend doesn't have a Docker socket. Use [BuildKit](https://github.com/moby/buildkit) (rootless) to build OCI images directly inside a build pod — no daemon, no socket mount, no privileged container.

!!! note "Kaniko was archived in June 2025"
    Earlier versions of this runbook recommended kaniko. The kaniko project was archived upstream; BuildKit's rootless image (`moby/buildkit:rootless`) is the maintained replacement and works identically for CI-style "build and push" flows.

```yaml
# In your app repo: .woodpecker.yml
when:
  - event: push
    branch: main

steps:
  build-and-push:
    image: moby/buildkit:rootless
    environment:
      BUILDKITD_FLAGS: --oci-worker-no-process-sandbox
    commands:
      - |
        buildctl-daemonless.sh build \
          --frontend dockerfile.v0 \
          --local context=. \
          --local dockerfile=. \
          --output type=image,\"name=git.yourdomain.com/youruser/myapp:${CI_COMMIT_SHA},git.yourdomain.com/youruser/myapp:latest\",push=true

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

!!! note "Registry auth for BuildKit"
    For pushes to a private Forgejo registry, mount a Docker-config-format Secret at `/home/user/.docker/config.json` in the build pod (Woodpecker `volumes:` or k8s backend pod-template overrides). The same `config.json` pattern works for any OCI registry.

!!! tip "Forgejo's built-in container registry"
    Enable Forgejo's container registry under `[packages]` in `app.ini` so pipelines push/pull images without Docker Hub.

## Renovate: keep dependencies and image tags current

Your CI pipeline above builds your own images. Third-party versions (helm charts, terraform providers, GitHub Actions, pip and ansible deps, image tags) need a different update strategy. [Renovate](https://docs.renovatebot.com/) watches your repos for outdated versions and opens PRs to bump them.

### Hosted vs self-hosted

- **Mend Renovate hosted (recommended):** install the [Mend Renovate GitHub App](https://github.com/marketplace/renovate), grant it access to the repos you want scanned, drop a `renovate.json` in each. No infra to run.
- **Self-hosted via Woodpecker:** a scheduled cron pipeline (shown at the end of this section). Use this if you want Renovate inside your cluster.

### Base config (every repo)

Every homelab repo ships a `renovate.json` that extends `config:recommended` and enables the pre-commit-hooks manager:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", ":enablePreCommit"],
  "dependencyDashboard": true,
  "labels": ["dependencies"],
  "packageRules": [
    { "matchManagers": ["pre-commit"], "groupName": "pre-commit hooks" }
  ]
}
```

`config:recommended` auto-discovers k8s manifests, helm values, terraform, GitHub Actions, pip requirements, ansible-galaxy, Dockerfile, and more — no explicit `fileMatch` overrides needed for the standard cases.

### Per-repo grouping rules

Each repo adds a single grouping rule for its primary manager so related bumps land in one PR:

| Repo | Extra `packageRules` entry |
|---|---|
| `homelab-ansible` | `matchManagers: ["ansible-galaxy"]` → `groupName: "ansible collections"` |
| `homelab-docs` | `matchManagers: ["pip_requirements"]` → `groupName: "mkdocs python deps"` |
| `homelab-terraform` | `matchManagers: ["terraform"]` → `groupName: "terraform providers"` |
| `homelab-manifests` | (none — see custom regex below) |

### Custom regex for pinned chart versions

`homelab-manifests` pins helm chart versions in two non-standard places — a `--version` flag in a README install snippet, and a `targetRevision:` line in an ArgoCD `bootstrap/*.yaml` Application. Neither is a path the built-in managers scan. A `customManagers` regex keyed off a `# renovate:` annotation makes those pins trackable:

```json
"customManagers": [
  {
    "customType": "regex",
    "managerFilePatterns": [
      "(^|/)README\\.md$",
      "(^|/)bootstrap/.+\\.yaml$"
    ],
    "matchStrings": [
      "# renovate: datasource=(?<datasource>\\S+) depName=(?<depName>\\S+) registryUrl=(?<registryUrl>\\S+)[\\s\\S]{0,200}?--version (?<currentValue>\\S+)",
      "# renovate: datasource=(?<datasource>\\S+) depName=(?<depName>\\S+) registryUrl=(?<registryUrl>\\S+)[\\s\\S]{0,200}?targetRevision: (?<currentValue>\\S+)"
    ],
    "versioningTemplate": "semver"
  }
]
```

To use it, drop a comment directly above the pin:

```yaml
# renovate: datasource=helm depName=traefik registryUrl=https://traefik.github.io/charts
targetRevision: 40.2.0
```

### Self-hosted: schedule via Woodpecker

If you'd rather not depend on Mend, run Renovate as a scheduled Woodpecker pipeline in each repo:

```yaml
# .woodpecker/renovate.yml
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

In Woodpecker, schedule the cron `renovate` to run nightly. With `RENOVATE_AUTODISCOVER=true`, Renovate walks every repo the token has access to — scope the token to the repos you actually want scanned.

## Container registry: Forgejo built-in vs Harbor

Forgejo's container registry is sufficient for a homelab — basic OCI storage and access control. As soon as you want any of: image scanning (CVE detection), retention policies, image signing (cosign / sigstore), or replication, [Harbor](https://goharbor.io/) is the answer:

```bash
helm repo add harbor https://helm.goharbor.io
helm upgrade --install harbor harbor/harbor \
  --version <X.Y.Z> \
  --namespace harbor --create-namespace \
  --set expose.type=clusterIP \
  --set externalURL=https://harbor.yourdomain.com \
  --set persistence.persistentVolumeClaim.registry.storageClass=nfs-storage \
  --set persistence.persistentVolumeClaim.registry.size=50Gi \
  --set trivy.enabled=true

# Then add a Traefik IngressRoute for harbor.yourdomain.com.
```

Pin `--version` to a current release listed on [goharbor/harbor-helm](https://github.com/goharbor/harbor-helm).

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
