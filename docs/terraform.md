# Terraform

Three things in your homelab are good Terraform fits: Cloudflare DNS records, UniFi network config, and a small cloud account used for learning.

!!! note "Why this runbook is retroactive"
    At this point in the guide you've already manually clicked through Cloudflare and UDM (Networking and Traefik). Terraform is therefore retroactive — you're IaC-ifying configuration you already created. That's still valuable: the next time you add a service, you'll add a Cloudflare record via `terraform apply` instead of clicking through the UI. If you want a stricter IaC-first workflow, you can move Terraform to before Traefik on a future rebuild.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 3–4 hours |
| **Runs On** | Your machine |
| **Depends On** | Traefik, Networking |

## Where Terraform Genuinely Fits

| Domain | Why it fits |
|---|---|
| Cloudflare DNS | Real API, records change as you add services |
| UniFi config | Provider exists, VLANs/firewall as code |
| Cloud practice | Cloud APIs are Terraform's natural habitat |

## Where Terraform Does NOT Fit

- Bare-metal node provisioning — use [Ansible](ansible.md)
- Kubernetes resource creation — ArgoCD watching Git is better
- Helm releases — same reason

## Install

```bash
# macOS
brew install terraform
# Arch / CachyOS
sudo pacman -S terraform

terraform version
```

!!! note "This build runs OpenTofu"
    The repo is applied with [OpenTofu](https://opentofu.org/) (`sudo pacman -S opentofu`),
    the open-source Terraform fork — the CLI is a drop-in `tofu` for `terraform`, and the
    pre-commit hooks are `tofu_fmt` / `tofu_validate`. Prose below says "Terraform" for the
    tool category; commands in the build-specific sections use `tofu`.

## Project Structure

```
homelab-terraform/
├── cloudflare/           # applied — one A record per published service
│   └── versions.tf       # providers + the remote-state backend (see below)
├── unifi/                # written, never applied (see the UniFi Module section)
└── .gitignore
```

!!! warning "Don't commit state or plaintext tfvars"
    Never commit `.tfstate`, `.tfvars` (plaintext), or anything containing API tokens. The `.gitignore` from Git covers this, with the `!*.enc.tfvars` exception allowing sops-encrypted tfvars through.

## Remote State

By default, Terraform writes state to `terraform.tfstate` in the working directory. That's local-only — every machine has a different view, and `.gitignore` blocks committing it. For a homelab this is fine if you only run Terraform from one host. For anything more, use a remote backend.

**This build uses the Garage S3 backend** — the `backend "s3"` block lives in each
module's `versions.tf` (bucket `tfstate`, one key prefix per module). State locking is
deliberately off: OpenTofu's S3-native locking needs conditional writes, which Garage
doesn't provide; with a single operator and single state writer that's acceptable.

Two reasonable options:

=== "S3 backend → Garage on NAS"
    Use the shared Garage S3 store from [Backups](backups.md) — create a dedicated `tfstate` bucket and key there (same `bucket create` / `key create` / `bucket allow` pattern). Garage already serves the S3 API on `:9000` with `s3_region = "us-east-1"` and path-style addressing, exactly what the backend config below expects.

    Then `backend.tf`:

    ```hcl
    terraform {
      backend "s3" {
        bucket                      = "tfstate"
        key                         = "homelab/cloudflare.tfstate"
        endpoints                   = { s3 = "http://10.0.20.50:9000" }  # NAS (Networking static-IP table)
        region                      = "us-east-1"
        use_path_style              = true
        skip_credentials_validation = true
        skip_metadata_api_check     = true
        skip_region_validation      = true
        skip_requesting_account_id  = true
      }
    }
    ```

=== "Terraform Cloud free tier"
    Sign up at app.terraform.io, create an organization + workspace, then:

    ```hcl
    terraform {
      cloud {
        organization = "your-org"
        workspaces { name = "homelab-cloudflare" }
      }
    }
    ```

    `terraform login` once on your machine. State lives in Terraform Cloud; their free tier handles homelab-scale usage indefinitely.

## Cloudflare Module

```hcl
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 5.0"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}

locals {
  services = [
    "nextcloud", "vault", "immich", "paperless",
    "ha", "git", "ci", "grafana", "traefik", "argocd",
  ]
}

resource "cloudflare_dns_record" "services" {
  for_each = toset(local.services)
  zone_id  = var.cloudflare_zone_id
  name     = each.value
  content  = var.traefik_ip
  type     = "A"
  proxied  = false
  ttl      = 300
}
```

!!! warning "v5 renamed the resource"
    Provider v5 renamed `cloudflare_record` → `cloudflare_dns_record` (and earlier in v4, the field `value` → `content`). If you're migrating from a v4 codebase, run `terraform apply` after upgrading the provider — v5 ships state upgraders that auto-rewrite v4 state on first plan. Always check the [provider docs](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs) against your pinned version.

## UniFi Module

!!! warning "Written, never applied"
    This build's UDM was configured by hand ([Networking](networking.md)); the `unifi/`
    module is a future codification target. Don't read its `.tf` as a description of live
    UDM config — get live facts from the controller (or a `tofu plan`), not the code.

The `paultyng/unifi` provider was archived in April 2026. The maintained drop-in successor is `ubiquiti-community/unifi` (v0.41.x schema-compatible). `filipowm/unifi` v1.x is a more aggressive refactor with schema changes — fine if you're starting fresh, but the drop-in is the safer first move.

```hcl
terraform {
  required_providers {
    unifi = {
      source  = "ubiquiti-community/unifi"
      version = "~> 0.42"   # match the pin in unifi/versions.tf
    }
  }
}

provider "unifi" {
  username       = var.unifi_username
  password       = var.unifi_password
  api_url        = "https://10.0.0.1"
  allow_insecure = true
}
```

!!! warning
    Community-maintained UniFi providers lag newer UDM features. Verify your UDM firmware version is supported before pinning a release.

## Secret Handling

Use sops + age ([Git Step 5](git.md#step-5-secret-management-with-sops-age)). This build
reads secrets **directly through the
[`carlpett/sops`](https://github.com/carlpett/terraform-provider-sops) provider**: each
module keeps a `secrets.enc.yaml` (ciphertext, safe to commit) and a `data "sops_file"`
block decrypts it in memory at plan/apply time — plaintext never lands on disk. Edit in
place with `sops cloudflare/secrets.enc.yaml`. (Encrypted `*.enc.tfvars` passed via
`-var-file` are the older fallback pattern; the `.gitignore` allows them through.)

## Workflow

```bash
cd homelab-terraform/cloudflare

tofu init          # backend: Garage S3 on the NAS (bucket `tfstate`)
tofu plan          # the sops provider decrypts secrets.enc.yaml in memory
tofu apply
```

No decrypt or cleanup steps — the sops provider handles secrets at run time, and the
pre-commit hooks run `tofu fmt` / `tofu validate` on every commit.

!!! tip
    The single most impressive resume item from this runbook: "managed bare-metal network infrastructure (UniFi VLANs, firewall rules, DHCP) and DNS as code using Terraform, with CI-driven apply via Woodpecker."

## Verification

- [ ] Drift check: after the first apply, `plan` returns zero changes:

    ```bash
    cd homelab-terraform/cloudflare && tofu plan
    # Expected: 'No changes. Your infrastructure matches the configuration.'
    ```

- [ ] Cloudflare zone reflects Terraform state: visit dash.cloudflare.com — the records for `vault` / `nextcloud` / `git` / etc. all match `var.traefik_ip`.
- [ ] If you've wired Woodpecker CI for Terraform: pushing a tfvars change triggers a `terraform plan` job whose output appears in the Woodpecker UI.
