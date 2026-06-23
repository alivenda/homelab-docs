# Homelab Runbooks

**Turing Pi 2 Edition — From Zero to Self-Hosted Everything**

A runbook guide for building a 4-node k3s cluster on a Turing Pi 2, learning DevOps practices that transfer to professional work, and replacing the cloud services you rent with services you own.

## Target build

- Turing Pi 2 cluster board (mini-ITX) + 4× Raspberry Pi CM4 with 8 GB RAM
- Ubiquiti UDM rackmount for VLANs, firewall, DHCP
- UGREEN DXP6800 Pro NAS (x86, 8 GB DDR5) for bulk media and Immich
- slate — repurposed Late-2014 Mac mini (16 GB RAM, 256 GB SSD), Proxmox host running Home Assistant OS in a VM (2 vCPU, 4 GB)
- 1× Raspberry Pi for AdGuard Home DNS (`pyrite` — this build: Pi 3 Model B; add a 2nd for optional failover)
- Cloudflare-registered domain (~$10/yr)

You can substitute hardware, but commands are written against this exact build. Full parts list (PSU, SSD, UPS, and the rest) is in [Prerequisites](prerequisites.md).

## What you end up with

A 4-node k3s cluster running self-hosted Git (Forgejo), password manager (Vaultwarden), SSO gateway (Authelia), file sync (Nextcloud), photo library (Immich), document archive (Paperless-ngx), smart-home hub (Home Assistant), monitoring (Prometheus / Grafana / Loki / Alloy), nightly backups (Velero + Garage S3), and a CI/CD pipeline (Woodpecker) that builds container images and reconciles them via GitOps (ArgoCD watching `homelab-manifests`). Plus a full personal cloud layer covering notes, tasks, finance, media automation, books, recipes, and more.

## How the runbooks fit together

[Prerequisites](prerequisites.md) sets the mental model — read it first. The guide is split into **Foundation → Infrastructure → Apps**:

```
─── Foundation ──────────────────────────────────────────────
Prerequisites
 ↓
Git foundation (5 repos, sops, pre-commit)

─── Infrastructure (platform + operator tooling) ────────────
Networking — UDM VLANs           ← Tailscale step waits on Turing Pi
 ↓
Turing Pi — flash DietPi to 4× CM4   ← SSD-prep step superseded by Ansible
 ↓
Ansible (replaces the SSD-prep step, installs k3s)
 ↓
Kubernetes — k3s bring-up (MetalLB, NFS storage, ArgoCD, Sealed Secrets)
 ↓
Traefik — HTTPS (DNS-01 via Cloudflare)
 ↓
Vaultwarden                      ← cred store for every later runbook
 ↓
Terraform (Cloudflare DNS + UniFi IaC; retroactive)
 ↓
Observability (Prometheus / Grafana / Loki / Alloy)
 ↓
Backups (Garage S3 + Velero)
 ↓
Forgejo
 ↓
Woodpecker CI/CD
 ↓
AdGuard Home             ← DNS ad-blocking (dedicated Pi — not k3s)
 ↓
Authelia + lldap         ← SSO + OIDC provider (depends on Traefik, Vaultwarden)
 ↓
ntfy                     ← push notifications for the whole stack
 ↓
NAS PostgreSQL           ← shared DB server (NAS Docker — not k3s)

─── Apps · full runbooks (DB / multi-service / special model) ─
 ├─→ Nextcloud       (cluster — DB on NAS Postgres)
 ├─→ Paperless-ngx   (cluster — DB on NAS Postgres + Redis)
 ├─→ Immich          (NAS-Docker — not on k3s, see runbook for why)
 ├─→ Home Assistant  (slate — Mac mini / Proxmox VM)
 ├─→ Homepage        (cluster — config-heavy dashboard)
 ├─→ Arr Stack       (cluster — 6 services + hardlinks; NAS after 16 GB)
 ├─→ BookStack       (cluster — MariaDB on NAS Postgres)
 ├─→ Syncthing       (per-device — not k3s)
 ├─→ RustDesk Server (cluster — TCP/UDP relay via MetalLB)
 ├─→ Reactive Resume (cluster — Postgres + Redis + MinIO)
 └─→ Ollama + WebUI  (NAS Docker — defer until 16 GB RAM upgrade)

─── Apps · catalog (simple HTTP apps — one shared pattern) ────
 │  See: Deploying an App (pattern) + App Catalog
 └─→ Actual Budget · Audiobookshelf · Collabora · Donetick · FreshRSS ·
     Kavita · linkding · Mealie · TriliumNext · Vikunja
```

## How to use this guide

- Read each runbook fully before starting it. Several reference "come back to this after the such-and-such runbook" patterns — skim first so you don't get stuck mid-step.
- Treat the `Depends On` header as the prerequisite check. If a runbook says "Depends On: Kubernetes", don't start until Kubernetes's Verification section passes.
- When a runbook gives you a `docker-compose.yml`, check the `Runs On` header. NAS-hosted services use compose; cluster-hosted services use Helm + manifests committed to `homelab-manifests` so ArgoCD manages them.
- The most common ordering confusion is Tailscale (Networking Step 3 needs ruby from Turing Pi) and ArgoCD (Kubernetes Step 8 needs port-forward to access before Traefik is up). Both are flagged where they appear.

!!! tip "Bookmark this page"
    When you hit a "wait, when am I supposed to do X" moment three weeks in, the dependency map above answers it without scrolling the whole guide.

## Version

The current source set is v19. See the [Version History](version-history.md) for the prior PDF lineage and what each release added.
