# Repositories

How this build's Git repos are structured, hosted, and kept secret-free.

| | |
|---|---|
| **Repos** | 5 — docs, ansible, manifests, terraform, secrets |
| **Primary host** | Forgejo — git.yourdomain.com (deploy and SSO details: [Forgejo](../deploy/forgejo.md)) |
| **GitHub's role** | Push mirror only — offsite IaC backup, not where you push or open PRs |
| **PR workflow** | AGit — `git push origin HEAD:refs/for/main -o topic=TOPIC` |
| **CI gate** | Woodpecker's `pull_request` pipeline runs on the AGit PR before merge — see [Woodpecker](../deploy/woodpecker.md) |
| **ArgoCD wiring** | App-of-apps over SSH — one `Application` per component, reconciled by `bootstrap/root.yaml` — see the `homelab-manifests` README |
| **Setup** | [Set up Git](../get-started/set-up-git.md) |

## Five separate repos

Each repo has a distinct purpose, security boundary, and consumer:

| Repo | Purpose | Consumer |
|---|---|---|
| `homelab-docs` | Runbooks, diagrams, decisions log | You (humans) |
| `homelab-ansible` | OS provisioning playbooks | Your machine |
| `homelab-manifests` | k3s YAML, Helm values, HTTPRoutes | ArgoCD |
| `homelab-terraform` | Cloudflare DNS, UniFi config, cloud practice | Your machine → Woodpecker |
| `homelab-secrets` | Encrypted secrets (sops/age) — **PRIVATE** | Your machine (with sops) |

!!! note "Two secrets systems, split by layer"
    sops + age encrypts what **your machine** reads: Ansible and Terraform secrets in
    their own repos, plus `homelab-secrets`' day-zero backup of the Sealed Secrets
    signing key. Sealed Secrets encrypts what the **cluster** reads: the SealedSecret
    ciphertext ArgoCD applies, which is committed in `homelab-manifests`, not here — the
    Sealed Secrets controller never reads `homelab-secrets`. See Kubernetes' [Two-layer
    secrets discipline](../build/kubernetes.md#step-13-encrypt-your-first-secret) for the
    mechanics.

### Why separate repos

- ArgoCD watches `homelab-manifests` — mixing in Ansible or Terraform confuses the controller.
- Different access controls: `homelab-secrets` is private; `homelab-docs` can be public.
- Different CI pipelines: Terraform repo runs `terraform plan`, manifests repo runs YAML lint.
- Cleaner blame and history when each repo has one concern.

## As-built state { #as-built-forgejo-primary-agit-prs-gitops-over-ssh }

This build runs **Pattern C** from the hosting-pattern table in [Set up Git](../get-started/set-up-git.md#choose-a-hosting-pattern): Forgejo, self-hosted at git.yourdomain.com, is primary. GitHub holds a **push mirror** — free offsite backup of the infrastructure-as-code — and is never pushed to or opened a PR against directly.

### Push a PR with AGit { #agit-pushing-a-pr-without-a-fork-or-a-branch-button }

Forgejo speaks Gitea's **AGit** flow: instead of pushing a branch and clicking **New Pull
Request** in a web UI, one `git push` both creates the branch server-side and opens the
PR:

```bash
git push origin HEAD:refs/for/main -o topic=add-paperless
```

- `refs/for/main` — targets a PR against `main`; it doesn't create a real branch named
  `for/main`.
- `-o topic=TOPIC` — names the PR. Forgejo groups pushes sharing a topic into the same
  PR, so re-running the command after fixups updates the existing PR instead of opening
  a new one.

The Forgejo webhook (configured in [Woodpecker](../deploy/woodpecker.md)) fires a `pull_request`
event on the push, so the lint and validation gate runs **before** merge — the same
guarantee GitHub Actions gave, now self-hosted.

!!! warning "A different topic opens a duplicate PR"
    Re-running the AGit push with a **different** `-o topic=` value opens a second PR
    instead of updating the first — Forgejo keys the PR to the topic string. Reuse the
    exact same topic for every push in one review cycle.

### Pull from Forgejo with ArgoCD { #gitops-argocd-pulls-from-forgejo-over-ssh }

Once Kubernetes and Forgejo are both up, ArgoCD stops needing a GitHub PAT. It watches
`homelab-manifests` over **SSH**, authenticated with a repo credential committed as a
SealedSecret (`infrastructure/argocd/manifests/`), and discovery is the **app-of-apps**
pattern — `bootstrap/root.yaml` reconciles one `Application` file per component, not the
single ApplicationSet the day-zero bootstrap uses. Full wiring detail lives in the
`homelab-manifests` README; this page only owns repo structure and hosting.
