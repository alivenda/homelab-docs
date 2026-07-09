# Git Foundation

How this build's Git repos are structured, hosted, and kept secret-free — the foundation Kubernetes (ArgoCD), Ansible, and Terraform all assume.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | 1–2 hours initial setup |
| **Hosts** | Forgejo primary (`git.yourdomain.com`) → GitHub push mirror — see [As-built](#as-built-forgejo-primary-agit-prs-gitops-over-ssh) |
| **PR Workflow** | AGit — `git push origin HEAD:refs/for/main -o topic=<short-topic>` |
| **DevOps Skills** | Git workflows, secret management, repo design |

## Repo Structure: Five Separate Repos

Resist the urge to put everything in one mono-repo. Each repo below has a distinct purpose, security boundary, and consumer:

| Repo | Purpose | Consumer |
|---|---|---|
| `homelab-docs` | These runbooks, diagrams, decisions log | You (humans) |
| `homelab-ansible` | OS provisioning playbooks | Your machine |
| `homelab-manifests` | k3s YAML, Helm values, HTTPRoutes | ArgoCD |
| `homelab-terraform` | Cloudflare DNS, UniFi config, cloud practice | Your machine → Woodpecker |
| `homelab-secrets` | Encrypted secrets (sops/age) — **PRIVATE** | Your machine (via sops) |

!!! note "Two secrets systems, split by layer"
    sops + age encrypts what **your machine** reads: Ansible and Terraform secrets in
    their own repos, plus `homelab-secrets`' day-zero backup of the Sealed Secrets
    signing key. Sealed Secrets encrypts what the **cluster** reads: the SealedSecret
    ciphertext ArgoCD applies, which is committed in `homelab-manifests`, not here — the
    Sealed Secrets controller never reads `homelab-secrets`. See Kubernetes' [Two-layer
    secrets discipline](kubernetes.md#step-13-encrypt-your-first-secret) for the
    mechanics.

**Why separate repos:**

- ArgoCD watches `homelab-manifests` — mixing in Ansible or Terraform confuses the controller
- Different access controls: `homelab-secrets` is private; `homelab-docs` can be public if you want to share
- Different CI pipelines: Terraform repo runs `terraform plan`, manifests repo runs YAML lint
- Cleaner blame/history when each repo has one concern

## As-built: Forgejo primary, AGit PRs, GitOps over SSH

This build runs **Pattern C** from the table below: Forgejo, self-hosted at
`git.yourdomain.com`, is primary. GitHub holds a **push mirror** — free offsite backup
of the infrastructure-as-code — and is never pushed to or opened a PR against directly.

| | |
|---|---|
| **Primary host** | Forgejo — `git.yourdomain.com` (deploy + SSO details: [Forgejo](forgejo.md)) |
| **GitHub's role** | Push mirror only — offsite IaC backup, not where you push or open PRs |
| **PR workflow** | AGit — `git push origin HEAD:refs/for/main -o topic=<short-topic>` |
| **CI gate** | Woodpecker's `pull_request` pipeline runs on the AGit PR before merge — see [Woodpecker](woodpecker.md) |
| **ArgoCD wiring** | App-of-apps over SSH — one `Application` per component, reconciled by `bootstrap/root.yaml` — see the `homelab-manifests` README |

### AGit: pushing a PR without a fork or a branch button

Forgejo speaks Gitea's **AGit** flow: instead of pushing a branch and clicking "New Pull
Request" in a web UI, one `git push` both creates the branch server-side and opens the
PR:

```bash
git push origin HEAD:refs/for/main -o topic=add-paperless
```

- `refs/for/main` — targets a PR against `main`; it does not create a real branch named
  `for/main`.
- `-o topic=<name>` — names the PR. Forgejo groups pushes sharing a topic into the same
  PR, so re-running the command after fixups updates the existing PR instead of opening
  a new one.

The Forgejo webhook (configured in [Woodpecker](woodpecker.md)) fires a `pull_request`
event on the push, so the lint/validation gate runs **before** merge — the same
guarantee GitHub Actions gave, now self-hosted.

!!! warning "A different topic opens a duplicate PR, it doesn't update the old one"
    Re-running the AGit push with a **different** `-o topic=` value opens a second PR
    instead of updating the first — Forgejo keys the PR to the topic string. Reuse the
    exact same topic for every push in one review cycle.

### GitOps: ArgoCD pulls from Forgejo over SSH

Once Kubernetes and Forgejo are both up, ArgoCD stops needing a GitHub PAT. It watches
`homelab-manifests` over **SSH**, authenticated with a repo credential committed as a
SealedSecret (`infrastructure/argocd/manifests/`), and discovery is the **app-of-apps**
pattern — `bootstrap/root.yaml` reconciles one `Application` file per component, not the
single ApplicationSet the day-zero bootstrap below uses. Full wiring detail lives in the
`homelab-manifests` README; this page only owns repo structure and hosting.

## Day-zero bootstrap: starting from GitHub

Before Forgejo exists, GitHub is where you create these repos and do your early commits.
The steps below are that from-scratch path — work through them in order, then
[migrate to Forgejo](#step-6-migrating-to-forgejo-when-you-get-there) (Step 6) once it's
running to reach the as-built state above.

### GitHub vs Forgejo: The Decision

Three viable patterns:

| Pattern | Description | Best For |
|---|---|---|
| **A: GitHub only** | Skip Forgejo entirely | Pragmatists — GitHub is honestly better |
| **B: GitHub primary + Forgejo mirror** | Start on GitHub, mirror to Forgejo later for learning | Most people, **recommended** |
| **C: Forgejo primary + GitHub mirror** | Self-host primary, mirror to GitHub for offsite backup | Maximum self-hosting |

**Recommendation: Pattern B, for day zero.** Start on GitHub now so nothing blocks on
infrastructure not yet built — this build did exactly that, then migrated to Pattern C
once Forgejo was running (Step 6). GitHub keeps serving as the offsite IaC backup either
way (which [Backups](backups.md) calls for anyway) — see
[As-built](#as-built-forgejo-primary-agit-prs-gitops-over-ssh) above for how it's wired
today.

!!! tip
    Whatever pattern you choose, the GitHub copy serves as offsite backup of your infrastructure-as-code. If your homelab burns down, you can rebuild it from these repos. That alone justifies the pattern.

### Step 1: Create the Repos on GitHub

You will create two Personal Access Tokens (PATs) over the course of this runbook — one for your local `gh` CLI to create and push to repos, and a second (later, after Kubernetes) for ArgoCD to pull from `homelab-manifests`. Different jobs, different permission scopes.

#### Token 1: Local `gh` CLI / development machine

Sign in to GitHub. Go to **Settings → Developer Settings → Personal access tokens → Fine-grained tokens → Generate new token**.

- **Resource owner:** your account
- **Repository access:** All repositories (you're about to create them; you can re-scope after)
- **Repository permissions:**
    - Administration — Read and write (required to create repos)
    - Contents — Read and write (push/pull code)
    - Metadata — Read-only (auto-selected baseline)
    - Pull requests — Read and write (optional, for the PR discipline in Step 8)
    - Secrets — Read and write (optional, if you manage repo secrets via CLI later)

!!! warning
    Fine-grained PATs scope to specific repos, but you are *creating* repos that don't exist yet — hence "All repositories" for now. Classic PATs with the `repo` scope sidestep this entirely; that's why a lot of guides still use them. Fine-grained is the modern recommendation, so we use it here.

#### Token 2: ArgoCD / read-only deploy (create later, in Kubernetes)

Note this here so you remember to come back. When you wire ArgoCD to `homelab-manifests` in Step 7, create a second fine-grained PAT scoped only to that single repo:

- **Repository access:** Only `homelab-manifests`
- **Repository permissions:**
    - Contents — Read-only
    - Metadata — Read-only
    - Commit statuses — Read and write (only if you want ArgoCD to post deploy status back)

Save both tokens in Vaultwarden (or 1Password if you haven't stood up Vaultwarden yet).

#### Install the GitHub CLI

```bash
# macOS
brew install gh

# Ubuntu / Debian
sudo apt install -y gh

# Arch / CachyOS
sudo pacman -S github-cli

gh auth login
```

#### Create all five repos

```bash
for repo in homelab-docs homelab-ansible homelab-manifests homelab-terraform homelab-secrets; do
  visibility="--private"
  [ "$repo" = "homelab-docs" ] && visibility="--public"
  gh repo create yourusername/$repo $visibility --description "Homelab: $repo"
done
```

!!! warning "Fish shell users"
    The loop above is bash/zsh syntax. If your shell is fish (CachyOS default for many setups), run it via `bash -c '...'` or rewrite as a fish `for ... in ... end` block. Fish heredocs and `&&` chaining inside the loop body do not work like bash.

### Step 2: Local Workspace Layout

```
~/homelab/
├── homelab-docs/
├── homelab-ansible/
├── homelab-manifests/
├── homelab-terraform/
└── homelab-secrets/
```

Clone them all:

```bash
mkdir ~/homelab && cd ~/homelab
for repo in homelab-docs homelab-ansible homelab-manifests homelab-terraform homelab-secrets; do
  gh repo clone yourusername/$repo
done
```

!!! tip "Pull-all-five helper (fish)"
    Do **not** make `~/homelab/` itself a Git repo or use submodules — that defeats the per-repo isolation. To pull all five at once, define a fish function:

    ```fish
    function homelab-pull
        for repo in homelab-docs homelab-ansible homelab-manifests homelab-terraform homelab-secrets
            echo "=== $repo ==="
            git -C ~/homelab/$repo pull
        end
    end
    funcsave homelab-pull
    ```

    Bash/zsh equivalent: drop the above (minus `funcsave`) into `~/.bashrc` or `~/.zshrc` and convert `end`/`end` to `done`/`done` plus quote the loop in `;`.

### Step 3: Universal .gitignore

Drop this in every repo as a starting point:

```gitignore
# Terraform
*.tfvars
!*.enc.tfvars
*.tfstate
*.tfstate.*
.terraform/

# Ansible
*.retry
.vault-password
vault_password*

# Secrets / keys
*.pem
*.key
id_rsa*
id_ed25519*
kubeconfig
*.kubeconfig

# OS / editor
.DS_Store
.idea/
.vscode/

# Backups / temp
*.bak
*.swp
*.log
```

!!! note "The `!*.enc.tfvars` negation is load-bearing"
    Without it, the `*.tfvars` rule would also exclude your sops-encrypted file, and you would never be able to commit it. The negation explicitly allows encrypted tfvars through while blocking plaintext.

### Step 4: Pre-Commit Hooks (Catch Mistakes Before Push)

Install the pre-commit framework. This single step prevents 95% of accidental secret commits:

```bash
# macOS
brew install pre-commit
# Linux
pip install pre-commit
# Arch / CachyOS
sudo pacman -S pre-commit
```

In each repo, create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: detect-private-key

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.1
    hooks:
      - id: gitleaks

  # Terraform-specific — ONLY include this block in homelab-terraform
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.105.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
```

Then install the hooks into git:

```bash
pre-commit install

# And periodically bump rev tags to the latest stable releases:
pre-commit autoupdate
```

!!! warning "Terraform hook scope"
    The Terraform hook block above must only be included in `homelab-terraform/.pre-commit-config.yaml`. In repos with no `.tf` files, `terraform_validate` will fail and block every commit.

!!! tip "Stale `rev:` tag"
    If a commit fails with `error: pathspec 'vX.Y.Z' did not match any file(s)`, the `rev:` tag is stale or wrong. Confirm the exact tag at the upstream `/releases` page, then run `pre-commit clean` to wipe the cached clones before retrying. `pre-commit autoupdate` rewrites every `rev:` in your config to the latest tag from upstream — run it in a separate commit so the version bumps are easy to review.

### Step 5: Secret Management with sops + age

The `homelab-secrets` repo (and any tfvars in `homelab-terraform`) get encrypted with sops + age. Public keys are committed; private keys live only on your machine and any device that needs to decrypt.

```bash
# Install
brew install sops age
# or on Arch / CachyOS
sudo pacman -S sops age

# Create the keys directory FIRST — age-keygen will NOT create it for you
mkdir -p ~/.config/sops/age

# Generate an age key (one time only — back this up)
age-keygen -o ~/.config/sops/age/keys.txt

# Note the public key from the output (looks like: age1abc...xyz)
grep "public key" ~/.config/sops/age/keys.txt
```

!!! warning
    `~/.config/sops/age/keys.txt` is the master key for every secret you encrypt. Lose it and the encrypted files are unrecoverable. Back it up to Vaultwarden, a hardware key, or a printout in a fireproof safe. Do **not** commit it to Git — it is the private half of your keypair.

In any repo that will hold encrypted files, create `.sops.yaml` at the root:

```yaml
creation_rules:
  - age: age1abc...xyz  # your PUBLIC key from above
```

!!! note "Skip `path_regex`"
    Some earlier guides include a `path_regex: \.enc\.(yaml|yml|json|tfvars)$` line. That regex matches the *input* filename, so encrypting `secrets.tfvars` (which does not contain `.enc.`) fails with `no matching creation rules found`. Drop the path_regex for a simple catch-all that encrypts everything in the repo — which is what you want in `homelab-secrets` and `homelab-terraform` anyway. Commit `.sops.yaml`: it only contains your public key and is safe to share.

Encrypt a file. Heredocs do not work in fish, so this is the bash version followed by a fish equivalent:

=== "bash / zsh"
    ```bash
    cat > secrets.tfvars << 'EOF'
    cloudflare_api_token = "abc123"
    EOF
    ```

=== "fish"
    ```fish
    echo 'cloudflare_api_token = "abc123"' > secrets.tfvars
    ```

Then encrypt:

```bash
# Encrypt -> .enc.tfvars (the .gitignore exception from Step 3 allows this through)
sops --encrypt secrets.tfvars > secrets.enc.tfvars
rm secrets.tfvars   # plaintext gone

# Decrypt when needed
sops --decrypt secrets.enc.tfvars > secrets.tfvars
```

The `.enc.tfvars` file IS safe to commit — it's encrypted with your age key. The plaintext `secrets.tfvars` never touches Git (blocked by the `*.tfvars` rule from Step 3).

### Step 6: Migrating to Forgejo (When You Get There)

Once Forgejo is running, you have three migration paths:

1. **Mirror from GitHub** (read-only Forgejo copy): Forgejo → New Migration → "This repository will be a mirror" → paste GitHub URL.
2. **Push mirror to GitHub** (Forgejo is primary): In each Forgejo repo, Settings → Mirror Settings → Add Push Mirror. Every push to Forgejo auto-syncs to GitHub.
3. **Full migration:** change git remote URLs to point to Forgejo, abandon GitHub. Loses the offsite backup benefit.

**Recommended:** push mirror. Forgejo becomes primary, GitHub becomes free offsite IaC backup. This build took that path — see [As-built](#as-built-forgejo-primary-agit-prs-gitops-over-ssh) above for how it's wired today.

### Step 7: Wire ArgoCD to homelab-manifests

!!! note "As-built vs. this step"
    This is the day-zero path — a GitHub PAT and an ApplicationSet. Once Forgejo and the
    app-of-apps root are running, it's superseded — see
    [As-built: GitOps](#gitops-argocd-pulls-from-forgejo-over-ssh) above.

**Come back to this step after Kubernetes.** Once your repos are set up and the cluster is running, point ArgoCD at `homelab-manifests` using the read-only Token 2 from Step 1:

??? example "Day-zero: read-only PAT + ApplicationSet"

    ```bash
    # Add the repo to ArgoCD using the read-only PAT
    argocd repo add https://github.com/yourusername/homelab-manifests.git \
      --username yourusername \
      --password <GITHUB_READONLY_PAT>

    # Create an ApplicationSet that auto-discovers apps in apps/*
    cat << 'EOF' | kubectl apply -f -
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: homelab
      namespace: argocd
    spec:
      generators:
        - git:
            repoURL: https://github.com/yourusername/homelab-manifests.git
            revision: main
            directories:
              - path: apps/*
      template:
        metadata:
          name: '{{path.basename}}'
        spec:
          project: default
          source:
            repoURL: https://github.com/yourusername/homelab-manifests.git
            targetRevision: main
            path: '{{path}}'
          destination:
            server: https://kubernetes.default.svc
            namespace: '{{path.basename}}'
          syncPolicy:
            automated:
              prune: true
              selfHeal: true
            syncOptions:
              - CreateNamespace=true
    EOF
    ```

Now any new directory under `apps/` in `homelab-manifests` automatically becomes a deployed app. Add a folder, push, ArgoCD deploys it.

### Step 8: Conventional Commits & PR Discipline

Even working solo, use [Conventional Commits](https://www.conventionalcommits.org/). It makes `git log` scannable and lets tools auto-generate changelogs:

```
feat(traefik): add HTTPRoute for paperless
fix(ansible): correct cgroups regex for newer DietPi
docs(runbooks): update Immich min RAM requirement
chore(deps): bump Helm chart versions
```

Workflow even for solo work:

1. Create a branch for non-trivial changes: `git checkout -b feat/add-paperless`
2. Push the branch, open a PR against `main`
3. Let CI run — lint, validate, plan
4. Merge after CI passes

!!! tip
    This sounds like overkill solo, but it's exactly the workflow you'd use professionally. The muscle memory transfers directly. It also catches mistakes — "oh that PR plan shows it would delete prod" — before they happen.

!!! note "Once you're on Forgejo"
    Item 2 of the workflow above becomes the AGit push — `git push origin HEAD:refs/for/main -o topic=<short-topic>` creates the branch and opens the PR in one command; there's no separate "push, then click New Pull Request" step. See [As-built: AGit](#agit-pushing-a-pr-without-a-fork-or-a-branch-button) above.

## Verification

- [ ] Five repos exist on GitHub (`gh repo list yourusername` shows all five)
- [ ] All cloned locally under `~/homelab/`
- [ ] `.gitignore` in each repo covers `*.tfvars` AND has `!*.enc.tfvars` exception
- [ ] `pre-commit` installed; gitleaks scans on commit
- [ ] `~/.config/sops/age/keys.txt` exists and is backed up
- [ ] `.sops.yaml` committed in `homelab-secrets` and `homelab-terraform`
- [ ] Test commit with a fake secret — verify gitleaks blocks it
- [ ] (Day-zero) ArgoCD ApplicationSet syncing `apps/*` from `homelab-manifests` over the GitHub PAT
- [ ] (As-built) `bootstrap/root.yaml`'s app-of-apps shows every `Application` `Synced/Healthy`:

    ```bash
    kubectl get application -n argocd
    ```

- [ ] (As-built) Push mirror is green — Forgejo repo → **Settings → Mirror Settings** shows a recent successful sync to GitHub
- [ ] (As-built) AGit test PR triggers CI:

    ```bash
    git push origin HEAD:refs/for/main -o topic=test-agit
    ```

    opens a PR and Woodpecker's `pull_request` pipeline runs — see [Woodpecker Verification](woodpecker.md#verification)
