# homelab-docs

Documentation, runbooks, diagrams, and decision logs for the homelab.

This is the only public repo in the five-repo layout — it contains no infrastructure code and no secrets, so it's safe to share.

## Related repos

| Repo | Purpose |
|---|---|
| homelab-docs | This repo — docs, runbooks, decisions |
| homelab-ansible | OS provisioning playbooks for the Turing Pi 2 cluster |
| homelab-manifests | k3s YAML / Helm values, watched by ArgoCD |
| homelab-terraform | Cloudflare DNS, UniFi, and other cloud/network IaC |
| homelab-secrets | sops/age-encrypted secrets (private) |
