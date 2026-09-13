# Forgejo container registry

**Date:** 2026-06
**Status:** Active

## Context

CI pipelines build OCI container images and need somewhere to push them.
Options: Docker Hub (public, rate-limited), Harbor (full-featured private
registry with scanning and signing), or Forgejo's built-in container registry
(basic OCI storage with access control, enabled through `[packages]` in
`app.ini`).

## Decision

Use Forgejo's built-in container registry. Harbor remains the upgrade path if
the cluster grows or image scanning (CVE detection), retention policies, or
image signing (cosign / sigstore) become requirements.

## Consequences

- No additional deployment — the registry ships with Forgejo and shares its
  storage and auth.
- Pipelines push and pull images without Docker Hub rate limits or external
  registry dependencies.
- The registry provides basic OCI storage only. No image scanning, no retention
  policies, no signature verification.
- BuildKit pods authenticate to the registry through a Docker-config-format
  Secret mounted at `/home/user/.docker/config.json`.
- Migrating to Harbor later is a container-image re-push, not a workflow
  redesign — the same `config.json` pattern works for any OCI registry.
