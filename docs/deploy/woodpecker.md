# Woodpecker

!!! success "Status — Live"
    Live in the cluster — the CI engine gating the homelab repos (today: manifest
    validation on `homelab-manifests`). This runbook teaches the full pipeline through
    image builds and GitOps deploys.

End-to-end pipeline: push code, auto-build container images, deploy to k3s through GitOps.

| | |
|---|---|
| **URL** | https://ci.yourdomain.com |
| **Namespace** | `woodpecker` |
| **Chart** | `woodpecker/woodpecker` (umbrella: `server` + `agent` subcharts) |
| **Storage** | SQLite on `local-path` (server, 2Gi) + `woodpecker-ci-scratch` (agent, per-pipeline) |
| **Auth** | Forgejo OAuth 2.0 (`WOODPECKER_ADMIN` gate) |
| **Runs on** | k3s cluster (32 GB node) |
| **Depends on** | Kubernetes, Traefik, Backups, Forgejo |
| **Difficulty** | Intermediate–Advanced |
| **Time estimate** | 2–3 hours |

## OAuth 2.0 app in Forgejo { #step-1-oauth2-app-in-forgejo }

**Site Administrator → Applications → OAuth 2.0 Applications.**

- **Redirect URI:** `https://ci.yourdomain.com/authorize`
- **App Name:** `Woodpecker`

Save the **Client ID** and **Client Secret** to Vaultwarden.

!!! warning "Webhook delivery from Forgejo to Woodpecker (same-cluster)"
    Forgejo refuses webhooks to private/loopback IPs by default, and
    `ci.yourdomain.com` resolves to the cluster gateway (a `10.x` address). Setting
    `ALLOW_LOCALNETWORKS = true` **alone is not enough** — Forgejo still denies the
    delivery with *"webhook can only call allowed HTTP servers."* Add your domain to
    `[webhook] ALLOWED_HOST_LIST` (for example `*.yourdomain.com`). And the Forgejo chart
    updates `app.ini` but does **not** roll the pod on a config change — run
    `kubectl -n forgejo rollout restart deployment forgejo` for it to take effect.

## Generate the agent secret { #step-2-generate-the-agent-secret }

```bash
WOODPECKER_AGENT_SECRET=$(openssl rand -hex 32)
echo "Woodpecker agent secret: $WOODPECKER_AGENT_SECRET"   # save to Vaultwarden
```

## Seal Woodpecker credentials { #step-3-seal-woodpecker-credentials }

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

## Deploy Woodpecker { #step-4-deploy-woodpecker }

### Helm values

The chart is an **umbrella** wrapping two subcharts, `server` and `agent`. Every
value **must** nest under one of them — a top-level key (for example `nodeSelector:`) is
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
    workload: heavy            # emerald — the node that holds the local-path DB

agent:
  replicaCount: 1
  mapAgentSecret: false        # pair to the server's createAgentSecret: false
  extraSecretNamesForEnvFrom:
    - woodpecker-secrets
  env:
    WOODPECKER_BACKEND: kubernetes
    WOODPECKER_BACKEND_K8S_NAMESPACE: woodpecker
    # Step pods pull their own images, so run them on a large-disk worker.
    # The control plane is large-disk too, so exclude it.
    WOODPECKER_BACKEND_K8S_POD_AFFINITY: |
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: storage
                  operator: In
                  values: ["large"]
                - key: node-role.kubernetes.io/control-plane
                  operator: DoesNotExist
    # Per-pipeline scratch workspace on a dedicated non-archiving class (defined in
    # the GitOps install below) so disposable CI volumes don't accumulate.
    WOODPECKER_BACKEND_K8S_STORAGE_CLASS: woodpecker-ci-scratch
    WOODPECKER_BACKEND_K8S_STORAGE_RWX: "true"
    WOODPECKER_BACKEND_K8S_VOLUME_SIZE: "1G"
  nodeSelector:
    workload: heavy
```

The agent needs RBAC in the namespace to create step pods; the chart creates the
Role/RoleBinding for its `woodpecker-agent` service account with
`agent.serviceAccount.rbac.create` (default `true`). Step pods need no API access. They
run as the namespace's `default` service account, which
`manifests/default-serviceaccount.yaml` sets to `automountServiceAccountToken: false`, so
pipeline code gets no Kubernetes token.

Step pods run on topaz, not on emerald with the server and agent. Each step pulls
its own image. On emerald's 16 GB eMMC, image GC deleted `python:3.14` after each
run, so the next run downloaded about 1.45 GiB again. topaz has a 32 GB eMMC and
already serves the NFS workspace.

!!! warning "A misspelled affinity key fails without an error"
    The agent parses `WOODPECKER_BACKEND_K8S_POD_AFFINITY` with `sigs.k8s.io/yaml`,
    which ignores unknown fields. A misspelled key leaves an empty affinity, so step
    pods can land on any untainted node, and the agent logs nothing. Steps can't
    override this affinity, because `WOODPECKER_BACKEND_K8S_POD_AFFINITY_ALLOW_FROM_STEP`
    defaults to `false`. They can add tolerations, because
    `WOODPECKER_BACKEND_K8S_POD_TOLERATIONS_ALLOW_FROM_STEP` defaults to `true`. That's
    why the affinity excludes the control plane even though the control plane is tainted.

!!! warning "Forge env vars: server only — and Forgejo vs GITEA"
    Put `WOODPECKER_FORGEJO*` on the **server**, never the agent — agent-side forge
    config is a known footgun ([`woodpecker-ci/helm#272`](https://github.com/woodpecker-ci/helm/issues/272),
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

Pin `--version` to a current release listed on [`woodpecker-ci/helm`](https://github.com/woodpecker-ci/helm).

### GitOps-managed install (recommended)

Commit two ArgoCD `Application`s, exactly as [Forgejo's GitOps-managed install](forgejo.md#step-3-gitops-managed-install-recommended): the chart
App with the multi-source `$values` pattern, plus a second App for
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

The second `woodpecker-manifests` App points at
`infrastructure/woodpecker/manifests/` — give it `CreateNamespace=true` but **not**
`ServerSideApply` (the same Gateway-API HTTPRoute trap as Forgejo). One of those
manifests is the scratch StorageClass the agent points at. The default
`nfs-storage` class archives every deleted PVC (a safety net for real app data) —
wrong for disposable per-pipeline scratch, which otherwise piles up unbounded.
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

## HTTPRoute { #step-5-httproute }

Standard HTTPRoute for `ci.yourdomain.com`. Same shape as [Vaultwarden HTTPRoute](vaultwarden.md#step-3-httproute) — change the backend service to `woodpecker-server`, port `80` (the server subchart's Service port; the `woodpecker-server` name comes from the mandatory `woodpecker` release name).

## Why Kubernetes backend (not Docker) { #step-6-why-kubernetes-backend-not-docker }

- Each pipeline step becomes its own pod, scheduled by k3s. Resource limits and node selectors work.
- **No `/var/run/docker.sock` mount in the agent.** The original compose mount was a privileged escape risk. The Kubernetes backend doesn't need it — image builds happen in dedicated build pods.
- Cross-architecture builds: pin steps to amd64 or arm64 with `nodeSelector` on the build pod if you ever add x86 nodes.

!!! warning "Pipeline images must be ARM64"
    All pipeline images must be ARM64-compatible. Most official images publish arm64 builds — verify third-party plugins. See [Reality of ARM64 Homelabs](../get-started/prerequisites.md#reality-of-arm64-homelabs) for debugging strategy.

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
          -print0 | xargs -0 -r /kubeconform \
            -strict -summary -schema-location default \
            -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
```

Three adaptations vs the GitHub Actions version: the **`-alpine`** image tag (the
plain `kubeconform` image is `scratch` — no shell for Woodpecker's `commands`), the
full `/kubeconform` path (the binary is the image's entrypoint and isn't on `PATH`), and
`! -name` instead of GNU `find`'s `-not -name` (alpine ships busybox `find`). The
check validates the **whole tree** every run, so one broken manifest on `main` reds
every subsequent PR until it's fixed — keep `main` green.

!!! note "AGit PRs do trigger Woodpecker"
    With the Forgejo webhook delivering ([OAuth 2.0 app in Forgejo](#step-1-oauth2-app-in-forgejo)), an AGit pull request
    (`git push origin HEAD:refs/for/main -o topic=…`) fires a `pull_request` event
    and the gate runs **before merge** — no Forgejo Actions runner needed.

## Required checks and branch protection

Every homelab repo runs a Woodpecker pipeline on each pull request and each push to
`main`, and Forgejo branch protection makes that pipeline's result required:

| Repo | Checks |
|---|---|
| `homelab-manifests` | gitleaks, kubeconform |
| `homelab-ansible` | gitleaks, pre-commit (`ansible-lint`, YAML, private keys) |
| `homelab-terraform` | gitleaks, `tofu fmt -check`, `tofu validate` |
| `homelab-docs` | gitleaks, `mkdocs build --strict`, Vale |
| `homelab-secrets` | gitleaks, pre-commit (SOPS encryption checks, private keys) |

A failed pipeline sends an ntfy push through its `notify-failure` step, which reads
`ntfy_token`, a Woodpecker secret on the org.

### The branch protection rule

| | |
|---|---|
| **Branch** | `main` |
| **Direct pushes** | Blocked, for admins too |
| **Required status** | `ci/woodpecker/pr/woodpecker` |
| **Approvals** | 0 — Forgejo doesn't let you approve your own PR |
| **Outdated branches** | Allowed — CI tests the PR's head, not the merged result |

AGit pushes still work: they go to `refs/for/main`, which Forgejo handles as a pull
request, not as a push to `main`. To give every repo the same rule, apply it through the
API with a Forgejo token that has `write:repository`:

```fish
read -s -P 'Forgejo token: ' tok
set rule '{"rule_name":"main","enable_push":false,"enable_status_check":true,"status_check_contexts":["ci/woodpecker/pr/woodpecker"],"apply_to_admins":true,"required_approvals":0}'
curl -fsS -X POST -H "Authorization: token $tok" -H 'Content-Type: application/json' -d $rule https://git.yourdomain.com/api/v1/repos/<org>/<repo>/branch_protections
```

!!! warning "There's no direct-push escape hatch"
    With the rule applied to admins, a hotfix to `main` also needs a PR with a passing
    pipeline. If CI itself is broken, delete the rule, merge the fix, and re-create the
    rule.

### Secret scanning (gitleaks)

The pre-commit gitleaks hook scans only staged changes, so it guards local commits but
checks nothing in a CI clone. Each pipeline therefore runs gitleaks as its first step,
over the full history:

```yaml
clone:
  git:
    image: docker.io/woodpeckerci/plugin-git:2.10.1@sha256:<digest>
    settings:
      partial: false
      depth: 0

steps:
  gitleaks:
    image: ghcr.io/gitleaks/gitleaks:v8.30.1@sha256:<digest>
    commands:
      - test "$(git rev-parse --is-shallow-repository)" = false
      - test -z "$(git config --get remote.origin.partialclonefilter)"
      - gitleaks git --redact --no-banner --verbose .
```

- **The clone override.** Woodpecker's default clone is shallow and treeless
  (`--depth=1 --filter=tree:0`), which gives gitleaks one commit to scan. Any
  `woodpeckerci/plugin-git` image stays on Woodpecker's trusted-clone list, so the
  override keeps the clone credentials.
- **The two guard commands.** gitleaks exits 0 when git fails or the history is
  missing, so without the clone block the step passes without scanning anything. The
  guards fail the step on a shallow or partial clone, and outside a readable repo.
- **`--redact`** keeps finding values out of the pipeline log.
- **False positives** go in `.gitleaksignore`. SealedSecret ciphertext is the usual
  one. A finding that only one old commit holds gets the commit-scoped form,
  `<commit>:<file>:<rule-id>:<line>`, so the entry can't hide a later secret on the same
  line.

## Sample pipeline (build → push → bump manifest)

The Kubernetes backend doesn't have a Docker socket. Use [BuildKit](https://github.com/moby/buildkit) (rootless) to build OCI images directly inside a build pod — no daemon, no socket mount, no privileged container.

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

!!! warning "Branch protection rejects the bump push"
    The `update-manifest` step pushes straight to `main`, which
    [branch protection](#the-branch-protection-rule) rejects. Push the bump as a PR
    instead — for example, `git push origin HEAD:refs/for/main -o topic=bump-myapp` — so
    it goes through the same checks.

!!! note "Registry auth for BuildKit"
    For pushes to a private Forgejo registry, mount a Docker-config-format Secret at `/home/user/.docker/config.json` in the build pod (Woodpecker `volumes:` or Kubernetes backend pod-template overrides). The same `config.json` pattern works for any OCI registry.

!!! tip "Forgejo's built-in container registry"
    Enable Forgejo's container registry under `[packages]` in `app.ini` so pipelines push/pull images without Docker Hub.

## Renovate: keep dependencies and image tags current

Your CI pipeline preceding builds your own images. Third-party versions (helm charts, Terraform providers, GitHub Actions, pip and Ansible deps, image tags) need a different update strategy. [Renovate](https://docs.renovatebot.com/) watches your repos for outdated versions and opens PRs to bump them.

### How it runs

This build runs Renovate as a Kubernetes CronJob in `homelab-manifests`
(`infrastructure/renovate/`). Mend's hosted Renovate app is GitHub-only, so a Forgejo
primary needs a self-hosted runner. A CronJob keeps the schedule in Git, where Argo CD
reconciles it like everything else.

| | |
|---|---|
| **Schedule** | 02:00 UTC daily |
| **Platform** | `forgejo`, through the in-cluster Forgejo API, so a run never depends on ingress or DNS |
| **Repos** | All five, as an explicit list in `RENOVATE_REPOSITORIES` — no autodiscover |
| **Onboarding** | Off; each repo already has a `renovate.json` |
| **Token** | The owner's Forgejo token (`read:user`, `write:repository`, `write:issue`, `read:organization`), sealed into the `renovate` namespace |
| **Commit author** | `Renovate Bot <bot@renovateapp.com>` (`RENOVATE_GIT_AUTHOR`) |
| **Automerge** | None — every update waits for review |
| **Dependency Dashboard** | One issue per repo |

**Why a separate commit author:** Renovate otherwise commits as the token's owner. With
the owner's email on its commits, Renovate can't tell a human edit on one of its
branches from its own commit, so a rebase can overwrite that edit.

**Why the owner's token:** a separate bot account shrinks what a stolen token can
reach. For a single-user Forgejo that's reachable only from the LAN and the tailnet,
that's an accepted trade-off.

??? note "Alternative: a scheduled Woodpecker pipeline"

    Renovate can also run as a Woodpecker cron pipeline in each repo:

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

    In Woodpecker, schedule the cron `renovate` to run nightly. With
    `RENOVATE_AUTODISCOVER=true`, Renovate walks every repo the token has access to —
    scope the token to the repos you actually want scanned. The CronJob won out because
    its schedule lives in Git instead of Woodpecker's cron table.

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

`config:recommended` turns on the managers with a default file pattern: Helm values,
Terraform, Woodpecker pipelines, pip requirements, `ansible-galaxy`, Dockerfiles, and
more. The `kubernetes` manager has no default file pattern, so raw manifests need the
explicit configuration [below](#raw-manifest-image-tags). The `dependencies` label must
exist in each repo; Renovate drops a missing label without an error.

### Per-repo grouping rules

Each repo adds a single grouping rule for its primary manager so related bumps land in one PR:

| Repo | Extra `packageRules` entry |
|---|---|
| `homelab-ansible` | `matchManagers: ["ansible-galaxy"]` → `groupName: "ansible collections"` |
| `homelab-docs` | `matchManagers: ["pip_requirements"]` → `groupName: "mkdocs python deps"` |
| `homelab-terraform` | `matchManagers: ["terraform"]` → `groupName: "terraform providers"` |
| `homelab-secrets` | (none — pre-commit hooks and CI images only) |
| `homelab-manifests` | (none — see the custom regular expression and raw-manifest rules below) |

### Custom regular expression for pinned chart versions

`homelab-manifests` pins helm chart versions in two non-standard places — a `--version` flag in a README install snippet, and a `targetRevision:` line in an ArgoCD `bootstrap/*.yaml` App. Neither is a path the built-in managers scan. A `customManagers` regular expression keyed off a `# renovate:` annotation makes those pins trackable. This excerpt shows the chart pins; the full config also tracks the Renovate CronJob's own image and the k3s version in the upgrade Plans:

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

To use it, drop a comment directly preceding the pin:

```yaml
# renovate: datasource=helm depName=traefik registryUrl=https://traefik.github.io/charts
targetRevision: 40.2.0
```

### Raw-manifest image tags

Images pinned in raw manifests, outside any Helm values file, need the `kubernetes`
manager with explicit file patterns:

```json
"kubernetes": {
  "managerFilePatterns": [
    "/^apps\\/[^/]+\\/manifests\\/.+\\.ya?ml$/",
    "/^infrastructure\\/[^/]+\\/manifests\\/.+\\.ya?ml$/"
  ]
}
```

Three package rules go with it:

- **The CronJob's own image** stays with the custom regular expression. A rule turns the
  `kubernetes` manager off for `infrastructure/renovate/manifests/cronjob.yaml`.
- **Homepage** PRs carry a note to diff the image's `src/skeleton/` against the
  ConfigMap keys before merging. A missing skeleton file crash-loops the pod on its
  read-only config mount.
- **local-path-provisioner** PRs say its manifest is vendored from
  upstream, so a bump can also need RBAC or ConfigMap changes. **Paperless-ngx** major
  PRs carry the environment changes the new major requires.

An image with no tag resolves to whatever `latest` was when a node first pulled it, and
Renovate can't track it. Pin every image to a tag.

### Update policy

- **Nothing merges unattended,** in any repo. In `homelab-manifests` a merge is a
  deploy, and `kubeconform` validates schema, not behavior.
- **OSV vulnerability alerts** run in `homelab-manifests` only, with a `security` label.
  `config:recommended` doesn't turn them on.
- **k3s** gets one PR per minor and one for patches, each held until the release is 7
  days old. See [Upgrade k3s](../build/kubernetes.md#upgrade-k3s).

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

- [ ] Manifest-bump step opens a PR in `homelab-manifests` titled
  `ci(myapp): bump to <sha>`, and its pipeline passes.

- [ ] ArgoCD reconciles the new image. In ArgoCD UI, the App status briefly shows `OutOfSync` then returns to `Synced`. The pod is now running the new tag.
