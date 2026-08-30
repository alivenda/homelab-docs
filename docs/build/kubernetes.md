# Kubernetes (k3s)

Deploy k3s across all 4 CM4 nodes, using the SATA SSD on Node 3 for NFS persistent storage and your UGREEN NAS for bulk media.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time estimate** | 1–2 hours (after Turing Pi) |
| **Runs on** | Turing Pi 2 (all 4 CM4 nodes) |
| **Depends on** | Turing Pi (or Ansible) |

!!! note "If you ran Ansible"
    Ansible's playbooks already installed NFS on topaz and k3s on all 4 nodes. **Skip the first four sections** — they are the imperative reference for what Ansible did, useful for understanding but redundant if you ran the playbook. Resume at [Install Helm](#step-5-install-helm).

## Get `kubectl` working on your machine { #step-0-get-kubectl-working-on-your-machine }

Every step from here on uses `kubectl`. Install it locally and copy the kubeconfig so you can talk to the cluster without SSHing into ruby first:

```bash
# macOS
brew install kubectl
# Arch / CachyOS
sudo pacman -S kubectl
# Ubuntu / Debian (snap)
sudo snap install kubectl --classic

# Pull the kubeconfig from ruby and rewrite the server URL.
# Log in as dietpi (root login is disabled by the Ansible playbook); k3s wrote the file
# world-readable via --write-kubeconfig-mode 644, so dietpi can read it.
mkdir -p ~/.kube
scp dietpi@10.0.20.10:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1/10.0.20.10/' ~/.kube/config
chmod 600 ~/.kube/config

kubectl get nodes
# Expected: 4 nodes Ready
```

## Install NFS server on topaz { #step-1-install-nfs-server-on-topaz }

```bash
apt install -y nfs-kernel-server
echo "/data 10.0.20.0/24(rw,sync,no_subtree_check,no_root_squash)" | tee -a /etc/exports
systemctl enable --now nfs-kernel-server
exportfs -ar
```

!!! tip
    `no_root_squash` with subnet restriction is a homelab tradeoff, not a production-grade NFS security posture. It works because every node on `10.0.20.0/24` is yours and the data is yours. Do not copy this config into a multi-tenant environment.

On ruby, emerald, and amethyst:

```bash
apt install -y nfs-common
```

## Install k3s server (ruby) { #step-2-install-k3s-server-ruby }

```bash
# Generate the cluster token ONCE and save to your password manager — use the SAME value in Step 3
K3S_TOKEN=$(openssl rand -hex 32)
echo "k3s token: $K3S_TOKEN"   # save this to your password manager NOW

curl -sfL https://get.k3s.io | sh -s - \
  --cluster-init \
  --write-kubeconfig-mode 644 \
  --disable servicelb \
  --disable traefik \
  --token "$K3S_TOKEN" \
  --node-ip 10.0.20.10 \
  --disable local-storage
```

!!! note "Disable servicelb, but keep the cloud controller"
    This disables `servicelb` (MetalLB replaces it) but **does not** pass `--disable-cloud-controller`. With servicelb disabled, that flag takes effect, and because k3s still runs the kubelet with `cloud-provider=external`, nodes get the `node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule` taint with nothing to remove it — pods don't schedule unless you deploy your own external CCM ([k3s#6554](https://github.com/k3s-io/k3s/issues/6554)). Keep the embedded cloud controller; it clears that taint and assigns node addresses.

!!! note "Why `--cluster-init`"
    Without `--cluster-init`, k3s defaults to a sqlite datastore (via kine). `--cluster-init` initializes the embedded etcd datastore instead — required for the `k3s etcd-snapshot` flow in Step 10. The CPU/memory overhead is modest on a CM4 with 8 GB RAM, and you can join additional server nodes later for HA without rebuilding.

!!! warning "Token reuse"
    The token you generate above is the join secret for the entire cluster. Step 3 uses the **same value** on every agent node. Generate once, store in your password manager, paste in both places.

## Join worker nodes { #step-3-join-worker-nodes }

Run on each of emerald, topaz, amethyst (use the same `K3S_TOKEN` value from [Install k3s server](#step-2-install-k3s-server-ruby)):

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://10.0.20.10:6443 \
  K3S_TOKEN=<paste_token_from_Step_2> sh -
```

Verify from your machine: `kubectl get nodes` (should show all 4 nodes Ready).

## Label nodes by capacity { #step-4-label-nodes-by-capacity }

ruby is the control plane; the other three are workers. Use labels to express role, storage capacity, and app-state placement for scheduling:

```bash
# Control plane
kubectl label nodes ruby node-role.kubernetes.io/control-plane=true storage=large

# Workers
kubectl label nodes emerald kubernetes.io/role=worker storage=small workload=heavy app-state=true
kubectl label nodes topaz kubernetes.io/role=worker storage=large role=nfs
kubectl label nodes amethyst kubernetes.io/role=worker storage=small
```

In this build the labels are managed declaratively — each host's `node_labels` in the `homelab-ansible` inventory is the source of truth, applied by the `labels` play — so treat the commands above as the imperative reference for what the play does.

!!! note "Three labels, three different jobs"
    `storage=large/small` records raw eMMC size (32 GB modules are ruby and topaz) and is **not** a placement signal — the two large-disk nodes are the control plane and the NFS/monitoring server, exactly where app data must *not* go. Placement uses the other two labels: `workload=heavy` (emerald-only) takes the spiky workloads the [Prerequisites](../get-started/prerequisites.md) runbook warns about — Paperless OCR and Woodpecker builds — and `app-state=true` (also emerald-only) marks the designated home for node-local `local-path` app data, so SQLite apps and their PVs land together on one known node. `node-role.kubernetes.io/control-plane=true` on ruby is only a **label**, not a taint — k3s doesn't taint its server by default. To hard-fence ruby, taint it (`kubectl taint nodes ruby node-role.kubernetes.io/control-plane=:NoSchedule`), but that also evicts the lighter app pods this build intentionally runs on ruby.

## Install Helm { #step-5-install-helm }

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh && ./get_helm.sh
```

## Install MetalLB { #step-6-install-metallb }

```bash
helm repo add metallb https://metallb.github.io/metallb
helm upgrade --install metallb metallb/metallb \
  --create-namespace --namespace metallb-system --wait
```

Then apply the address pool and L2 advertisement:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.20.200-10.0.20.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-pool-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - lab-pool
```

!!! note "Source of truth"
    These manifests live at `homelab-manifests/infrastructure/metallb/{ipaddresspool,l2advertisement}.yaml` and are applied by ArgoCD (the `metallb` Application) once GitOps is wired; the YAML above is shown inline for learning context. Edit the repo, not your local copy, so the changes survive a rebuild.

!!! warning "MetalLB pool vs DHCP"
    MetalLB's pool (10.0.20.200–10.0.20.250) **must be excluded from the Lab VLAN DHCP scope**. The [Network plan](network.md#network-plan) bounds Lab VLAN DHCP to .100–.199 for exactly this reason — if you change either side, change both. Without the bound, UDM eventually hands out an address in .200–.250 to a random device and you get intermittent IP conflicts that are a nightmare to debug. In UniFi Network: Settings → Networks → Lab → DHCP Range must stay 10.0.20.100–10.0.20.199.

## Install NFS storage provisioner { #step-7-install-nfs-storage-provisioner }

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
helm upgrade --install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=10.0.20.12 \
  --set nfs.path=/data \
  --set storageClass.name=nfs-storage \
  --set storageClass.defaultClass=true
```

## Install ArgoCD { #step-8-install-argocd }

!!! note "If you ran Ansible"
    The `argocd` play bootstraps ArgoCD with the **Helm chart** (`argo-cd`, version pinned
    in `site.yml`) — skip the manual install below and go straight to retrieving the
    initial admin secret. Once GitOps is wired ([Wire ArgoCD to homelab-manifests](../get-started/set-up-git.md#step-7-wire-argocd-to-homelab-manifests)),
    ArgoCD manages itself from `homelab-manifests` and the chart pin there takes over.

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Save the printed password to your password manager. (Once Vaultwarden is up, it becomes the home for everything cluster-generated.)

!!! note "Why `--server-side`"
    The upstream `install.yaml` is large enough that a client-side `kubectl apply` can hit the 256 KB annotation limit on the embedded CRDs. The current ArgoCD docs recommend server-side apply with `--force-conflicts` for the initial install.

## Access ArgoCD (before Traefik is up) { #step-9-access-argocd-before-traefik-is-up }

ArgoCD has no Ingress yet — Traefik is the next page. To reach the UI in the meantime, port-forward:

```bash
# In a separate terminal:
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then:

- Open <https://localhost:8080> in a browser.
- Accept the self-signed cert warning.
- Username: `admin`
- Password: the value you decoded from the secret in [Install ArgoCD](#step-8-install-argocd).

!!! tip "Port-forward is the bootstrap path"
    After Traefik is up, you add an HTTPRoute so argocd.yourdomain.com works without port-forwarding. The port-forward method is the bootstrap path that works regardless.

## Control-plane redundancy (SPOF awareness) { #step-10-control-plane-redundancy-spof-awareness }

!!! warning
    ruby is the only k3s server in this cluster. If it dies, the control plane is offline and workloads keep running but you cannot deploy, scale, or restart anything until ruby is recovered. For a real homelab this is acceptable — but you must have etcd snapshots and a documented restore path.

k3s ships with etcd-snapshot built in. Snapshot regularly:

```bash
# On ruby, take a manual snapshot now to verify it works
sudo k3s etcd-snapshot save
ls /var/lib/rancher/k3s/server/db/snapshots/

# Enable scheduled snapshots (every 12h, retain 10).
# Edit /etc/rancher/k3s/config.yaml and add:
etcd-snapshot-schedule-cron: "0 */12 * * *"
etcd-snapshot-retention: 10
etcd-snapshot-dir: /var/lib/rancher/k3s/server/db/snapshots

sudo systemctl restart k3s
```

Get the snapshots off-node too: k3s can upload them straight to the Garage S3 store on the NAS through its built-in `etcd-s3` config — [Backups](backups.md#off-node-etcd-snapshots-k3s-native) sets that up (codified in `homelab-ansible`).

## Restore procedure (when ruby dies) { #step-11-restore-procedure-when-ruby-dies }

If ruby is unrecoverable: reflash DietPi (per [Turing Pi](turing-pi.md)), then re-bootstrap with [Ansible](ansible.md) — `site.yml` reinstalls the **pinned** k3s version (`k3s_version` in `inventory.yml`, so the new server matches the surviving agents) and re-renders `/etc/rancher/k3s/config.yaml`, including the `etcd-s3` credentials the restore needs. Then replace the fresh empty etcd with your latest snapshot:

```bash
# List snapshots — the etcd-s3 config makes this show the Garage copies too
sudo k3s etcd-snapshot list          # pick the newest s3://etcd-snapshots/… row

# Stop the freshly-installed server and restore. For an S3 snapshot the restore
# path is the bare filename — the connection details come from config.yaml.
sudo systemctl stop k3s
sudo k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=<snapshot-filename>

# Wait for: "Managed etcd cluster membership has been reset,
#            restart without --cluster-reset flag now."
sudo systemctl start k3s
```

The agents (emerald, topaz, amethyst) rejoin on their own: ruby keeps its static IP and Ansible pins the same `k3s_token`, so their existing config still points at a server that answers. The restored state is as old as the snapshot (up to 12 h) — ArgoCD's self-heal replays anything that changed in Git since then, but anything applied imperatively with `kubectl` after the snapshot is gone.

Post-restore checks:

```bash
kubectl get nodes                        # all four Ready
kubectl get pods -A | grep -v Running    # settles to nothing crash-looping
# Sealed Secrets signing keys are back — they live in etcd, so the snapshot carries them
kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key
```

Finish with ArgoCD: every Application `Synced`/`Healthy`.

!!! tip "Back up the k3s token"
    Document your K3S_TOKEN in your password manager the day you stand the cluster up. Losing it on top of losing ruby turns a 2-hour rebuild into a full cluster rebuild.

## Sealed Secrets (cluster-side secret management) { #step-12-sealed-secrets-cluster-side-secret-management }

You have two flavors of secrets: those that live in `homelab-secrets` as sops + age (Terraform tfvars, Cloudflare tokens consumed by your machine) and those that need to be Kubernetes Secrets at runtime (DB passwords, OAuth client secrets, image-pull credentials). Don't commit raw `kubectl create secret` commands to Git — install Sealed Secrets so you can commit encrypted manifests to `homelab-manifests` safely.

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace sealed-secrets --create-namespace \
  --set fullnameOverride=sealed-secrets-controller

# Install the kubeseal CLI on your machine
# macOS
brew install kubeseal
# Arch / CachyOS
sudo pacman -S kubeseal
# Or download from github.com/bitnami-labs/sealed-secrets/releases
```

## Encrypt your first secret { #step-13-encrypt-your-first-secret }

Worked example: encrypt the Cloudflare API token (which Traefik uses for DNS-01) as a SealedSecret manifest. Commit the encrypted form to `homelab-manifests`; the cluster decrypts it at runtime.

```bash
# The traefik namespace needs to exist for the runtime materialization;
# create it now if it doesn't (the SealedSecret can be sealed before the
# namespace exists, but the runtime Secret can only be created after).
kubectl create namespace traefik

# Create a plaintext Secret manifest LOCALLY (do not commit)
cat > cf-token-plain.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: traefik
type: Opaque
stringData:
  token: <YOUR_CF_TOKEN>
EOF

# Seal it with the cluster's public key
kubeseal --controller-name=sealed-secrets-controller \
  --controller-namespace=sealed-secrets \
  --format yaml \
  < cf-token-plain.yaml > cf-token-sealed.yaml

rm cf-token-plain.yaml   # never commit plaintext

# cf-token-sealed.yaml is safe to commit. Move it into:
# homelab-manifests/apps/traefik/cf-token-sealed.yaml
# ArgoCD will apply it; the controller will create the matching Secret.
```

!!! tip "Two-layer secrets discipline"
    sops + age for repo-side (your machine reads them), Sealed Secrets for cluster-side (ArgoCD reads them). The encryption keys are different — sops uses your age private key on your machine; Sealed Secrets uses the controller's private key inside the cluster.

    Back up the Sealed Secrets controller key:

    ```bash
    kubectl get secret -n sealed-secrets \
      -l sealedsecrets.bitnami.com/sealed-secrets-key \
      -o yaml > sealed-secrets-master.key.yaml
    ```

    Store this file encrypted (with sops) in `homelab-secrets`. Losing it means losing the ability to decrypt anything you've sealed.

    That manual export is the **day-zero** bootstrap only — the controller mints a new key
    every 30 days, so a one-time copy goes stale. Ongoing backups are automated: a daily
    in-cluster CronJob ships **all** controller keys to Garage (see
    [Backups](backups.md#keep-the-signing-key-backup-current)).

## Upgrade k3s

Bumping `k3s_version` in `homelab-ansible/inventory.yml` **does not upgrade a running
cluster**. The install task carries `creates: /usr/local/bin/k3s`, so Ansible skips it
on any node that already has k3s. The pin governs fresh installs and rebuilds — keeping
a reflashed node on the same version as its neighbours — while the live roll is manual
and deliberate. Bump the pin and roll in the same sitting, or a later rebuild silently
lands on a different version than the running fleet.

### Order: control plane first { #order-control-plane-first }

Servers go first, one at a time, then agents. This is the **opposite** of the DietPi
OS-update order (agents first, `ruby` last), and getting it backwards is the one way to
actually break the cluster: the Kubernetes version-skew policy lets a kubelet lag the
API server by up to three minors, but a kubelet must **never lead it**. Upgrade an agent
first and it may refuse to register with the older control plane.

### Pre-flight { #pre-flight }

```bash
# 1. What's actually running, and what are you going to?
kubectl get nodes -o wide            # note the current version on every node

# 2. Audit the target release's removals against this cluster. Check the
#    upstream "Deprecated API Migration Guide" for the target minor, then:
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
#    Anything with a non-empty removed_release= must be migrated BEFORE upgrading.

# 3. Fresh etcd snapshot — the only real rollback for a control-plane upgrade
sudo k3s etcd-snapshot save --name pre-upgrade

# 4. Confirm the nightly Velero backup actually completed
velero backup get
```

### Roll the server (ruby)

The install script **regenerates the systemd unit from the arguments you pass it**. Pass
nothing and you lose `--disable traefik`, `--disable servicelb`, `--disable local-storage`
and the rest — the cluster comes back up fighting itself over the ingress. Re-supply the
*exact* argument list Ansible installed with:

```bash
# Drain first: workloads keep running while k3s restarts, but the API server is
# briefly gone, and anything mid-write would rather be elsewhere.
kubectl drain ruby --ignore-daemonsets --delete-emptydir-data

# On ruby. K3S_TOKEN comes from homelab-ansible's sops secrets (k3s_token) —
# the same value the agents joined with.
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=vX.Y.Z+k3s1 sh -s - \
  --cluster-init \
  --write-kubeconfig-mode 644 \
  --disable servicelb \
  --disable traefik \
  --disable local-storage \
  --token "$K3S_TOKEN" \
  --node-ip 10.0.20.10

kubectl uncordon ruby
kubectl get nodes            # ruby should report the new version, Ready
```

`/etc/rancher/k3s/config.yaml` (the etcd snapshot schedule and the S3 credentials) is
**not** touched by the install script — it persists across the upgrade.

### Roll the agents, one at a time

```bash
kubectl drain emerald --ignore-daemonsets --delete-emptydir-data

# On the agent:
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION=vX.Y.Z+k3s1 \
  K3S_URL=https://10.0.20.10:6443 \
  K3S_TOKEN="$K3S_TOKEN" sh -

kubectl uncordon emerald
```

Repeat for `topaz` and `amethyst`. Wait for each node to return `Ready` at the new
version before starting the next — `emerald` carries the `app-state=true` local-path
apps, so draining two at once has nowhere to put them.

!!! warning "Single control plane: the snapshot is the rollback"
    There is one server node. If the control plane fails to come back, there is no
    second server to carry the cluster — recovery is the
    [Restore procedure](#step-11-restore-procedure-when-ruby-dies) against the snapshot
    you took in pre-flight. Downgrading k3s in place is not supported once etcd has been
    written by the newer version.

### After the roll { #after-the-roll }

```bash
kubectl get nodes                    # all 4 at the new version, all Ready
kubectl get pods -A | grep -v Running | grep -v Completed
```

Then re-run the Ansible play so a future rebuild matches, and confirm the pinned version
and the live version agree.

## Verification

- [ ] All 4 nodes Ready:

    ```bash
    kubectl get nodes
    ```

- [ ] NFS storage class is default:

    ```bash
    kubectl get sc
    # Expected: nfs-storage (default)
    ```

- [ ] MetalLB pool configured:

    ```bash
    kubectl get ipaddresspool -n metallb-system
    ```

- [ ] ArgoCD pods Running:

    ```bash
    kubectl get pods -n argocd
    # Expected: argocd-server, argocd-repo-server, argocd-application-controller,
    # argocd-applicationset-controller, redis — all Running. (No dex in this build:
    # it's disabled in the Helm values; Authelia is the OIDC provider.)
    ```

- [ ] NFS provisioner Running:

    ```bash
    kubectl get pods -A | grep nfs-provisioner
    ```

- [ ] Sealed Secrets controller Running:

    ```bash
    kubectl get pods -n sealed-secrets
    ```

!!! tip "Next"
    Set up Traefik before deploying any user-facing service. After it's up, return to [Wire ArgoCD to homelab-manifests](../get-started/set-up-git.md#step-7-wire-argocd-to-homelab-manifests) to connect ArgoCD to `homelab-manifests` using the read-only PAT.
