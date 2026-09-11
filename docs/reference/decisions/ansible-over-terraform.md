# Ansible over Terraform for OS provisioning

**Date:** 2025-12
**Status:** Active

## Context

The four CM4 nodes need OS-level configuration: static IPs, SSH hardening,
kernel parameters, NFS mounts, and package installs. Both Ansible and
Terraform can manage remote hosts.

## Decision

Use Ansible for OS provisioning. Terraform handles only cloud and network
resources (Cloudflare DNS, UniFi).

Ansible operates over SSH against the DietPi nodes and models configuration as
convergent tasks (packages, files, services). Terraform's strength is
declarative cloud-provider resources with a state file — it can manage remote
hosts through provisioners, but those are imperative escape hatches, not the
tool's design center.

## Consequences

- `homelab-ansible` owns everything from first boot to a k3s-ready node.
- `homelab-terraform` owns Cloudflare DNS records, UniFi network
  configuration, and any future cloud resources.
- The two tools share no state — Ansible inventories nodes by IP, Terraform
  tracks resources in its state file.
- Secrets follow different paths: Ansible uses sops + age (`community.sops`
  lookup), Terraform uses the `sops` provider.
