# Version History

The doc lineage. Body content always reflects the current state; this appendix captures the shape of each release.

## v21 (current)

Documentation style rewrite. A repo-root **`STYLE.md`** codifies the page anatomy every runbook now follows: status banner → one-sentence description → **facts table** (the fields a returning operator looks up — URL, namespace, storage, auth, dependencies) → body prose → `## Verification`. Prose is reserved for *why*; anything with a value — a port, a schedule, a version — belongs in a table or code block instead. Admonitions carry fixed semantics: `!!! danger` for irreversible loss, `!!! warning` for debugging-cost traps, `!!! note` for context, `!!! tip` for optional shortcuts — never stacked more than two deep. Long reference-only material (full systemd units, values files, command transcripts) collapses behind `??? note` / `??? example` so skimming doesn't scroll past it. `docs/backups.md` and `docs/forgejo.md` were rewritten first as the two reference implementations — infrastructure chapter and service runbook — and the rest of the guide was converted to match, page by page. `docs/index.md` gets the one showcase exception the guide carves out: at-a-glance tables for the hardware build, the five-repo split, and the App Catalog, in place of the operational formatting every other page keeps. The **Planned & Future** stubs (Arr Stack, BookStack, Syncthing, RustDesk, Reactive Resume, Ollama) were brought in line with the same anatomy. Separately, the **`mkdocs-redirects`** plugin was dropped: it pulled an unpinned fork of MkDocs by a departed maintainer into the dependency tree just to serve retired `/NN-name/` redirects on a site that was never published, so the plugin, its pin, and the redirect map are gone — `mkdocs build --strict` now runs against three pins instead of four.

## v20

Documentation reorganization — **runbook numbers retired**. The 30 numbered files (`00-prerequisites.md` … `29-miniflux.md`) were renamed to slug-based filenames (`prerequisites.md`, `vaultwarden.md`, `woodpecker.md`, …) and their `# Runbook N — Title` headings became plain titles; the MkDocs nav now carries the ordering. Old `/NN-name/` URLs keep resolving via the `mkdocs-redirects` plugin, so external bookmarks don't 404. The **`Rxx` / `Runbook N` shorthand is retired** in favour of app names — in prose and in the `Depends On` headers alike — because the number cross-references had drifted (the deployment-order list cited runbook numbers that no longer matched). The nav gained a **Platform Services** tier (Vaultwarden, Authelia + lldap, ntfy, NAS PostgreSQL, Forgejo, Woodpecker, AdGuard) and a **Planned & Future** section, and every app/service page now opens with a **Status** banner (Live / Planned / Shelved / Retired). The App Catalog is split into Live / Planned / Retired, and the stale catalog list (FreshRSS, double-listed Vikunja) was reconciled against what's actually deployed. Page content is otherwise unchanged. *(The appendix had lagged at v15 while page bodies advanced through the incremental app-runbook additions of v16–v19; this entry brings it current.)*

## v15

R3/R4 alignment to the first real deployment on DietPi. Non-root throughout — Ansible and SSH use the `dietpi` user with `sudo`, not root, and R4 adds the post-key SSH hardening (disable root login + password auth). Secrets move from Ansible Vault to sops + age via the `community.sops` lookup, unifying with the `homelab-secrets` mechanism and R5's two-layer model. R4 bootstrap drops the `hostname` and netplan tasks: DietPi owns node identity (hostname + static IP from `dietpi.txt`), has no dbus for `ansible.builtin.hostname`, and ships no netplan. The cmdline path is corrected to `/boot/firmware/cmdline.txt`, `stdout_callback` becomes `default` + `callback_result_format: yaml` (the standalone `yaml` callback was removed in community.general 12), and the NFS play is tagged `nfs` so the cluster can come up before the SATA SSD is installed.

## v14

Structural reorder: Vaultwarden moves to R7 (right after Traefik), shifting Terraform, Observability, Backups, and Forgejo down by one (now R8, R9, R10, R11). Rationale: Vaultwarden's only dependency is Traefik, so it can come up immediately after R6 — once it's running it serves as the password manager for every later runbook's credentials, removing the "save to Bitwarden, migrate later" workaround that v13 documented. R12-R16 are unaffected. Older `R7`/`R8`/`R9`/`R10`/`R11` references in the v13 entry below reflect the **old** numbering for context.

## v13

Format migration to Markdown + MkDocs Material. The runbooks are now authored as one `.md` file per runbook under `docs/`, rendered as a static site via MkDocs Material, and render natively in GitHub as a fallback view. Folded in the 27 read-through findings from the v12 dry run:

- **Critical fixes:** Numbered-list continuation bug is gone (Markdown restarts numbering per section). Fish-function helper renders as a proper fenced code block. Woodpecker sample pipeline no longer mounts `/var/run/docker.sock` — replaced with kaniko, matching the doc's own "kubernetes backend doesn't need the socket" claim. Token variable in the bump-manifest step is now `${FORGEJO_TOKEN}` to match the `forgejo_token` secret. R6 Step 3 has a complete title ("Create the Cloudflare Token Secret and Install Traefik").
- **High-impact additions:** New Step 0 in R5 covering `kubectl` install + kubeconfig copy to your machine. R5 opens with an explicit "skip Steps 1–4 if you ran R4" callout. R3 Step 3 names the `tpi advanced msd -n <node>` command. R5 Step 2 generates the cluster token once with a clear "use the same value in Step 3" warning. R9 includes a complete MinIO setup for Velero's S3 target. R9 backup script gates each kubectl-exec behind a `kubectl get pod` existence check. R13 enumerates the .env fields you actually need to edit before launching Immich.
- **Security tightening:** All cluster services (R8 Grafana, R10 Forgejo, R11 Vaultwarden, R14 Paperless, R16 Woodpecker) now reference SealedSecret-backed `existingSecret` instead of passing passwords via `--set` (which leaks to shell history and `ps`).
- **Consistency:** R12 Nextcloud now installs into its own `nextcloud` namespace. R6 sample IngressRoute lives in `nextcloud`, not `default`. R5 Step 4 labels ruby with `node-role.kubernetes.io/control-plane=true`. R7 Cloudflare resource uses `content` (Cloudflare provider v4) instead of the v3 `value`. R0 Hardware list specifies the 200 W minimum PSU.
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
