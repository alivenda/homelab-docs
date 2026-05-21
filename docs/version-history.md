# Version History

The doc lineage. Body content always reflects the current state; this appendix captures the shape of each release.

## v13 (current)

Format migration to Markdown + MkDocs Material. The runbooks are now authored as one `.md` file per runbook under `docs/`, rendered as a static site via MkDocs Material, and render natively in GitHub as a fallback view. Folded in the 27 read-through findings from the v12 dry run:

- **Critical fixes:** Numbered-list continuation bug is gone (Markdown restarts numbering per section). Fish-function helper renders as a proper fenced code block. Woodpecker sample pipeline no longer mounts `/var/run/docker.sock` — replaced with kaniko, matching the doc's own "kubernetes backend doesn't need the socket" claim. Token variable in the bump-manifest step is now `${FORGEJO_TOKEN}` to match the `forgejo_token` secret. R6 Step 3 has a complete title ("Create the Cloudflare Token Secret and Install Traefik").
- **High-impact additions:** New Step 0 in R5 covering `kubectl` install + kubeconfig copy to the laptop. R5 opens with an explicit "skip Steps 1–4 if you ran R4" callout. R3 Step 3 names the `tpi advanced msd -n <node>` command. R5 Step 2 generates the cluster token once with a clear "use the same value in Step 3" warning. R9 includes a complete MinIO setup for Velero's S3 target. R9 backup script gates each kubectl-exec behind a `kubectl get pod` existence check. R13 enumerates the .env fields you actually need to edit before launching Immich.
- **Security tightening:** All cluster services (R8 Grafana, R10 Forgejo, R11 Vaultwarden, R14 Paperless, R16 Woodpecker) now reference SealedSecret-backed `existingSecret` instead of passing passwords via `--set` (which leaks to shell history and `ps`).
- **Consistency:** R12 Nextcloud now installs into its own `nextcloud` namespace. R6 sample IngressRoute lives in `nextcloud`, not `default`. R5 Step 4 labels cube01 with `node-role.kubernetes.io/control-plane=true`. R7 Cloudflare resource uses `content` (Cloudflare provider v4) instead of the v3 `value`. R0 Hardware list specifies the 200 W minimum PSU.
- **Format wins from the migration itself:** MkDocs admonitions render correctly across themes, code blocks copy via the built-in clipboard button, navigation has search and dark mode, every page is anchor-linkable, and editing is per-`.md` instead of XML surgery.

## v12

Operational maturity pass. Added Reality of ARM64 Homelabs and Resource Budget Expectations to R0 to surface planning constraints up front. Added UPS + NUT section to R3 covering coordinated graceful shutdown across the cluster and NAS. Added Module Preferences guidance to R4 with a native-module quick reference. Added an explicit "homelab tradeoff, not production" sentence to both NFS export configurations. Consolidated all prior per-version changelogs into a single appendix (which v13 has now carried over to this file).

## v11

Platform-engineering polish. Introduced the Sealed Secrets controller for cluster-side secrets, a Namespace Strategy section (system suffix conventions, per-app namespaces, never `default`), Renovate for automated dependency updates, an optional Harbor section as a Forgejo-registry alternative, and per-runbook Verification sections so every runbook ends with a "did this actually work" checklist.

## v10

Architecture coherence pass. Converted all cluster services from Docker Compose to Helm with `values.yaml` managed in `homelab-manifests` (NAS-Docker remained the home for Immich and optionally Home Assistant). Added k3s control-plane SPOF handling via etcd snapshots, Velero for k8s-native PVC backup, and cert-manager for TLS automation. Completed the Gitea → Forgejo swap throughout R10 and R16 after the Gitea LLC fork.

## v9

Structural flow improvements. Introduced Runbook 0 (Prerequisites and Mental Model) as a foundations chapter readers should bookmark. Added a Tailscale callout to R2, the ArgoCD-before-Traefik bootstrap path in R5 (port-forward access), and an early Vaultwarden deploy tip so credentials have somewhere to live early.

## v8

Mechanical correctness fixes. Replaced the wrong nfs-server package name, tightened `/etc/exports` from world-readable to `10.0.0.0/24`, hardened Vaultwarden defaults (admin token, `SIGNUPS_ALLOWED=false`), fixed Ansible `host_key_checking` documentation, and replaced the `kubectl-set-image` CI pattern with a proper Woodpecker build → push → bump-manifest GitOps loop.

## v7

Baseline of the current runbook structure: 16 numbered runbooks covering GitHub repo layout, network segmentation, Turing Pi 2 board setup, Ansible automation, k3s cluster bootstrap, Traefik, Terraform for Cloudflare/UniFi, observability stack, backups, and the self-hosted service suite (Forgejo via later swap, Vaultwarden, Nextcloud, Immich, Paperless-ngx, Home Assistant, Woodpecker CI).
