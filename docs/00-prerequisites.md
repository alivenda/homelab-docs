# Runbook 0: Prerequisites and Mental Model

Read this before Runbook 1 — it covers what you need to **have and understand** before you start: the hardware to buy, the accounts to create, and the platform realities (ARM64 images, namespace conventions, resource budget) that shape every later runbook. For the big picture — how the runbooks fit together and what you end up with — see the [Home overview](index.md). You can skip this runbook if you already run a homelab; none of the later runbooks reference it as a hard dependency.

## Hardware Shopping List

This guide is written against a specific build. You can substitute (an x86 mini-PC cluster, a single beefy node, an existing NAS, etc.), but the runbook commands assume this exact hardware:

- Turing Pi 2 cluster board (mini-ITX)
- 4× Raspberry Pi CM4 modules with 8 GB RAM and WiFi — 2× with 32 GB eMMC (ruby (Node 1), emerald (Node 2)), 2× with 16 GB eMMC (topaz (Node 3), amethyst (Node 4))
- 1× SATA III SSD (any size 250 GB+) for cluster NFS storage — connected to topaz (Node 3)
- PicoPSU (24-pin), **120 W** — the cluster pulls 30–60 W under load, so 120 W leaves comfortable headroom while staying tiny and silent. A standard ATX PSU works too, but it's bulky and runs inefficiently at this low draw.
- Ubiquiti UDM-Pro or UDM-SE for VLANs, firewall, DHCP
- UGREEN DXP6800 Pro NAS (or any NAS that runs Docker) for bulk media + Immich + offsite-friendly bulk storage
- 1× Raspberry Pi 5 (4 GB) for Home Assistant OS — a dedicated host, kept off the cluster so the smart-home hub survives cluster reboots and upgrades (R16). 4 GB is the right size (HA's own reference spec); only go 8 GB if you'll run Frigate NVR or long-retention history.
- 2× Raspberry Pi Zero 2 W for AdGuard Home (primary + secondary DNS) — run natively rather than on k3s so DNS stays up independently of the cluster (R17)
- Domain name registered through Cloudflare (~$10/yr for `.com` / `.net` / etc.)
- Optional: UPS — recommended once you start storing real data on the cluster ([sizing in R3](03-turing-pi.md#power-ups-nut-for-graceful-shutdown))

## Accounts You'll Need

Create these before starting Runbook 1. All free tiers are sufficient.

- **GitHub** — primary Git host (Runbooks 1, 5, 11, 16)
- **Cloudflare** — DNS + free TLS via DNS-01 (Runbooks 6, 8)
- **Tailscale** — free tier covers up to 100 devices (Runbook 2)
- **Docker Hub** (optional) — for pulling public images without rate limits
- **A bootstrap password manager** (1Password, Bitwarden Cloud, KeePassXC) — you will replace this with self-hosted Vaultwarden at Runbook 7, but you need somewhere to store PATs and API tokens during the bootstrap weeks

## Namespace Strategy

k3s isn't a single shared bucket — every workload lives in a namespace. Set conventions now so you can `grep` your cluster meaningfully a year in:

- **System-level (cluster-wide infrastructure):** namespace ends in `-system`. Examples: `metallb-system`, `traefik`, `argocd`, `sealed-secrets`, `monitoring`. These shouldn't host application workloads.
- **One namespace per user-facing app:** `forgejo`, `vaultwarden`, `nextcloud`, `paperless`, `woodpecker`. Never use `default` for anything you'll later need to clean up.
- **All resources for an app** (Deployment, Service, PVC, SealedSecret, HTTPRoute) live in the app's namespace. RBAC and NetworkPolicies operate at namespace granularity — keeping things colocated is what makes those tools effective.
- **ArgoCD Application objects themselves live in `argocd`.** Each Application points at a manifests path that targets the destination namespace.

!!! tip
    Run `kubectl get ns` periodically and prune anything you don't recognize. A messy namespace list is the canary for sloppy GitOps — every namespace should map to either a system component or an Application in `homelab-manifests/`.

## Reality of ARM64 Homelabs

Running CM4 modules means everything you deploy needs to exist as an arm64 image. Most mainstream projects publish multiarch manifests now, but the long tail does not, and the failure mode is rarely a clean error — it is often a confusing crashloop or a registry pull that silently grabs the amd64 image and segfaults on launch.

Specific shapes the friction takes:

- Some Helm charts hardcode amd64 in initContainer images or sidecar versions. Check `values.yaml` for `image.tag` fields you can override before assuming a chart works.
- CI plugin ecosystems (Drone, Woodpecker) lag on multiarch. Plugins that work locally on your machine may not have arm64 builds. When in doubt, run the build natively inside the runner instead of via plugin.
- Vendor images (database GUIs, observability sidecars, niche connectors) are the most common amd64-only offenders. Self-host the open-source equivalent if you hit one.
- Multiarch tags are inconsistent. `:latest` may be multiarch while `:1.2.3` is amd64-only, or vice versa. Always pin a tag, then verify with `docker manifest inspect <image>:<tag>`.

**Debugging strategy** when a pod refuses to start: check events with `kubectl describe pod`, then `docker manifest inspect` the image. If the manifest only lists `linux/amd64`, you have three options — find a community arm64 build, build it yourself from source (often easier than expected for Go binaries), or pin that service to a NAS-Docker host that runs amd64.

!!! tip
    Treat ARM image debugging as a learning feature, not a bug. You will get fluent with OCI manifests, multiarch builds, and image inspection — all transferable skills.

## Resource Budget Expectations

Four CM4 modules give you roughly 32 GB total RAM (8 GB each). After k3s overhead and Traefik / ArgoCD / Prometheus baseline, you have meaningful but not infinite headroom. Use the table below as a rough planning aid — real numbers will drift based on config, retention windows, and load.

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

!!! tip
    Pi cluster memory pressure surprises people. Add resource requests + limits to every workload you deploy so the scheduler can refuse to pack a node into OOM territory. Without limits, one runaway pod can wedge a whole node.

!!! warning
    Paperless OCR and Woodpecker builds are the two workloads most likely to push a node over. Pin them to a 32 GB-eMMC worker that is **not** the control plane — `emerald (Node 2)` in this build. Their spikes should not coexist with control-plane pods on `ruby (Node 1)`.
