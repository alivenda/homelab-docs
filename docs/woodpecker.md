# Woodpecker CI/CD

!!! success "Status — Live"
    Live in the cluster — the CI engine gating the homelab repos (today: manifest
    validation on `homelab-manifests`). This runbook teaches the full pipeline through
    image builds and GitOps deploys.

End-to-end pipeline: push code, auto-build container images, deploy to k3s via GitOps.

| | |
|---|---|
| **URL** | `https://ci.yourdomain.com` |
| **Namespace** | `woodpecker` |
| **Chart** | `woodpecker/woodpecker` (umbrella: `server` + `agent` subcharts) |
| **Storage** | SQLite on `local-path` (server, 2Gi) + `woodpecker-ci-scratch` (agent, per-pipeline) |
| **Auth** | Forgejo OAuth2 (`WOODPECKER_ADMIN` gate) |
| **Runs On** | k3s cluster (32 GB node) |
| **Depends On** | Kubernetes, Traefik, Backups, Forgejo |
| **Difficulty** | Intermediate–Advanced |
| **Time Estimate** | 2–3 hours |

## Step 1: OAuth2 App in Forgejo

**Site Admin → Applications → OAuth2 Applications.**

- **Redirect URI:** `https://ci.yourdomain.com/authorize`
- **Application Name:** `Woodpecker`

Save the **Client ID** and **Client Secret** to Vaultwarden.

!!! warning "Webhook delivery from Forgejo to Woodpecker (same-cluster)"
    Forgejo refuses webhooks to private/loopback IPs by default, and
    `ci.yourdomain.com` resolves to the cluster gateway (a `10.x` address). Setting
    `ALLOW_LOCALNETWORKS = true` **alone is not enough** — Forgejo still denies the
    delivery with *"webhook can only call allowed HTTP servers"*. Add your domain to
    `[webhook] ALLOWED_HOST_LIST` (e.g. `*.yourdomain.com`). And the Forgejo chart
    updates `app.ini` but does **not** roll the pod on a config change — run
    `kubectl -n forgejo rollout restart deployment forgejo` for it to take effect.

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
  --from-literal=WOODPECKER_FORGEJO_CLIENT="<CLIENT_ID_FROM_STEP_1>" \
  --from-literal=WOODPECKER_FORGEJO_SECRET="<CLIENT_SECRET_FROM_STEP_1>" \
  --from-literal=WOODPECKER_AGENT_SECRET="$WOODPECKER_AGENT_SECRET" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > woodpecker-secrets-sealed.yaml

# Commit to homelab-manifests/infrastructure/woodpecker/manifests/
```

!!! warning "Verify the seal captured real values"
    A classic bring-up bug: running `kubeseal` with the `<CLIENT_ID…>` placeholders
    left in literally — login then fails with *"Client ID not registered"*. After
    sealing, decode and eyeball the result:
    ```bash
    kubectl -n woodpecker get secret woodpecker-secrets \
      -o jsonpath='{.data.WOODPECKER_FORGEJO_CLIENT}' | base64 -d
    ```

## Step 4: Deploy Woodpecker

### Helm values

The chart is an **umbrella** wrapping two subcharts, `server` and `agent`. Every
value **must** nest under one of them — a top-level key (e.g. `nodeSelector:`) is
**silently ignored**. `values.yaml`:

```yaml
server:
  env:
    WOODPECKER_HOST: https://ci.yourdomain.com
    WOODPECKER_OPEN: "false"
    WOODPECKER_ADMIN: yourforgejousername
    # Forge config lives on the SERVER, never the agent (see warning below).
    WOODPECKER_FORGEJO: "true"
    WOODPECKER_FORGEJO_URL: https://git.yourdomain.com
  # envFrom the sealed secret; its keys match the env names exactly
  # (WOODPECKER_FORGEJO_CLIENT / _SECRET / _AGENT_SECRET) so no per-key mapping.
  extraSecretNamesForEnvFrom:
    - woodpecker-secrets
  # Use OUR fixed agent secret, not a chart-minted random one: the random value
  # regenerates on every ArgoCD resync and breaks the server<->agent handshake.
  createAgentSecret: false
  # SQLite on local-path, NOT nfs-storage — SQLite needs POSIX byte-range locking
  # that NFS doesn't do reliably (corrupts the DB), the same constraint as Forgejo.
  # The DB is small + reconstructible; pin it to the heavy node and let
  # Velero back it up.
  persistentVolume:
    enabled: true
    storageClass: local-path
    size: 2Gi
  nodeSelector:
    workload: heavy            # emerald — keep build spikes off the control plane

agent:
  replicaCount: 1
  mapAgentSecret: false        # pair to the server's createAgentSecret: false
  extraSecretNamesForEnvFrom:
    - woodpecker-secrets
  env:
    WOODPECKER_BACKEND: kubernetes
    WOODPECKER_BACKEND_K8S_NAMESPACE: woodpecker
    WOODPECKER_BACKEND_K8S_POD_NODE_SELECTOR: '{"workload":"heavy"}'   # JSON string
    # Per-pipeline scratch workspace on a dedicated non-archiving class (defined in
    # the GitOps install below) so disposable CI volumes don't accumulate.
    WOODPECKER_BACKEND_K8S_STORAGE_CLASS: woodpecker-ci-scratch
    WOODPECKER_BACKEND_K8S_STORAGE_RWX: "true"
    WOODPECKER_BACKEND_K8S_VOLUME_SIZE: "1G"
  nodeSelector:
    workload: heavy
```

The kubernetes backend's step pods need RBAC in the namespace; the chart creates
the Role/RoleBinding for you via `agent.serviceAccount.rbac.create` (default `true`).

!!! warning "Forge env vars: server only — and FORGEJO vs GITEA"
    Put `WOODPECKER_FORGEJO*` on the **server**, never the agent — agent-side forge
    config is a known footgun ([woodpecker-ci/helm#272](https://github.com/woodpecker-ci/helm/issues/272),
    "forge not configured"). Also: Woodpecker ≥ 2.x ships a dedicated Forgejo
    provider (`WOODPECKER_FORGEJO_*`); older versions only had `WOODPECKER_GITEA_*`,
    which still works against Forgejo (same API). On an older chart, swap
    `FORGEJO` → `GITEA` in the names.

### Manual bootstrap (one-time)

```bash
helm repo add woodpecker https://woodpecker-ci.org/
helm repo update
helm upgrade --install woodpecker woodpecker/woodpecker \
  --version <X.Y.Z> \
  --namespace woodpecker --create-namespace \
  --values values.yaml
```

Pin `--version` to a current release listed on [woodpecker-ci/helm](https://github.com/woodpecker-ci/helm).

### GitOps-managed install (recommended)

Commit two ArgoCD `Application`s, exactly as Forgejo does (its Step 3): the chart
Application via the multi-source `$values` pattern, plus a second Application for
the raw manifests (HTTPRoute, the sealed secret, the scratch StorageClass).

```yaml
# bootstrap/woodpecker.yaml (chart Application — abridged)
metadata:
  annotations:
    # server AND agent are StatefulSets with volumeClaimTemplates; the apiserver
    # defaults them -> a phantom permanent-OutOfSync diff under the normal differ.
    argocd.argoproj.io/compare-options: ServerSideDiff=true
spec:
  sources:
    - repoURL: https://woodpecker-ci.org/
      chart: woodpecker
      targetRevision: 3.6.4           # pin a version; don't track latest
      helm:
        releaseName: woodpecker       # MUST be `woodpecker`: the agent's default
        valueFiles:                   # server address and the HTTPRoute backend
          - $values/infrastructure/woodpecker/values.yaml   # both derive woodpecker-server
    - repoURL: https://github.com/<you>/homelab-manifests.git
      targetRevision: HEAD
      ref: values
  syncPolicy:
    automated: { prune: false, selfHeal: true }
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

The second `woodpecker-manifests` Application points at
`infrastructure/woodpecker/manifests/` — give it `CreateNamespace=true` but **not**
`ServerSideApply` (the same Gateway-API HTTPRoute trap as Forgejo). One of those
manifests is the scratch StorageClass the agent points at. The default
`nfs-storage` class archives every deleted PVC (a safety net for real app data) —
wrong for disposable per-pipeline scratch, which would otherwise pile up unbounded.
A dedicated class with `archiveOnDelete: "false"` (same provisioner) makes it
genuinely disposable:

```yaml
# manifests/ci-scratch-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: woodpecker-ci-scratch
provisioner: cluster.local/nfs-provisioner-nfs-subdir-external-provisioner  # match nfs-storage
parameters:
  archiveOnDelete: "false"
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

!!! note "Applications register on merge"
    The app-of-apps `root.yaml` is live, so committing `bootstrap/woodpecker.yaml`
    and merging is enough — root creates the Applications on its next sync. No
    manual `kubectl apply`.

## Step 5: HTTPRoute

Standard HTTPRoute for `ci.yourdomain.com`. Same shape as [Vaultwarden Step 3](vaultwarden.md#step-3-httproute) — change the backend service to `woodpecker-server`, port `80` (the server subchart's Service port; the `woodpecker-server` name comes from the mandatory `woodpecker` release name).

## Step 6: Why kubernetes backend (not docker)

- Each pipeline step becomes its own pod, scheduled by k3s. Resource limits and node selectors work.
- **No `/var/run/docker.sock` mount in the agent.** The original compose mount was a privileged escape risk. The kubernetes backend doesn't need it — image builds happen in dedicated build pods.
- Cross-architecture builds: pin steps to amd64 or arm64 via `nodeSelector` on the build pod if you ever add x86 nodes.

!!! warning "Pipeline images must be ARM64"
    All pipeline images must be ARM64-compatible. Most official images publish arm64 builds — verify third-party plugins. See [Reality of ARM64 Homelabs](prerequisites.md#reality-of-arm64-homelabs) for debugging strategy.

## Pre-merge validation gate (kubeconform)

The first pipeline to port is usually the manifest-validation gate — the same
kubeconform check the repo ran on GitHub Actions, now on Woodpecker so it gates
AGit PRs (and replaces the GitHub workflow as the single source of CI). Drop
`.woodpecker.yml` at the repo root:

```yaml
when:
  - event: pull_request
    branch: main
  - event: push
    branch: main

steps:
  kubeconform:
    image: ghcr.io/yannh/kubeconform:v0.7.0-alpine
    commands:
      - |
        find apps infrastructure bootstrap -name '*.yaml' \
          ! -name 'values.yaml' \
          ! -name 'kustomization.yaml' \
          -print0 | xargs -0 -r kubeconform \
            -strict -summary -schema-location default \
            -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
```

Two adaptations vs the GitHub Actions version: the **`-alpine`** image tag (the
plain `kubeconform` image is `scratch` — no shell for Woodpecker's `commands`), and
`! -name` instead of GNU `find`'s `-not -name` (alpine ships busybox `find`). The
check validates the **whole tree** every run, so one broken manifest on `main` reds
every subsequent PR until it's fixed — keep `main` green.

!!! note "AGit PRs do trigger Woodpecker"
    With the Forgejo webhook delivering (Step 1), an AGit pull request
    (`git push origin HEAD:refs/for/main -o topic=…`) fires a `pull_request` event
    and the gate runs **before merge** — no Forgejo Actions runner needed.

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

# Then add an HTTPRoute for harbor.yourdomain.com.
```

Pin `--version` to a current release listed on [goharbor/harbor-helm](https://github.com/goharbor/harbor-helm).

!!! tip
    For a 4-node CM4 cluster, Harbor is overkill. The recommendation: ship with Forgejo's registry. If your cluster grows or you start hosting images for other people, migrate to Harbor. "I deployed Harbor with Trivy scanning" is also a stronger resume line than "I used the built-in Forgejo registry."

## Verification

- [ ] `https://ci.yourdomain.com` loads. "Login with Forgejo" redirects to `git.yourdomain.com` and back successfully.
- [ ] Open an AGit PR touching a manifest — the `kubeconform` gate runs and shows green/red in the Woodpecker UI and on the PR (proves the webhook + `pull_request` event work end-to-end).
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
