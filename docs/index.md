# Homelab Runbooks

**Turing Pi 2 Edition — From Zero to Self-Hosted Everything**

A 16-runbook guide for building a 4-node k3s cluster on a Turing Pi 2, learning DevOps practices that transfer to professional work, and replacing the cloud services you rent with services you own.

## Target build

- Turing Pi 2 cluster board (mini-ITX) + 4× Raspberry Pi CM4 with 8 GB RAM
- Ubiquiti UDM rackmount for VLANs, firewall, DHCP
- UGREEN DXP6800 Pro NAS (or any Docker-capable NAS) for bulk media and Immich
- Cloudflare-registered domain (~$10/yr)

You can substitute hardware, but commands are written against this exact build.

## What you end up with

A 4-node k3s cluster running self-hosted Git (Forgejo), password manager (Vaultwarden), file sync (Nextcloud), photo library (Immich), document archive (Paperless-ngx), smart-home hub (Home Assistant), monitoring (Prometheus / Grafana / Loki), nightly backups (Restic + Velero), and a CI/CD pipeline (Woodpecker) that builds container images and reconciles them via GitOps (ArgoCD watching `homelab-manifests`).

## How the runbooks fit together

[Runbook 0](00-prerequisites.md) sets the mental model — read it first. The guide is split into **Foundation → Infrastructure → Apps**:

```
─── Foundation ──────────────────────────────────────────────
R0 Prerequisites
 ↓
R1 Git foundation (5 repos, sops, pre-commit)

─── Infrastructure (platform + operator tooling) ────────────
R2 UDM VLANs                   ← Tailscale step waits on R3
 ↓
R3 Flash DietPi to 4× CM4      ← step 5 (SSD prep) superseded by R4
 ↓
R4 Ansible (replaces R3 step 5, installs k3s)
 ↓
R5 k3s bring-up (MetalLB, NFS storage, ArgoCD, Sealed Secrets)
 ↓
R6 Traefik HTTPS (DNS-01 via Cloudflare)
 ↓
R7 Vaultwarden                 ← cred store for every later runbook
 ↓
R8 Terraform (Cloudflare DNS + UniFi IaC; retroactive)
 ↓
R9 Prometheus + Grafana + Loki
 ↓
R10 Restic + Velero backups
 ↓
R11 Forgejo
 ↓
R12 Woodpecker CI/CD

─── Apps (user-facing services) ─────────────────────────────
 ├─→ R13 Nextcloud       (cluster)
 ├─→ R14 Paperless-ngx   (cluster)
 ├─→ R15 Immich          (NAS-Docker — not on k3s, see runbook for why)
 └─→ R16 Home Assistant  (NAS-Docker)
```

## How to use this guide

- Read each runbook fully before starting it. Several reference "come back to this after Runbook N" patterns — skim first so you don't get stuck mid-step.
- Treat the `Depends On` header as the prerequisite check. If a runbook says "Depends On: Runbook 5", don't start until R5's Verification section passes.
- When a runbook gives you a `docker-compose.yml`, check the `Runs On` header. NAS-hosted services use compose; cluster-hosted services use Helm + manifests committed to `homelab-manifests` so ArgoCD manages them.
- The most common ordering confusion is Tailscale (R2 Step 3 needs ruby from R3) and ArgoCD (R5 Step 8 needs port-forward to access before R6 is up). Both are flagged where they appear.

!!! tip "Bookmark Runbook 0"
    When you hit a "wait, when am I supposed to do X" moment three weeks in, the dependency map in R0 answers it without scrolling the whole guide.

## Version

The current source set is v15. See the [Version History](version-history.md) for the prior PDF lineage and what each release added.
