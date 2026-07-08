# Git Foundation

How to structure your homelab project across Git repositories, where to host them, and how to keep secrets out of them. Sets up the foundation that Kubernetes (ArgoCD), Ansible, and Terraform all assume.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | 1–2 hours initial setup |
| **Hosts** | Forgejo primary → GitHub push mirror (this build's end-state; the runbook starts on GitHub and migrates in Step 6) |
| **DevOps Skills** | Git workflows, secret management, repo design |

## Repo Structure: Five Separate Repos

Resist the urge to put everything in one mono-repo. Each repo below has a distinct purpose, security boundary, and consumer:

| Repo | Purpose | Consumer |
|---|---|---|
| `homelab-docs` | These runbooks, diagrams, decisions log | You (humans) |
| `homelab-ansible` | OS provisioning playbooks | Your machine |
| `homelab-manifests` | k3s YAML, Helm values, HTTPRoutes | ArgoCD |
| `homelab-terraform` | Cloudflare DNS, UniFi config, cloud practice | Your machine → Woodpecker |
| `homelab-secrets` | Encrypted secrets (sops/age) — **PRIVATE** | Your machine (sops); Sealed Secrets controller for in-cluster manifests |

**Why separate repos:**

- ArgoCD watches `homelab-manifests` — mixing in Ansible or Terraform confuses the controller
- Different access controls: `homelab-secrets` is private; `homelab-docs` can be public if you want to share
- Different CI pipelines: Terraform repo runs `terraform plan`, manifests repo runs YAML lint
- Cleaner blame/history when each repo has one concern

## GitHub vs Forgejo: The Decision

Three viable patterns:

| Pattern | Description | Best For |
|---|---|---|
| **A: GitHub only** | Skip Forgejo entirely | Pragmatists — GitHub is honestly better |
| **B: GitHub primary + Forgejo mirror** | Start on GitHub, mirror to Forgejo later for learning | Most people, **recommended** |
| **C: Forgejo primary + GitHub mirror** | Self-host primary, mirror to GitHub for offsite backup | Maximum self-hosting |

**Recommendation: Pattern B.** Start on GitHub now so nothing blocks on infrastructure not yet built. Migrate later if you want the Forgejo experience, with GitHub as your offsite-IaC-backup (which Backups calls for anyway).

!!! tip
    Whatever pattern you choose, the GitHub copy serves as offsite backup of your infrastructure-as-code. If your homelab burns down, you can rebuild it from these repos. That alone justifies the pattern.

## Step 1: Create the Repos on GitHub

You will create two Personal Access Tokens (PATs) over the course of this runbook — one for your local `gh` CLI to create and push to repos, and a second (later, after Kubernetes) for ArgoCD to pull from `homelab-manifests`. Different jobs, different permission scopes.

### Token 1: Local `gh` CLI / development machine

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

### Token 2: ArgoCD / read-only deploy (create later, in Kubernetes)

Note this here so you remember to come back. When you wire ArgoCD to `homelab-manifests` in Step 7, create a second fine-grained PAT scoped only to that single repo:

- **Repository access:** Only `homelab-manifests`
- **Repository permissions:**
    - Contents — Read-only
    - Metadata — Read-only
    - Commit statuses — Read and write (only if you want ArgoCD to post deploy status back)

Save both tokens in Vaultwarden (or 1Password if you haven't stood up Vaultwarden yet).

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
    The loop above is bash/zsh syntax. If your shell is fish (CachyOS default for many setups), run it via `bash -c '...'` or rewrite as a fish `for ... in ... end` block. Fish heredocs and `&&` chaining inside the loop body do not work like bash.

## Step 2: Local Workspace Layout

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

## Step 3: Universal .gitignore

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

## Step 4: Pre-Commit Hooks (Catch Mistakes Before Push)

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

## Step 5: Secret Management with sops + age

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

## Step 6: Migrating to Forgejo (When You Get There)

Once Forgejo is running, you have three migration paths:

1. **Mirror from GitHub** (read-only Forgejo copy): Forgejo → New Migration → "This repository will be a mirror" → paste GitHub URL.
2. **Push mirror to GitHub** (Forgejo is primary): In each Forgejo repo, Settings → Mirror Settings → Add Push Mirror. Every push to Forgejo auto-syncs to GitHub.
3. **Full migration:** change git remote URLs to point to Forgejo, abandon GitHub. Loses the offsite backup benefit.

**Recommended:** push mirror. Forgejo becomes primary, GitHub becomes free offsite IaC backup.

## Step 7: Wire ArgoCD to homelab-manifests

!!! note "What this build runs today"
    After the Forgejo migration (Step 6), ArgoCD watches `homelab-manifests` **on Forgejo
    over SSH** (a repo credential committed as a SealedSecret in
    `infrastructure/argocd/manifests/`), not GitHub — and the discovery model is the
    **app-of-apps** pattern (`bootstrap/root.yaml` reconciling one Application file per
    component; see the `homelab-manifests` README), not the ApplicationSet below. The
    GitHub + read-only-PAT wiring remains the correct day-zero path when nothing
    self-hosted exists yet.

**Come back to this step after Kubernetes.** Once your repos are set up and the cluster is running, point ArgoCD at `homelab-manifests` using the read-only Token 2 from Step 1:

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

## Step 8: Conventional Commits & PR Discipline

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

## Verification Checklist

- [ ] Five repos exist on GitHub (`gh repo list yourusername` shows all five)
- [ ] All cloned locally under `~/homelab/`
- [ ] `.gitignore` in each repo covers `*.tfvars` AND has `!*.enc.tfvars` exception
- [ ] `pre-commit` installed; gitleaks scans on commit
- [ ] `~/.config/sops/age/keys.txt` exists and is backed up
- [ ] `.sops.yaml` committed in `homelab-secrets` and `homelab-terraform`
- [ ] Test commit with a fake secret — verify gitleaks blocks it
- [ ] (After Kubernetes) ArgoCD ApplicationSet syncing from `homelab-manifests`
- [ ] (After Forgejo) Push mirror configured Forgejo → GitHub
