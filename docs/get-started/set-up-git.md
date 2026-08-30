# Set up Git

Create the five repos, install secret management, and wire CI — the foundation Kubernetes (ArgoCD), Ansible, and Terraform all assume.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time estimate** | 1–2 hours initial setup |
| **Depends on** | [Prerequisites](prerequisites.md) |
| **See also** | [Repositories](../concepts/repositories.md) for the as-built state and repo rationale |
| **DevOps skills** | Git workflows, secret management, repo design |

This page walks through the day-zero bootstrap: starting from GitHub, creating the five
repos, and wiring sops, pre-commit, and ArgoCD. Once Forgejo is running, you migrate to
it and reach the [as-built state](../concepts/repositories.md#as-built-forgejo-primary-agit-prs-gitops-over-ssh).

## Choose a hosting pattern

Three viable patterns:

| Pattern | Description | Best for |
|---|---|---|
| **A: GitHub only** | Skip Forgejo entirely | Pragmatists — GitHub is honestly better |
| **B: GitHub primary + Forgejo mirror** | Start on GitHub, mirror to Forgejo later for learning | Most people |
| **C: Forgejo primary + GitHub mirror** | Self-host primary, mirror to GitHub for offsite backup | Maximum self-hosting |

**Recommendation: Pattern B for day zero.** Start on GitHub so nothing blocks on
infrastructure not yet built — this build did exactly that, then migrated to Pattern C
once Forgejo was running. GitHub keeps serving as the offsite IaC backup either way
(which [Backups](../build/backups.md) calls for anyway).

!!! tip "Offsite backup regardless of pattern"
    Whatever pattern you choose, the GitHub copy serves as offsite backup of your infrastructure-as-code. If your homelab burns down, you can rebuild from these repos. That alone justifies the pattern.

## Create the repos on GitHub { #step-1-create-the-repos-on-github }

You create two Personal Access Tokens (PATs) over the course of this page — one for your local `gh` CLI to create and push to repos, and a second (later, after Kubernetes) for ArgoCD to pull from `homelab-manifests`. Different jobs, different permission scopes.

### Token 1: local `gh` CLI and development machine

Sign in to GitHub. Go to **Settings → Developer Settings → Personal access tokens → Fine-grained tokens → Generate new token**.

- **Resource owner:** your account
- **Repository access:** All repositories (you're about to create them; you can re-scope after)
- **Repository permissions:**
    - Administration — Read and write (required to create repos)
    - Contents — Read and write (push/pull code)
    - Metadata — Read-only (auto-selected baseline)
    - Pull requests — Read and write (optional, for PR discipline)
    - Secrets — Read and write (optional, if you manage repo secrets by CLI later)

!!! warning "Fine-grained PATs need 'All repositories' at first"
    Fine-grained PATs scope to specific repos, but you're *creating* repos that don't exist yet — hence "All repositories" for now. Classic PATs with the `repo` scope sidestep this entirely; that's why a lot of guides still use them. Fine-grained is the current recommendation.

### Token 2: ArgoCD read-only deploy (create later, in Kubernetes)

Note this here so you remember to come back. When you wire ArgoCD to `homelab-manifests` in [Wire ArgoCD to homelab-manifests](#step-7-wire-argocd-to-homelab-manifests), create a second fine-grained PAT scoped only to that single repo:

- **Repository access:** Only `homelab-manifests`
- **Repository permissions:**
    - Contents — Read-only
    - Metadata — Read-only
    - Commit statuses — Read and write (only if you want ArgoCD to post deploy status back)

Save both tokens in Vaultwarden (or your bootstrap password manager if you haven't stood up Vaultwarden yet).

### Install the GitHub CLI

```bash
# macOS
brew install gh

# Ubuntu / Debian
sudo apt install -y gh

# Arch / CachyOS
sudo pacman -S github-cli

gh auth login
```

### Create all five repos

```bash
for repo in homelab-docs homelab-ansible homelab-manifests homelab-terraform homelab-secrets; do
  visibility="--private"
  [ "$repo" = "homelab-docs" ] && visibility="--public"
  gh repo create yourusername/$repo $visibility --description "Homelab: $repo"
done
```

!!! warning "Fish shell users"
    The preceding loop is bash/zsh syntax. If your shell is fish (CachyOS default for many setups), run it with `bash -c '...'` or rewrite as a fish `for ... in ... end` block.

## Set up the local workspace { #step-2-local-workspace-layout }

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
    Don't make `~/homelab/` itself a Git repo or use submodules — that defeats the per-repo isolation. To pull all five at once, define a fish function:

    ```fish
    function homelab-pull
        for repo in homelab-docs homelab-ansible homelab-manifests homelab-terraform homelab-secrets
            echo "=== $repo ==="
            git -C ~/homelab/$repo pull
        end
    end
    funcsave homelab-pull
    ```

    Bash/zsh equivalent: drop the function (minus `funcsave`) into `~/.bashrc` or `~/.zshrc` and convert `end`/`end` to `done`/`done` plus quote the loop in `;`.

## Add a universal gitignore { #step-3-universal-gitignore }

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
    Without it, the `*.tfvars` rule also excludes your sops-encrypted file, and you can never commit it. The negation explicitly allows encrypted tfvars through while blocking plaintext.

## Install pre-commit hooks { #step-4-pre-commit-hooks-catch-mistakes-before-push }

Install the pre-commit framework. This single step prevents most accidental secret commits:

```bash
# macOS
brew install pre-commit
# Linux
pip install pre-commit
# Arch / CachyOS
sudo pacman -S pre-commit
```

In each repo, create a `.pre-commit-config.yaml` file:

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

Then install the hooks into Git:

```bash
pre-commit install

# And periodically bump rev tags to the latest stable releases:
pre-commit autoupdate
```

!!! warning "Terraform hook scope"
    Include the Terraform hook block only in `homelab-terraform/.pre-commit-config.yaml`. In repos with no `.tf` files, `terraform_validate` fails and blocks every commit.

!!! tip "Stale rev tag"
    If a commit fails with `error: pathspec 'vX.Y.Z' did not match any file(s)`, the `rev:` tag is stale or wrong. Confirm the exact tag at the upstream `/releases` page, then run `pre-commit clean` to wipe the cached clones before retrying. `pre-commit autoupdate` rewrites every `rev:` in your config to the latest tag from upstream — run it in a separate commit so the version bumps are straightforward to review.

## Encrypt secrets with sops and age { #step-5-secret-management-with-sops-age }

The `homelab-secrets` repo (and any tfvars in `homelab-terraform`) gets encrypted with sops + age. Public keys are committed; private keys live only on your machine and any device that needs to decrypt.

```bash
# Install
brew install sops age
# or on Arch / CachyOS
sudo pacman -S sops age

# Create the keys directory FIRST — age-keygen does not create it for you
mkdir -p ~/.config/sops/age

# Generate an age key (one time only — back this up)
age-keygen -o ~/.config/sops/age/keys.txt

# Note the public key from the output (looks like: age1abc...xyz)
grep "public key" ~/.config/sops/age/keys.txt
```

!!! warning "Back up the age private key"
    `~/.config/sops/age/keys.txt` is the master key for every secret you encrypt. Lose it and the encrypted files are unrecoverable. Back it up to your password manager, a hardware key, or a printout in a fireproof safe. Don't commit it to Git — it's the private half of your keypair.

In any repo that holds encrypted files, create a `.sops.yaml` file at the root:

```yaml
creation_rules:
  - age: age1abc...xyz  # your PUBLIC key from the preceding step
```

!!! note "Skip path_regex"
    Some earlier guides include a `path_regex: \.enc\.(yaml|yml|json|tfvars)$` line. That regex matches the *input* filename, so encrypting `secrets.tfvars` (which doesn't contain `.enc.`) fails with `no matching creation rules found`. Drop the path_regex for a catch-all that encrypts everything in the repo — which is what you want in `homelab-secrets` and `homelab-terraform` anyway. Commit `.sops.yaml`: it contains only your public key and is safe to share.

Encrypt a file. Heredocs don't work in fish, so this is the bash version followed by a fish equivalent:

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
# Encrypt -> .enc.tfvars (the .gitignore exception from the gitignore step allows this through)
sops --encrypt secrets.tfvars > secrets.enc.tfvars
rm secrets.tfvars   # plaintext gone

# Decrypt when needed
sops --decrypt secrets.enc.tfvars > secrets.tfvars
```

The `.enc.tfvars` file IS safe to commit — it's encrypted with your age key. The plaintext `secrets.tfvars` file never touches Git (blocked by the `*.tfvars` rule from the gitignore step).

## Migrate to Forgejo { #step-6-migrating-to-forgejo-when-you-get-there }

Once Forgejo is running, you have three migration paths:

1. **Mirror from GitHub** (read-only Forgejo copy): Forgejo → **New Migration** → "This repository will be a mirror" → paste GitHub URL.
2. **Push mirror to GitHub** (Forgejo is primary): in each Forgejo repo, **Settings → Mirror Settings → Add Push Mirror**. Every push to Forgejo auto-syncs to GitHub.
3. **Full migration:** change Git remote URLs to point to Forgejo, abandon GitHub. Loses the offsite backup benefit.

**Recommended:** push mirror. Forgejo becomes primary, GitHub becomes free offsite IaC backup. This build took that path — see [As-built state](../concepts/repositories.md#as-built-forgejo-primary-agit-prs-gitops-over-ssh) for how it's wired.

## Wire ArgoCD to homelab-manifests { #step-7-wire-argocd-to-homelab-manifests }

!!! note "As-built versus this step"
    This is the day-zero path — a GitHub PAT and an ApplicationSet. Once Forgejo and the
    app-of-apps root are running, it's superseded — see
    [Pull from Forgejo with ArgoCD](../concepts/repositories.md#gitops-argocd-pulls-from-forgejo-over-ssh).

**Come back to this step after Kubernetes.** Once your repos are set up and the cluster is running, point ArgoCD at `homelab-manifests` using the read-only Token 2 from the first section:

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

Any new directory under `apps/` in `homelab-manifests` automatically becomes a deployed app. Add a folder, push, and ArgoCD deploys it.

## Follow commit conventions { #step-8-conventional-commits-pr-discipline }

Even working solo, use [Conventional Commits](https://www.conventionalcommits.org/). It makes `git log` scannable and lets tools auto-generate changelogs:

```
feat(traefik): add HTTPRoute for paperless
fix(ansible): correct cgroups regex for newer DietPi
docs(runbooks): update Immich min RAM requirement
chore(deps): bump Helm chart versions
```

Workflow even for solo work:

1. Create a branch for non-trivial changes: `git checkout -b feat/add-paperless`
2. Push the branch, open a PR against `main`.
3. Let CI run — lint, validate, plan.
4. Merge after CI passes.

!!! tip "Solo PRs build muscle memory"
    This sounds like overkill solo, but it's exactly the workflow you'd use professionally. The muscle memory transfers directly. It also catches mistakes — "that PR plan shows it deletes prod" — before they happen.

!!! note "Once you're on Forgejo"
    Item 2 of the preceding workflow becomes the AGit push — `git push origin HEAD:refs/for/main -o topic=TOPIC` creates the branch and opens the PR in one command; there's no separate "push, then click New Pull Request" step. See [Push a PR with AGit](../concepts/repositories.md#agit-pushing-a-pr-without-a-fork-or-a-branch-button).

## Verification

- [ ] Five repos exist on GitHub (`gh repo list yourusername` shows all five).
- [ ] All cloned locally under `~/homelab/`.
- [ ] `.gitignore` in each repo covers `*.tfvars` AND has `!*.enc.tfvars` exception.
- [ ] `pre-commit` installed; gitleaks scans on commit.
- [ ] `~/.config/sops/age/keys.txt` exists and is backed up.
- [ ] `.sops.yaml` committed in `homelab-secrets` and `homelab-terraform`.
- [ ] Test commit with a fake secret — verify gitleaks blocks it.
- [ ] (Day-zero) ArgoCD ApplicationSet syncing `apps/*` from `homelab-manifests` over the GitHub PAT.
- [ ] (As-built) `bootstrap/root.yaml`'s app-of-apps shows every `Application` Synced/Healthy:

    ```bash
    kubectl get application -n argocd
    ```

- [ ] (As-built) Push mirror is green — Forgejo repo → **Settings → Mirror Settings** shows a recent successful sync to GitHub.
- [ ] (As-built) AGit test PR triggers CI:

    ```bash
    git push origin HEAD:refs/for/main -o topic=test-agit
    ```

    Opens a PR and Woodpecker's `pull_request` pipeline runs — see [Woodpecker verification](../deploy/woodpecker.md#verification).
