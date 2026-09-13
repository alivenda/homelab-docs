# Blackbox probes over Uptime Kuma

**Date:** 2026-06
**Status:** Active

## Context

Uptime Kuma ran as a standalone monitoring dashboard, checking whether public
endpoints were reachable and displaying a status page. It ran as raw manifests
on `local-path` storage (SQLite needs POSIX locking).

Two problems emerged:

- For a solo operator, a status page has no audience.
- Uptime Kuma's checks are click-ops SQLite state, not GitOps — they can't be
  version-controlled or reproduced from a repo.

## Decision

Replace Uptime Kuma with declarative blackbox-exporter `Probe` custom resources
that feed Prometheus. Retire Uptime Kuma.

## Consequences

- The six public HTTPS endpoints are monitored by `Probe` CRs in the
  `infrastructure/blackbox-exporter` manifests — fully GitOps, reproducible
  from a fresh cluster build.
- `PublicEndpointDown` and certificate-expiry alerts fire through Alertmanager
  to ntfy, replacing Uptime Kuma's notification system.
- No status page. If a user-facing status page becomes useful, Uptime Kuma
  remains a viable option — it runs as `louislam/uptime-kuma:2` on `local-path`
  behind Authelia ForwardAuth.
