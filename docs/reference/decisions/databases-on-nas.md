# Databases on the NAS

**Date:** 2026-06
**Status:** Active

## Context

Apps that outgrow SQLite need PostgreSQL. The cluster could run an in-cluster
PostgreSQL operator (CloudNativePG, Zalando, CrunchyData), but CM4 nodes have
limited RAM and eMMC storage, and the NAS already has an x86 CPU, real disks,
and Docker.

## Decision

Run a shared PostgreSQL instance on the NAS (10.0.20.50:5433) in Docker,
outside the cluster. Each app gets its own database and role, scoped with
`pg_hba.conf` entries.

## Consequences

- The database benefits from the NAS's x86 CPU and disk I/O — sustained
  fsync'd writes perform better than on CM4 eMMC.
- A nightly `pg_dumpall` script enumerates databases dynamically and ships
  dumps to a Garage S3 bucket. Adding a new app's database requires no
  backup-script change.
- The database is outside Kubernetes — Velero doesn't back it up, and the
  nightly dump is the recovery path.
- Apps connect to the NAS IP on port 5433 (UGOS owns 5432). Network
  partitions between the cluster and the NAS take down database-dependent
  apps.
- An in-cluster operator (CloudNativePG) remains the upgrade path if the
  cluster hardware improves.
