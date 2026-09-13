# SQLite on local-path storage

**Date:** 2026-05
**Status:** Active

## Context

Several cluster apps (Forgejo, Vaultwarden, linkding, Actual Budget, Donetick)
use SQLite as their database. The cluster's default `StorageClass` is
`nfs-storage`, backed by an NFS export from topaz.

SQLite requires POSIX file-locking semantics that NFS doesn't reliably
provide. Running SQLite on an NFS volume risks silent database corruption.

## Decision

Every app that uses SQLite must store its database on the `local-path`
`StorageClass` (node-local SSD on emerald), pinned with
`nodeSelector: app-state=true` and `strategy: Recreate`.

## Consequences

- SQLite apps are pinned to the emerald node. A node failure takes them
  offline until the node recovers.
- Velero's file-system backup covers the local-path volumes. The
  `defaultVolumeType` must be set to `local` in the Velero schedule, or
  `PodVolumeBackup` reports zero bytes.
- NFS-backed apps (Nextcloud, Paperless) use an external PostgreSQL database
  on the NAS instead of SQLite — the storage tier and the database engine are
  chosen together.
- The [Storage and data architecture](../../concepts/storage.md) documents the
  four-tier model this decision enforces.
