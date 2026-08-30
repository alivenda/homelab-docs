# Homelab Runbooks

**Turing Pi 2 Edition — From zero to self-hosted everything**

Build a 4-node k3s cluster on a Turing Pi 2, learn DevOps practices that transfer to professional work, and replace the cloud services you rent with services you own.

## Target build

| Component | Spec |
|---|---|
| Cluster | Turing Pi 2 (mini-ITX) + 4× Raspberry Pi CM4, 8 GB RAM each |
| Network | Ubiquiti UDM rackmount — VLANs, firewall, DHCP |
| NAS | UGREEN DXP6800 Pro (x86, 8 GB DDR5) — bulk media and Immich |
| Home Assistant host | slate — repurposed Late-2014 Mac mini (16 GB RAM, 256 GB SSD), Proxmox host running Home Assistant OS in a VM (2 vCPU, 4 GB) |
| DNS | 1× Raspberry Pi for AdGuard Home (pyrite — this build: Pi 3 Model B; add a 2nd for optional failover) |
| Domain | Cloudflare-registered domain (~$10/yr) |

You can substitute hardware, but commands are written against this exact build. Full parts list (PSU, SSD, UPS, and the rest) is in [Prerequisites](get-started/prerequisites.md).

## What you end up with

A 4-node k3s cluster running:

- Self-hosted Git (Forgejo), password manager (Vaultwarden), SSO gateway (Authelia)
- File sync (Nextcloud), photo library (Immich), document archive (Paperless-ngx)
- Smart-home hub (Home Assistant), monitoring (Prometheus / Grafana / Loki / Alloy)
- Nightly backups (Velero + Garage S3), CI (Woodpecker), GitOps deployment (ArgoCD)

Plus a full personal-cloud layer covering notes, tasks, finance, media automation, books, recipes, and more.

## The five repos

Everything is split across five Git repositories, each with its own purpose, security boundary, and consumer. See [Repositories](concepts/repositories.md) for the rationale and [Set up Git](get-started/set-up-git.md) for the setup steps.

| Repo | Purpose | Consumer |
|---|---|---|
| `homelab-docs` | Runbooks, diagrams, decisions log | You (humans) |
| `homelab-ansible` | OS provisioning playbooks | Your machine |
| `homelab-manifests` | k3s YAML, Helm values, HTTPRoutes | ArgoCD |
| `homelab-terraform` | Cloudflare DNS, UniFi config, cloud practice | Your machine → Woodpecker |
| `homelab-secrets` | Encrypted secrets (sops/age) — **PRIVATE** | Your machine (with sops) |

## How the runbooks fit together

Read [Prerequisites](get-started/prerequisites.md) first — it sets the mental model.

The navigation groups pages by what you're doing with them: **Overview**, **Get started**,
**Concepts**, **Build the cluster**, **Deploy services**, **Operations**, **Reference**, and
**Release notes**. Build order cuts across those groups. Vaultwarden is a Deploy services
page, but it comes early because later runbooks store their credentials there.

```
─── Get started ────────────────────────────────────────────
Prerequisites
 ↓
Set up Git (5 repos, sops, pre-commit)

─── Cluster and platform, in build order ────────────────────
Network — UDM VLANs              ← Tailscale step waits on Turing Pi
 ↓
Turing Pi — flash DietPi to 4× CM4   ← SSD-prep step superseded by Ansible
 ↓
Ansible (replaces the SSD-prep step, installs k3s)
 ↓
Kubernetes — k3s bring-up (MetalLB, NFS storage, ArgoCD, Sealed Secrets)
 ↓
Traefik — HTTPS (DNS-01 via Cloudflare)
 ↓
Vaultwarden                      ← credential store for later runbooks
 ↓
Terraform (Cloudflare DNS + UniFi IaC; retroactive)
 ↓
Observability (Prometheus / Grafana / Loki / Alloy)
 ↓
Backups (Garage S3 + Velero)
 ↓
Forgejo
 ↓
Woodpecker
 ↓
AdGuard Home             ← DNS ad-blocking (dedicated Pi — not k3s)
 ↓
Authelia + lldap         ← SSO + OIDC provider (depends on Traefik, Vaultwarden)
 ↓
ntfy                     ← push notifications for the whole stack
 ↓
NAS PostgreSQL           ← shared DB server (NAS Docker — not k3s)

─── Applications · full runbooks · live (DB / multi-service / special) ─
 ├─→ Nextcloud       (cluster — DB on NAS Postgres)
 ├─→ Paperless-ngx   (cluster — DB on NAS Postgres + Redis)
 ├─→ Immich          (NAS-Docker — not on k3s, see runbook for why)
 ├─→ Home Assistant  (slate — Mac mini / Proxmox VM)
 ├─→ Homepage        (cluster — config-heavy dashboard)
 ├─→ Vikunja         (cluster — DB on NAS Postgres)
 └─→ Miniflux        (cluster — DB on NAS Postgres, stateless app)

─── Applications · full runbooks · planned ─────────────────────
 ├─→ Arr Stack       (shelved — physical-media-first library)
 ├─→ BookStack       (cluster — MariaDB on NAS)
 ├─→ Syncthing       (per-device — not k3s)
 ├─→ RustDesk Server (cluster — TCP/UDP relay via MetalLB)
 ├─→ Reactive Resume (cluster — Postgres + Redis + object store)
 └─→ Ollama + WebUI  (NAS Docker — after 16 GB RAM upgrade)

─── Applications · catalog (one shared pattern) ──────────────
 │  See: Deploying an App (pattern) + App Catalog
 ├─→ live:    Actual Budget · Audiobookshelf · Collabora · Donetick · linkding
 └─→ planned: Kavita · Mealie · TriliumNext
```

**Runbook or catalog row?** An app gets its own runbook when it has a relational database, multiple components, a non-cluster deployment model, a non-HTTP protocol, heavy configuration, or is an auth backbone. Everything simpler is a one-pattern HTTP app and lives as a row in the [App Catalog](reference/app-catalog.md). Each app and service page carries a **Status** banner (Live / Planned / Shelved / Retired) at the top.

| Status | Apps |
|---|---|
| Live | Actual Budget · Audiobookshelf · Collabora Online · Donetick · linkding |
| Planned | Kavita · Mealie · TriliumNext Notes |
| Retired | Uptime Kuma |

→ [App Catalog](reference/app-catalog.md) for the per-app deltas, [Deploying an App](deploy/index.md) for the shared pattern.

## How to use this guide

- Read each runbook fully before starting it. Several reference "come back to this after X" patterns — skim first so you don't get stuck mid-step.
- Treat the **Depends On** header as the prerequisite check. If a runbook says "Depends On: Kubernetes," don't start until the Kubernetes verification section passes.
- When a runbook gives you a `docker-compose.yml` file, check the **Runs On** header. NAS-hosted services use Compose; cluster-hosted services use Helm and manifests committed to `homelab-manifests` so ArgoCD manages them.
- The most common ordering confusion: Tailscale (the Network page needs ruby from Turing Pi) and ArgoCD (the Kubernetes page needs a port-forward before Traefik is up). Both are flagged where they appear.

!!! tip "Bookmark this page"
    When you hit a "when am I supposed to do X" moment three weeks in, the dependency map answers it.

## Version

The current source set is v21. See [Release notes](release-notes.md) for the prior PDF lineage and what each release added.
