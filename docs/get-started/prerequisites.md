# Prerequisites

What to have and understand before you start: the hardware to buy, the accounts to create, and the platform realities that shape every later runbook.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | Varies — shopping + account setup, before any deploy work starts |
| **See Also** | [Home overview](../index.md) for how the runbooks fit together and what you end up with |
| **Skip If** | You already run a homelab — no later runbook treats this one as a hard dependency |

## Hardware shopping list { #hardware-shopping-list }

The runbook commands assume this exact hardware. You can substitute (an x86 mini-PC cluster, a single beefy node, an existing NAS), but the commands won't match.

| Component | Spec | Notes |
|---|---|---|
| Turing Pi 2 cluster board | mini-ITX | |
| 4× Raspberry Pi CM4 modules | 8 GB RAM + WiFi each — 2× 32 GB eMMC (ruby / Node 1, topaz / Node 3), 2× 16 GB eMMC (emerald / Node 2, amethyst / Node 4) | |
| SATA III SSD | any size 250 GB+ | Cluster NFS storage — connects to topaz (Node 3) |
| PicoPSU (24-pin) | 120 W | The cluster draws 30–60 W under load. 120 W leaves headroom while staying small and silent. A standard ATX PSU works too, but runs inefficiently at this low draw. |
| Ubiquiti UDM-Pro or UDM-SE | — | VLANs, firewall, DHCP |
| UGREEN DXP6800 Pro NAS | or any NAS that runs Docker | Bulk media + Immich + offsite-friendly bulk storage |
| Dedicated Home Assistant OS host | this build uses **slate**, a repurposed Late-2014 Mac mini (16 GB RAM, 256 GB SSD) running Proxmox, with HAOS as a VM (2 vCPU, 4 GB RAM) | Kept off the cluster so the smart-home hub survives cluster reboots and upgrades. 4 GB for the VM matches HA's reference spec; only raise it for Frigate NVR or long-retention history. A Raspberry Pi 5 (4 GB) running HAOS bare-metal is an equally good dedicated host. |
| Raspberry Pi for AdGuard Home DNS | pyrite; this build uses a Pi 3 Model B — add a 2nd for optional failover | Run natively rather than on k3s so DNS stays up independently of the cluster |
| Domain name | registered through Cloudflare, ~$10/yr for `.com` / `.net` / etc. | |
| UPS (optional) | | Recommended once you start storing real data on the cluster — [sizing in Turing Pi](../build/turing-pi.md#power-ups-nut-for-graceful-shutdown) |

## Accounts you need { #accounts-youll-need }

Create these before starting [Set up Git](set-up-git.md). All free tiers are sufficient.

| Account | Why |
|---|---|
| **GitHub** | Primary Git host (Git, Kubernetes, Forgejo, Home Assistant) |
| **Cloudflare** | DNS + free TLS through DNS-01 (Traefik, Terraform) |
| **Tailscale** | Free tier covers up to 100 devices |
| **Docker Hub** (optional) | Pulling public images without rate limits |
| **A bootstrap password manager** (1Password, Bitwarden Cloud, KeePassXC) | You replace this with self-hosted Vaultwarden later, but you need somewhere to store PATs and API tokens during the bootstrap weeks |

## Namespace strategy { #namespace-strategy }

Every workload lives in a namespace. Set conventions now so you can `grep` your cluster meaningfully a year in:

- **System-level (cluster-wide infrastructure):** namespace ends in `-system`. Examples: `metallb-system`, `traefik`, `argocd`, `sealed-secrets`, `monitoring`. Don't host application workloads in these.
- **One namespace per user-facing app:** `forgejo`, `vaultwarden`, `nextcloud`, `paperless`, `woodpecker`. Don't use `default` for anything you need to clean up later.
- **All resources for an app** (Deployment, Service, PVC, SealedSecret, HTTPRoute) live in the app's namespace. RBAC and NetworkPolicies operate at namespace granularity, so colocation is what makes those tools effective.
- **ArgoCD Application objects themselves live in `argocd`.** Each Application points at a manifests path that targets the destination namespace.

!!! tip "Prune stale namespaces"
    Run `kubectl get ns` periodically and prune anything you don't recognize. A messy namespace list is a sign of sloppy GitOps — every namespace must map to either a system component or an Application in `homelab-manifests/`.

## Reality of ARM64 homelabs { #reality-of-arm64-homelabs }

Running CM4 modules means every image you deploy must exist as arm64. Most mainstream projects publish multiarch manifests, but the long tail does not. The failure mode is rarely a clean error — it's often a confusing crashloop or a registry pull that silently grabs the amd64 image and segfaults on launch.

Friction takes these shapes:

- Some Helm charts hardcode amd64 in initContainer images or sidecar versions. Check the `values.yaml` file for `image.tag` fields you can override before assuming a chart works.
- CI plugin ecosystems (Drone, Woodpecker) lag on multiarch. Plugins that work on your machine might not have arm64 builds. When in doubt, run the build natively inside the runner instead of through a plugin.
- Vendor images (database GUIs, observability sidecars, niche connectors) are the most common amd64-only offenders. Self-host the open-source equivalent if you hit one.
- Multiarch tags are inconsistent. `:latest` might be multiarch while `:1.2.3` is amd64-only, or vice versa. Pin a tag, then verify with `docker manifest inspect <image>:<tag>`.

**Debugging strategy** when a pod refuses to start: check events with `kubectl describe pod`, then run `docker manifest inspect` on the image. If the manifest lists only `linux/amd64`, you have three options — find a community arm64 build, build it yourself from source, or pin that service to a NAS-Docker host that runs amd64.

!!! tip "ARM image debugging is a skill"
    Treat ARM image debugging as a learning feature, not a bug. You get fluent with OCI manifests, multiarch builds, and image inspection — all transferable skills.

## Resource budget expectations { #resource-budget-expectations }

Four CM4 modules give you roughly 32 GB total RAM (8 GB each). After k3s overhead and the Traefik / ArgoCD / Prometheus baseline, you have meaningful but not infinite headroom. Use the table below as a planning aid — real numbers drift based on config, retention windows, and load.

| Service | Memory (steady) | CPU (steady) | Notes |
|---|---|---|---|
| k3s (per node) | 300–500 Mi | 0.1–0.3 vCPU | Server slightly heavier than agent |
| Traefik | 80–150 Mi | <0.1 vCPU | Per replica |
| ArgoCD | 400–600 Mi | 0.1 vCPU idle | Spikes during sync |
| Forgejo | 300–500 Mi | 0.1 vCPU idle | Higher on clone/push bursts |
| Vaultwarden | 50–100 Mi | <0.05 vCPU | Lightweight Bitwarden compat |
| Prometheus | 500–800 Mi | 0.1–0.2 vCPU | Grows with retention + targets |
| Loki | 200–400 Mi | 0.1 vCPU | Add ingester memory if log volume is high |
| Grafana | 150–250 Mi | <0.1 vCPU | Per replica |
| Paperless-ngx | 200 Mi idle, 1 Gi+ OCR | 0.1 idle, 1+ vCPU OCR | OCR is the spike |
| Sealed Secrets controller | 50–100 Mi | <0.05 vCPU | One per cluster |
| Woodpecker server | 100–200 Mi | <0.1 vCPU | Excludes runner build load |
| Woodpecker runner | 200 Mi idle, 1–2 Gi build | 1+ vCPU during build | Spike during pipelines |

!!! tip "Set resource limits on every workload"
    Pi cluster memory pressure surprises people. Add resource requests and limits to every workload you deploy so the scheduler can refuse to pack a node into OOM territory. Without limits, one runaway pod can wedge a whole node.

!!! warning
    Paperless OCR and Woodpecker builds are the two workloads most likely to push a node over. Pin them to a dedicated worker that is **not** the control plane — emerald (Node 2) in this build. Their spikes must not coexist with control-plane pods on ruby (Node 1). The pin is about CPU and memory isolation, not disk: emerald's eMMC is 16 GB, so bulk data (documents, build artifacts) belongs on the NFS tier, not the node disk.
