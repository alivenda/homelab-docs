# Runbook 5: Kubernetes Cluster (k3s) on Turing Pi 2

Deploy k3s across all 4 CM4 nodes, using the SATA SSD on Node 3 for NFS persistent storage and your UGREEN NAS for bulk media.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 1–2 hours (after Runbook 3) |
| **Runs On** | Turing Pi 2 (all 4 CM4 nodes) |
| **Depends On** | Runbook 3 (or 4) |

!!! note "If you ran Runbook 4 (Ansible)"
    R4's playbooks already installed NFS on topaz and k3s on all 4 nodes. **Skip Steps 1–4 below** — they are the imperative reference for what Ansible did, useful for understanding but redundant if you ran the playbook. Resume at [Step 5: Install Helm](#step-5-install-helm).

## Step 0: Get `kubectl` working on your machine

Every step from here on uses `kubectl`. Install it locally and copy the kubeconfig so you can talk to the cluster without SSHing into ruby first:

```bash
# macOS
brew install kubectl
# Arch / CachyOS
sudo pacman -S kubectl
# Ubuntu / Debian (snap)
sudo snap install kubectl --classic

# Pull the kubeconfig from ruby and rewrite the server URL.
# Log in as dietpi (root login is disabled in R4); k3s wrote the file
# world-readable via --write-kubeconfig-mode 644, so dietpi can read it.
mkdir -p ~/.kube
scp dietpi@10.0.20.10:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1/10.0.20.10/' ~/.kube/config
chmod 600 ~/.kube/config

kubectl get nodes
# Expected: 4 nodes Ready
```

## Step 1: Install NFS Server on topaz

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

## Step 2: Install k3s Server (ruby)

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
    We disable `servicelb` (MetalLB replaces it) but **do not** pass `--disable-cloud-controller`. With servicelb disabled, that flag takes effect, and because k3s still runs the kubelet with `cloud-provider=external`, nodes get the `node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule` taint with nothing to remove it — pods won't schedule unless you deploy your own external CCM ([k3s#6554](https://github.com/k3s-io/k3s/issues/6554)). Keep the embedded cloud controller; it clears that taint and assigns node addresses.

!!! note "Why `--cluster-init`"
    Without `--cluster-init`, k3s defaults to a sqlite datastore (via kine). `--cluster-init` initializes the embedded etcd datastore instead — required for the `k3s etcd-snapshot` flow in Step 10. The CPU/memory overhead is modest on a CM4 with 8 GB RAM, and you can join additional server nodes later for HA without rebuilding.

!!! warning "Token reuse"
    The token you generate above is the join secret for the entire cluster. Step 3 uses the **same value** on every agent node. Generate once, store in your password manager, paste in both places.

## Step 3: Join Worker Nodes

Run on each of emerald, topaz, amethyst (use the same `K3S_TOKEN` value from Step 2):

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://10.0.20.10:6443 \
  K3S_TOKEN=<paste_token_from_Step_2> sh -
```

Verify from your machine: `kubectl get nodes` (should show all 4 nodes Ready).

## Step 4: Label Nodes by Capacity

ruby is the control plane; the other three are workers. Use labels to express both role and storage capacity for scheduling:

```bash
# Control plane
kubectl label nodes ruby node-role.kubernetes.io/control-plane=true storage=large

# Workers
kubectl label nodes emerald kubernetes.io/role=worker storage=large workload=heavy
kubectl label nodes topaz kubernetes.io/role=worker storage=small role=nfs
kubectl label nodes amethyst kubernetes.io/role=worker storage=small
```

!!! note "`workload=heavy` keeps OCR/build spikes off the control plane"
    `node-role.kubernetes.io/control-plane=true` on ruby is only a **label**, not a taint — k3s does not taint its server by default, so a `nodeSelector: storage: large` (which matches both ruby and emerald) can still land a pod on ruby. The two spiky workloads R0 warns about — Paperless OCR (R14) and Woodpecker builds (R12) — target `workload=heavy` instead, which is only on emerald. If you'd rather hard-fence ruby, taint it (`kubectl taint nodes ruby node-role.kubernetes.io/control-plane=:NoSchedule`), but that also evicts the lighter app pods this build intentionally runs on ruby.

## Step 5: Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh && ./get_helm.sh
```

## Step 6: Install MetalLB

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
    These manifests live at `homelab-manifests/infrastructure/metallb/{ipaddresspool,l2advertisement}.yaml`. Apply via `kubectl apply -f` from a clone of that repo; the YAML above is shown inline for learning context. Edit the repo, not your local copy, so the changes survive a rebuild.

!!! warning "MetalLB pool vs DHCP"
    MetalLB's pool (`10.0.20.200–10.0.20.250`) **must be excluded from the Lab VLAN DHCP scope**. R2's Network Plan bounds Lab VLAN DHCP to `.100–.199` for exactly this reason — if you change either side, change both. Without the bound, UDM will eventually hand out an address in `.200–.250` to a random device and you'll get intermittent IP conflicts that are nightmare to debug. In UniFi Network: Settings → Networks → Lab → DHCP Range should stay `10.0.20.100–10.0.20.199`.

## Step 7: Install NFS Storage Provisioner

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

## Step 8: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Save the printed password to your password manager. (Once R7 is up, Vaultwarden becomes the home for everything cluster-generated.)

!!! note "Why `--server-side`"
    The upstream `install.yaml` is large enough that a client-side `kubectl apply` can hit the 256 KB annotation limit on the embedded CRDs. The current ArgoCD docs recommend server-side apply with `--force-conflicts` for the initial install.

## Step 9: Access ArgoCD (before Traefik is up)

ArgoCD has no Ingress yet — Runbook 6 (Traefik) is the next runbook. To reach the UI in the meantime, port-forward:

```bash
# In a separate terminal:
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then:

- Open <https://localhost:8080> in a browser.
- Accept the self-signed cert warning.
- Username: `admin`
- Password: the value you decoded from the secret in Step 8.

!!! tip
    After Runbook 6 (Traefik) is up, you'll add an IngressRoute so `argocd.yourdomain.com` works without port-forwarding. The port-forward method is the bootstrap path that always works.

## Step 10: Control-plane redundancy (SPOF awareness)

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

Back up the snapshots directory to your NAS (Runbook 10 picks this up if you add it to the restic include list).

## Step 11: Restore procedure (when ruby dies)

If ruby is unrecoverable: reflash DietPi (Runbook 3), re-bootstrap with Ansible (Runbook 4), then restore etcd from your latest snapshot instead of running k3s server fresh:

```bash
# On the new ruby, copy the latest snapshot from your NAS to:
# /var/lib/rancher/k3s/server/db/snapshots/

# Install k3s with --cluster-reset --cluster-reset-restore-path
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/<latest>

# Once it's up, the worker agents (emerald, topaz, amethyst) need to be re-joined with the
# same K3S_TOKEN they used before.
```

!!! tip
    Document your K3S_TOKEN in your password manager the day you stand the cluster up. Losing it on top of losing ruby turns a 2-hour rebuild into a full cluster rebuild.

## Step 12: Sealed Secrets (cluster-side secret management)

You now have two flavors of secrets: those that live in `homelab-secrets` as sops + age (Terraform tfvars, Cloudflare tokens consumed by your machine) and those that need to be Kubernetes Secrets at runtime (DB passwords, OAuth client secrets, image-pull credentials). Don't commit raw `kubectl create secret` commands to Git — install Sealed Secrets so you can commit encrypted manifests to `homelab-manifests` safely.

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

## Step 13: Encrypt your first secret

Worked example: encrypt the Cloudflare API token (which Runbook 6 will use for Traefik DNS-01) as a SealedSecret manifest. Commit the encrypted form to `homelab-manifests`; the cluster decrypts it at runtime.

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
    # Expected: argocd-server, argocd-repo-server, argocd-application-controller, redis, dex-server all 1/1
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
    Runbook 6 (Traefik) before deploying any user-facing service. After Traefik is up, return to [Runbook 1 Step 7](01-git.md#step-7-wire-argocd-to-homelab-manifests) to wire ArgoCD to `homelab-manifests` using the read-only PAT.
