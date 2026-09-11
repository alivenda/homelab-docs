# Zone-based firewall policies

**Date:** 2026-05
**Status:** Active

## Context

The UDM routes between three VLANs: Default (management), Trusted (user
devices), and Lab (cluster + NAS). UniFi OS 3.x used a legacy `LAN IN / LAN
OUT` rule list with a default-allow model — you add drop rules for traffic you
don't want.

UniFi OS 4 introduced the Policy Engine with zones and a zone matrix. Zones
group networks, the matrix sets a default action (Allow or Block) between each
pair, and policies override the matrix for specific flows.

## Decision

Use the Policy Engine's zone-based firewall (UniFi OS 4+) instead of legacy LAN
IN rules. Three custom zones (Trusted, Lab, IoT) map to VLANs 10, 20, and 30.
The zone matrix defaults to Block between custom zones, and explicit allow
policies open only the flows each zone needs.

## Consequences

- Default-deny between custom zones is built in. Only allowed flows are listed,
  and everything else is dropped.
- Lab can't initiate connections to any other zone. Blast radius from a
  compromised workload is bounded to the Lab VLAN.
- Trusted devices reach specific NAS services (SMB, UGOS web UI) through
  per-service allow policies, not a blanket zone-to-zone Allow.
- The UniFi Terraform provider doesn't support zone-based firewall policies, so
  this configuration is manual (click-ops) for now.
- The IoT zone isolates smart-home devices. They reach the internet and
  respond to mDNS reflections, but can't initiate to Trusted or Lab.
