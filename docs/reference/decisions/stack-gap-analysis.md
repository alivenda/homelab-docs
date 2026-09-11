# Stack gap analysis

**Date:** 2026-05
**Status:** Active

## Context

The homelab cluster, NAS, and supporting infrastructure were operational, but no
systematic review had mapped what the stack covered against what a full
personal-cloud replacement requires.

## Decision

Conduct a comprehensive gap analysis: map every
[awesome-selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted)
category against the specific hardware (Turing Pi 2 cluster, UGREEN DXP6800 Pro
NAS, slate Mac mini, pyrite Pi 3), existing services, RAM budget, and day-to-day
needs. Record the result as a permanent decision document.

## Consequences

- The analysis produced a prioritized service list — 🔴 high-priority gaps,
  🟡 good additions, ⚪ skips, and 🚫 traps — with RAM estimates and ARM64
  compatibility checked for each candidate.
- A recommended deployment order (22 services) sequences the build so
  foundational services (DNS, SSO, notifications) land before the apps that
  depend on them.
- The document lives at `reference/service-selection.md` as the frozen record
  of what was decided and why. Priority markers reflect the moment of analysis,
  not current deployment status — the [App Catalog](../app-catalog.md) and
  home page track what's live.
- Several entries were later revised: the Arr stack was shelved, Uptime Kuma
  was retired in favor of blackbox probes, and Collabora landed as a Nextcloud
  app rather than a standalone deployment.
