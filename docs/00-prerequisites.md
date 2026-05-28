# Runbook 0: Prerequisites and Mental Model

Read this before Runbook 1. It tells you what to buy, what accounts to create, and how the 16 runbooks fit together. You can skip it if you already have a homelab — none of the later runbooks reference Runbook 0 content as a hard dependency.

## Hardware Shopping List

This guide is written against a specific build. You can substitute (an x86 mini-PC cluster, a single beefy node, an existing NAS, etc.), but the runbook commands assume this exact hardware:

- Turing Pi 2 cluster board (mini-ITX)
- 4× Raspberry Pi CM4 modules with 8 GB RAM and WiFi — 2× with 32 GB eMMC (ruby (Node 1), emerald (Node 2)), 2× with 16 GB eMMC (topaz (Node 3), amethyst (Node 4))
- 1× SATA III SSD (any size 250 GB+) for cluster NFS storage — connected to topaz (Node 3)
- ATX or PicoPSU power supply (24-pin), **minimum 200 W** — the cluster pulls 30–60 W under load, so any modern ATX PSU is overkill but cheapest
- Ubiquiti UDM-Pro or UDM-SE for VLANs, firewall, DHCP
- UGREEN DXP6800 Pro NAS (or any NAS that runs Docker) for bulk media + Immich + offsite-friendly bulk storage
- Domain name registered through Cloudflare (~$10/yr for `.com` / `.net` / etc.)
- Optional: UPS — recommended once you start storing real data on the cluster ([sizing in R3](03-turing-pi.md#power-ups-nut-for-graceful-shutdown))

## Accounts You'll Need

Create these before starting Runbook 1. All free tiers are sufficient.

- **GitHub** — primary Git host (Runbooks 1, 5, 11, 16)
- **Cloudflare** — DNS + free TLS via DNS-01 (Runbooks 6, 8)
- **Tailscale** — free tier covers up to 100 devices (Runbook 2)
- **Docker Hub** (optional) — for pulling public images without rate limits
- **A bootstrap password manager** (1Password, Bitwarden Cloud, KeePassXC) — you will replace this with self-hosted Vaultwarden at Runbook 7, but you need somewhere to store PATs and API tokens during the bootstrap weeks

## Dependency Map

The guide is split into **Foundation → Infrastructure → Apps**. Infrastructure is the recommended linear path: each runbook unblocks the next. Apps (R13–R16) only hard-depend on R6 (Traefik) — the order shown is convenience, not requirement.

```
─── Foundation ──────────────────────────────────────────────
R0 Prerequisites + Mental Model
 ↓
R1 Git / GitOps foundation (sops, pre-commit, 5 repos)

─── Infrastructure (platform + operator tooling) ────────────
R2 UDM VLANs                  ← Tailscale step deferred until R3 done
 ↓
R3 Flash DietPi to 4× CM4     ← step 5 superseded by R4 Ansible
 ↓
R4 Ansible: nodes, NFS, k3s install (replaces R3 step 5)
 ↓
R5 k3s bring-up: MetalLB, NFS storage class, ArgoCD, Sealed Secrets
 ↓
R6 Traefik HTTPS (DNS-01 via Cloudflare)
 ↓
R7 Vaultwarden                ← cred store; becomes the password manager for every later runbook
 ↓
R8 Terraform (Cloudflare DNS + UniFi IaC; retroactive)
 ↓
R9 Prometheus + Grafana + Loki
 ↓
R10 Restic + Velero backups
 ↓
R11 Forgejo                   ← self-hosted Git host
 ↓
R12 Woodpecker CI/CD          ← build → push → bump manifest

─── Apps (user-facing services) ─────────────────────────────
 ├─→ R13 Nextcloud       (cluster)
 ├─→ R14 Paperless-ngx   (cluster)
 ├─→ R15 Immich          (NAS-Docker)
 └─→ R16 Home Assistant  (NAS-Docker)
```

## How to Use This Guide

- Read each runbook fully before starting it. Several runbooks reference "come back to this step after Runbook N" patterns — don't get stuck mid-step.
- Treat the `Depends On` header as the prerequisite check. If a runbook says `Depends On: Runbook 5`, do not start it until R5's Verification section passes.
- When a runbook gives you a `docker-compose.yml`, check the `Runs On` header carefully. NAS-hosted services use compose; cluster-hosted services use Helm charts or k8s manifests committed to `homelab-manifests` so ArgoCD manages them.
- If you get stuck on the order, re-read the dependency map above. The most common confusion is Tailscale (R2 Step 3) requiring ruby (Node 1) from R3 first, and ArgoCD (R5 Step 8) needing `kubectl port-forward` to access before R6 is up.

## Namespace Strategy

k3s isn't a single shared bucket — every workload lives in a namespace. Set conventions now so you can `grep` your cluster meaningfully a year in:

- **System-level (cluster-wide infrastructure):** namespace ends in `-system`. Examples: `metallb-system`, `traefik`, `argocd`, `sealed-secrets`, `monitoring`. These shouldn't host application workloads.
- **One namespace per user-facing app:** `forgejo`, `vaultwarden`, `nextcloud`, `paperless`, `woodpecker`. Never use `default` for anything you'll later need to clean up.
- **All resources for an app** (Deployment, Service, PVC, SealedSecret, IngressRoute) live in the app's namespace. RBAC and NetworkPolicies operate at namespace granularity — keeping things colocated is what makes those tools effective.
- **ArgoCD Application objects themselves live in `argocd`.** Each Application points at a manifests path that targets the destination namespace.

!!! tip
    Run `kubectl get ns` periodically and prune anything you don't recognize. A messy namespace list is the canary for sloppy GitOps — every namespace should map to either a system component or an Application in `homelab-manifests/`.

## End State

When you finish all 16 runbooks you will have: a 4-node k3s cluster on your desk; HTTPS reverse-proxied internal services at `*.yourdomain.com`; self-hosted password vault, photo library, file sync, document archive, and smart-home hub; a GitOps loop where pushing to `homelab-manifests` triggers a cluster reconcile; monitoring with Prometheus / Grafana / Loki; nightly Restic backups; and a CI/CD pipeline that builds container images and bumps deployments via Git.

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
| Immich (NAS-side) | 2–4 Gi | 1–2 vCPU + GPU helpful | Not on cluster by design |

!!! tip
    Pi cluster memory pressure surprises people. Add resource requests + limits to every workload you deploy so the scheduler can refuse to pack a node into OOM territory. Without limits, one runaway pod can wedge a whole node.

!!! warning
    Paperless OCR and Woodpecker builds are the two workloads most likely to push a node over. Pin them to a 32 GB-eMMC worker that is **not** the control plane — `emerald (Node 2)` in this build. Their spikes should not coexist with control-plane pods on `ruby (Node 1)`.
