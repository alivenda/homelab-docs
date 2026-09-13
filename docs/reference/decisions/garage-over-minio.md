# Garage over MinIO for S3

**Date:** 2026-06
**Status:** Active

## Context

The backup system needs an S3-compatible object store on the NAS. Every backup
job — etcd snapshots, Velero, Sealed Secrets key dumps, Immich DB syncs, Home
Assistant, Audiobookshelf — targets a dedicated bucket on this store.

MinIO Community Edition was the default homelab choice for years. In 2025–2026
MinIO effectively abandoned the free tier: the admin console was stripped from
CE (May 2025), free Docker/Quay image publishing stopped (October 2025, last
tag `RELEASE.2025-10-15T17-29-55Z`), and the `minio/minio` repo was archived
read-only (April 2026).

## Decision

Use [Garage](https://garagehq.deuxfleurs.fr/) (Rust, self-host-focused) as the
S3 store, running on the NAS at 10.0.20.50:9000. Each consumer gets its own
bucket and least-privilege key.

## Consequences

- Garage is a single static binary with no runtime dependencies — the Docker
  image is small and starts in seconds.
- Garage doesn't support S3 conditional writes (If-Match / If-None-Match), so
  S3-native locking (for example, Terraform's S3 state backend lock) isn't
  available. The Terraform backend uses DynamoDB-style locking through a
  separate mechanism instead.
- The plain-HTTP S3 API (`http://10.0.20.50:9000`) stays on the Lab VLAN.
  Clients connect with `s3ForcePathStyle` and `s3_region = "us-east-1"` to
  match k3s's defaults.
- Off-site backup ships the entire Garage data directory to Backblaze B2, so
  every bucket inherits an off-site copy without per-consumer cloud targets.
