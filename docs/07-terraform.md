# Runbook 7: Terraform for the Homelab

Three things in your homelab are good Terraform fits: Cloudflare DNS records, UniFi network config, and a small cloud account used for learning.

!!! note "Why this runbook is retroactive"
    At this point in the guide you've already manually clicked through Cloudflare and UDM (Runbooks 2 and 6). Runbook 7 is therefore retroactive — you're IaC-ifying configuration you already created. That's still valuable: the next time you add a service, you'll add a Cloudflare record via `terraform apply` instead of clicking through the UI. If you want a stricter IaC-first workflow, you can move Runbook 7 to before Runbook 6 on a future rebuild.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 3–4 hours |
| **Runs On** | Your machine |
| **Depends On** | Runbook 6, Runbook 2 |

## Where Terraform Genuinely Fits

| Domain | Why it fits |
|---|---|
| Cloudflare DNS | Real API, records change as you add services |
| UniFi config | Provider exists, VLANs/firewall as code |
| Cloud practice | Cloud APIs are Terraform's natural habitat |

## Where Terraform Does NOT Fit

- Bare-metal node provisioning — use Ansible (Runbook 4)
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

## Project Structure

```
homelab-terraform/
├── backend.tf            # remote-state config (see below)
├── cloudflare/
├── unifi/
├── cloud-practice/
└── .gitignore
```

!!! warning "Don't commit state or plaintext tfvars"
    Never commit `.tfstate`, `.tfvars` (plaintext), or anything containing API tokens. The `.gitignore` from Runbook 1 covers this, with the `!*.enc.tfvars` exception allowing sops-encrypted tfvars through.

## Remote State

By default, Terraform writes state to `terraform.tfstate` in the working directory. That's local-only — every machine has a different view, and `.gitignore` blocks committing it. For a homelab this is fine if you only run Terraform from one host. For anything more, use a remote backend.

Two reasonable options:

=== "S3 backend → MinIO on NAS"
    Stand up MinIO on the NAS (Runbook 9 references this for Velero too):

    ```yaml
    # NAS docker-compose snippet
    services:
      minio:
        image: minio/minio:latest
        command: server /data --console-address ":9001"
        environment:
          MINIO_ROOT_USER: <USER>
          MINIO_ROOT_PASSWORD: <PASSWORD>
        volumes:
          - ./minio-data:/data
        ports: ["9000:9000", "9001:9001"]
    ```

    Then `backend.tf`:

    ```hcl
    terraform {
      backend "s3" {
        bucket                      = "tfstate"
        key                         = "homelab/cloudflare.tfstate"
        endpoint                    = "http://10.0.0.50:9000"   # NAS IP
        region                      = "us-east-1"
        skip_credentials_validation = true
        skip_metadata_api_check     = true
        skip_region_validation      = true
        force_path_style            = true
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

The `paultyng/unifi` provider was archived in April 2026. The maintained drop-in successor is `ubiquiti-community/unifi` (v0.41.x schema-compatible). `filipowm/unifi` v1.x is a more aggressive refactor with schema changes — fine if you're starting fresh, but the drop-in is the safer first move.

```hcl
terraform {
  required_providers {
    unifi = {
      source  = "ubiquiti-community/unifi"
      version = "~> 0.41"
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

Use sops + age ([Runbook 1 Step 5](01-git.md#step-5-secret-management-with-sops--age)). Tfvars are encrypted as `secrets.enc.tfvars`, which is allowed through the `.gitignore` via the `!*.enc.tfvars` exception.

## Workflow

```bash
cd homelab-terraform/cloudflare

# Decrypt tfvars at-need
sops --decrypt secrets.enc.tfvars > secrets.tfvars

terraform init     # uses backend.tf for remote state
terraform plan -var-file=secrets.tfvars
terraform apply -var-file=secrets.tfvars
terraform fmt -recursive

rm secrets.tfvars  # plaintext gone again
```

!!! tip
    The single most impressive resume item from this runbook: "managed bare-metal network infrastructure (UniFi VLANs, firewall rules, DHCP) and DNS as code using Terraform, with CI-driven apply via Woodpecker."

## Verification

- [ ] Drift check: after the first apply, `plan` returns zero changes:

    ```bash
    cd homelab-terraform/cloudflare && terraform plan
    # Expected: 'No changes. Your infrastructure matches the configuration.'
    ```

- [ ] Cloudflare zone reflects Terraform state: visit dash.cloudflare.com — the records for `vault` / `nextcloud` / `forgejo` / etc. all match `var.traefik_ip`.
- [ ] If you've wired Woodpecker CI for Terraform: pushing a tfvars change triggers a `terraform plan` job whose output appears in the Woodpecker UI.
