# homelab-docs

Documentation, runbooks, diagrams, and decision logs for the homelab.

The runbooks are authored as Markdown under `docs/` and rendered as a static site with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). The Markdown source renders natively in GitHub as a fallback view.

This is the only public repo in the five-repo layout — it contains no infrastructure code and no secrets, so it's safe to share.

## Local preview

```bash
python -m venv .venv
source .venv/bin/activate   # fish users: source .venv/bin/activate.fish
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>. Live-reload watches `docs/` for changes.

## Build static site

```bash
mkdocs build
# Output in ./site/
```

## Related repos

| Repo | Purpose |
|---|---|
| homelab-docs | This repo — docs, runbooks, decisions |
| homelab-ansible | OS provisioning playbooks for the Turing Pi 2 cluster |
| homelab-manifests | k3s YAML / Helm values, watched by ArgoCD |
| homelab-terraform | Cloudflare DNS, UniFi, and other cloud/network IaC |
| homelab-secrets | sops/age-encrypted secrets (private) |

## Contributing

Branch + PR per repo convention — including for README and runbook edits. No direct pushes to `main`.

Page structure, admonition usage, anchor rules, and the public-repo constraints are codified in [STYLE.md](STYLE.md) — `docs/backups.md` and `docs/forgejo.md` are the reference implementations.
