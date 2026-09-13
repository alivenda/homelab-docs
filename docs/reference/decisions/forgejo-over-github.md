# Forgejo over GitHub

**Date:** 2025-12
**Status:** Active

## Context

The homelab needs a Git host for its five repositories. GitHub is the default
choice, but the project aims to self-host as much of the stack as possible, and
a local Git server removes a dependency on an external service for the GitOps
deployment pipeline (ArgoCD pulls from the Git host on every sync).

## Decision

Use [Forgejo](https://forgejo.org) as the primary Git host. GitHub mirrors the
public `homelab-docs` repo as a read-only copy.

Forgejo is a community fork of Gitea (itself a Go fork of Gogs) — lightweight,
ARM64-native, and actively maintained. It includes a built-in container
registry and supports AGit-flow pull requests over SSH, which removes the need
for a CLI tool or browser round-trip to open a PR.

## Consequences

- ArgoCD syncs against the local Forgejo instance — no external network
  dependency for deployments.
- PRs use AGit pushes (`git push origin HEAD:refs/for/main -o topic=…`)
  rather than GitHub-style fork workflows.
- Merges use the Forgejo API, not `git push -o merge` (which fast-forwards
  `main` but doesn't close the PR).
- Contributors who discover the project find it on GitHub, not Forgejo.
