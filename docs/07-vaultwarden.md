# Runbook 7: Vaultwarden

Lightweight self-hosted Bitwarden-compatible server. First service runbook after Traefik — smallest dependency footprint of anything in this guide, and the first **stateful** app, so it's where we work the [GitOps deploy pattern](apps-deploy-pattern.md) end to end and meet the SQLite storage rules for real. Once it's up you have a self-hosted password manager to hold every later runbook's credentials instead of pasting them into a cloud vault.

| | |
|---|---|
| **Difficulty** | Beginner (the GitOps pattern, walked slowly) |
| **Time Estimate** | 45 minutes |
| **Runs On** | `amethyst` (pinned via `nodeSelector` — see Step 2) |
| **Depends On** | Runbook 6 (HTTPS), plus the cluster baseline this guide stands up before any app: ArgoCD, the Sealed Secrets controller, and the standalone `local-path` provisioner |

This runbook follows the [GitOps deploy pattern](apps-deploy-pattern.md) — nothing here is applied imperatively; every object is committed to `homelab-manifests` and ArgoCD reconciles it. What's *specific* to Vaultwarden, and worth slowing down for, is the **SQLite storage model**. Get that wrong and the vault silently runs on ephemeral disk — wiped on every restart.

## Step 1: Seal the admin token

The `/admin` panel is gated by a single token. It's a secret, so it goes in a `SealedSecret` (safe to commit) rather than inline in `values.yaml`:

```bash
VW_ADMIN_TOKEN=$(openssl rand -base64 48)
echo "Vaultwarden admin token: $VW_ADMIN_TOKEN"   # save to your bootstrap password manager

kubectl create secret generic vaultwarden-admin \
  --namespace vaultwarden \
  --from-literal=admin-token="$VW_ADMIN_TOKEN" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > apps/vaultwarden/manifests/sealed-admin-token.yaml
```

`--dry-run=client` means the plaintext Secret is only ever built in memory and piped to `kubeseal` — it never touches the cluster or the disk. The output is a `SealedSecret` that decrypts, in-cluster, to a Secret named `vaultwarden-admin` (key `admin-token`); `values.yaml` references it by name.

!!! warning "Save the token now, off-cluster"
    The token is the only way into `/admin`, and `/admin` is the only way to manage settings without DB access. Save the plaintext to the bootstrap password manager you're migrating *away from* — not this new Vaultwarden, which would be circular. The Sealed Secrets signing key is itself backed up; see [Backups & DR](10-backups.md).

## Step 2: Values — storage is the part everyone gets wrong

`apps/vaultwarden/values.yaml` holds the chart config. The manifest in `homelab-manifests` is the source of truth; the storage block is the part to understand rather than copy blindly:

```yaml
domain: https://vault.yourdomain.com

adminToken:
  existingSecret: vaultwarden-admin
  existingSecretKey: admin-token

signupsAllowed: false

# Pin the workload type. SQLite (database.type stays "default") already makes the
# chart render a StatefulSet; stating it stops a future edit from silently flipping
# it to a Deployment.
resourceType: StatefulSet

# Pin to the light worker. Keeps the vault off the control plane; local-path's
# WaitForFirstConsumer then provisions the PV here and the pod always reschedules
# back to its data.
nodeSelector:
  kubernetes.io/hostname: amethyst

# Persistence: the guerzon chart's real schema is storage.data.
storage:
  data:
    name: vaultwarden-data
    size: 2Gi
    class: local-path
    accessMode: ReadWriteOnce

ingress:
  enabled: false        # routing is a separate HTTPRoute — see Step 3

service:
  type: ClusterIP
```

Three things make or break this app:

!!! danger "The schema is `storage.data`, **not** a top-level `persistence:` block"
    The guerzon chart has no `persistence:` key. If you write one (as older guides do), Helm **silently drops it** and `/data` lands on the ephemeral container layer — a vault wiped on every pod restart, with no error to warn you. Persistence lives under `storage.data` (`name`/`size`/`class`/`accessMode`).

!!! danger "SQLite goes on `local-path`, never `nfs-storage`"
    Vaultwarden uses SQLite. NFS does not do POSIX file locking reliably and **will corrupt a SQLite DB**. Bind the node-local `local-path` class (see the [storage architecture](storage-architecture.md#the-local-path-tier)). That class is `Retain`, and the StatefulSet renders `persistentVolumeClaimRetentionPolicy: Retain`, so the data survives a PVC/StatefulSet delete.

!!! note "Don't add `strategy: Recreate` here"
    The general SQLite advice in the [deploy pattern](apps-deploy-pattern.md) pairs `local-path` with `strategy: Recreate` — that's for *Deployment*-based apps. Vaultwarden renders a single-replica **StatefulSet**, which already rolls serially (the old pod terminates before its replacement starts), so it can't double-mount the `ReadWriteOnce` volume. On a StatefulSet, `Recreate` maps to `updateStrategy`, where it's an invalid value.

## Step 3: Routing — an HTTPRoute, in the manifests app { #step-3-httproute }

Vaultwarden configures no TLS of its own; the shared Traefik Gateway terminates it with the wildcard cert. Commit the route to `apps/vaultwarden/manifests/` (Service port `80` → the chart's `vaultwarden` Service):

```yaml
# apps/vaultwarden/manifests/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: vaultwarden
  namespace: vaultwarden
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - vault.yourdomain.com
  rules:
    - backendRefs:
        - name: vaultwarden
          port: 80
```

!!! warning "The manifests app must **not** use Server-Side Apply"
    The HTTPRoute is applied by the `vaultwarden-manifests` Application with client-side apply. `ServerSideApply=true` on an app that manages a Gateway API `HTTPRoute` leaves it permanently `OutOfSync` (Healthy but never reconciled). The Helm chart app keeps SSA; the manifests app does not.

## Step 4: Register the Applications and sync

`bootstrap/vaultwarden.yaml` holds **two** ArgoCD Applications:

- **`vaultwarden`** — the guerzon Helm chart via the multi-source `$values` pattern (`valueFiles: [$values/apps/vaultwarden/values.yaml]`), `ServerSideApply=true`, and `prune: false` so a sync can't drop the StatefulSet/PVC mid-flight.
- **`vaultwarden-manifests`** — the raw `manifests/` (HTTPRoute + the admin-token SealedSecret), **no SSA**. Both set `CreateNamespace=true` so the `vaultwarden` namespace exists for whichever syncs first.

Commit everything on a branch, open a PR, merge, and let ArgoCD reconcile. The StatefulSet comes up on `amethyst`, the PVC binds on `local-path`.

!!! tip "A brief CrashLoop is expected"
    If the chart app syncs before the Sealed Secrets controller has decrypted `vaultwarden-admin`, the pod CrashLoops until the Secret exists, then `selfHeal` recovers it. If it's *still* crashing after `kubectl -n vaultwarden get secret vaultwarden-admin` shows the Secret, check the logs.

## Step 5: Create your first account (signups toggle — the GitOps way)

`signupsAllowed: false` disables the public `/register` form. To create your first account, flip it on **through Git** (not `helm --set`, which ArgoCD's `selfHeal` would immediately revert), register, then flip it back:

1. PR: set `signupsAllowed: true` → merge.
2. Nudge ArgoCD instead of waiting for its poll, and watch the pod roll:
   ```bash
   kubectl -n argocd annotate application vaultwarden argocd.argoproj.io/refresh=hard --overwrite
   kubectl -n vaultwarden rollout status statefulset/vaultwarden
   ```
3. Register at `https://vault.yourdomain.com`.
4. PR: set `signupsAllowed: false` → merge → refresh again. Confirm `/register` no longer offers **Create Account**.

!!! warning "Keep the window short; skip the admin-invite shortcut"
    While signups are on, anyone who can reach the host can register — stage the flip-back PR before you start so you can merge it the moment you're done. The `/admin` "invite user" path looks like it would avoid the open window, but without SMTP configured it's unreliable on current Vaultwarden (invited users hit *"Registration not allowed or user already exists"*). The toggle is the deterministic path.

## Step 6: Verify the backup actually captures the vault

This is the step that matters most for a password manager, and the one the storage choice makes subtle. Velero's filesystem backup writes **zero bytes** for a `local-path` PVC unless the StorageClass emits `local`-type PVs (`defaultVolumeType: local` — see the [local-path tier](storage-architecture.md#the-local-path-tier)). It still reports `Completed`, so you **gate on bytes, not status**.

Take an on-demand backup of just this namespace and read the byte counters:

```bash
velero backup create vaultwarden-bytes-check \
  --include-namespaces vaultwarden \
  --default-volumes-to-fs-backup \
  --wait

kubectl -n velero get podvolumebackup \
  -l velero.io/backup-name=vaultwarden-bytes-check \
  -o custom-columns='VOLUME:.spec.volume,PHASE:.status.phase,TOTAL:.status.progress.totalBytes,DONE:.status.progress.bytesDone'
```

A healthy result is a `PodVolumeBackup` for the `vaultwarden-data` volume with `PHASE=Completed` and `TOTAL`/`DONE` clearly non-zero (a populated SQLite DB is hundreds of KB). **No row, or zero bytes, means the volume isn't being read** — recheck the PV's volume type before trusting the backup.

## Verification

- [ ] Vaultwarden pod Running:

    ```bash
    kubectl get pods -n vaultwarden
    ```

- [ ] ArgoCD shows both `vaultwarden` and `vaultwarden-manifests` Synced / Healthy.
- [ ] Browser at `https://vault.yourdomain.com` loads over the wildcard cert (no warning, no "cannot reach server" from extensions).
- [ ] The Bitwarden/Vaultwarden browser extension connects after entering the server URL.
- [ ] `/admin` loads with the admin token from the SealedSecret.
- [ ] Signups are disabled — `/register` shows the disabled message, no **Create Account** button.
- [ ] Velero `PodVolumeBackup` for `vaultwarden-data` shows **bytes > 0** (Step 6).
- [ ] Plaintext admin token saved to your bootstrap password manager.

!!! tip "HTTPS is mandatory"
    Vaultwarden requires HTTPS or browser extensions silently refuse to connect. The HTTPRoute above — TLS terminated by the Gateway's wildcard cert — handles this. If an extension reports "cannot reach server", verify the cert with `curl -v https://vault.yourdomain.com`.
