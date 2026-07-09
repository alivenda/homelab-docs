# Storage & Data Architecture

Where every kind of data lives, and why. This is the **decision record** behind the
storage choices the runbooks and the [App Catalog](apps-catalog.md) assume — read it
once and a per-app storage line never reads as arbitrary again.

!!! note "Source of truth is the manifest"
    The deployed StorageClasses live in `homelab-manifests/infrastructure/`. This page
    is the *rationale and the rules*; it deliberately doesn't restate YAML.

## The four storage tiers

| Tier | Backing | Where | Use it for |
|---|---|---|---|
| **Bulk / flat / RWX** | topaz SATA SSD over NFS | `nfs-storage` (default class) | Media, Nextcloud *files*, document blobs, config dirs — anything that isn't a live database |
| **Embedded DB (node-local)** | each node's local disk | `local-path` (standalone provisioner) | SQLite app data — node-pinned, POSIX-correct |
| **Relational DB** | NAS (x86), Docker | Postgres / MariaDB server on the NAS | Nextcloud, Paperless, Reactive Resume, BookStack, … |
| **Cache (ephemeral)** | node memory | in-cluster Redis/Valkey pod | Caches for Paperless, Reactive Resume — not a system of record, no durable PVC |

!!! note "Implementation status (2026-06)"
    - `nfs-storage` — **live** (`infrastructure/nfs-provisioner`).
    - `local-path` standalone provisioner — **live** (`infrastructure/local-path-provisioner`).
      k3s was installed with `--disable local-storage`, so the built-in class is gone; this
      separate provisioner gives SQLite apps a correct home. The SQLite apps (Vaultwarden,
      lldap, Forgejo, linkding, …) live on it, and a backup→restore drill confirmed
      local-path volumes round-trip through velero (see [below](#the-local-path-tier)).
    - NAS relational DB — **live** (PostgreSQL 18 on the NAS at `10.0.20.50:5433`, see
      [NAS PostgreSQL](nas-postgres.md)); databases are provisioned per app at each app's
      bring-up. MariaDB stays unbuilt until an app forces it. Immich's own Postgres
      predates the shared server and stays separate.

## The one-paragraph version

The cluster nodes are ARM CM4s with eMMC plus a single shared SATA SSD — strong at
running many small app pods, weak at database I/O. So data is split by **what it is**,
not by which app owns it: bulk and flat files on NFS, embedded SQLite on node-local
disk, real relational databases on x86 hardware (the NAS), caches in memory. Durability
everywhere is **backups to Garage S3, not redundancy** — there is one real disk, by
design (the storage SPOF is mitigated by backups, see [Backups & DR](backups.md)).

## Why this isn't "just use the default StorageClass"

Two hardware facts drive every choice below:

1. **SQLite corrupts on NFS.** [sqlite.org/howtocorrupt](https://www.sqlite.org/howtocorrupt.html)
   §2.1: *"This is especially true of network filesystems and NFS in particular… database
   corruption might result."* A large share of self-hosted apps embed SQLite (linkding, Actual Budget, the Arr stack, Forgejo, Vaultwarden, …) and several have no
   other backend. They **cannot** sit on `nfs-storage`.
2. **The CM4s are the wrong place for a database.** PostgreSQL does sustained, fsync'd
   writes; the nodes have only eMMC (slow, write-endurance-limited, *and the boot medium*)
   plus the one SATA SSD on topaz that already serves NFS. Real relational databases belong
   on x86 hardware with a proper disk.

## How to decompose one app

A stateful app is rarely "one volume." Split it by data type and send each part to its tier:

- **App pods** → the cluster (their job; the CM4s are good at this).
- **Embedded SQLite** → `local-path` + `strategy: Recreate`.
- **Relational DB** (Postgres/MariaDB) → the NAS database server.
- **Bulk files** (uploads, documents, media) → `nfs-storage`.
- **Cache** (Redis/Valkey) → an ephemeral in-cluster pod, no durable PVC.

> **Worked example — Nextcloud:** PHP/app pods on the cluster, *files* on `nfs-storage`,
> *database* on the NAS Postgres, Redis as an ephemeral cache pod. Four parts, four homes.

## The local-path tier

k3s ships Rancher's local-path provisioner and marks its class the cluster **default**,
re-applying that annotation on **every restart** — which fights a custom default like
`nfs-storage`. So the cluster disables it (`--disable local-storage` in
`homelab-ansible/site.yml`) and hands "default" cleanly to `nfs-storage`.

The fix for SQLite apps is **not** to undo that disable — it's to run a **separate,
non-default** [rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)
as its own ArgoCD app. Because it's *our* manifest, not k3s's bundled one, k3s never
touches it or re-marks defaults, so it coexists with `nfs-storage`. SQLite apps opt in
with `storageClassName: local-path` + `strategy: Recreate` (the volume is RWO and pins to
a single node). velero's node-agent (filesystem/kopia backup) captures these volumes —
that's their durability, replacing the redundancy NFS can't provide here.

!!! danger "The tier only works with `defaultVolumeType: local`"
    velero's filesystem backup **cannot read `hostPath` volumes** — only `local` ones
    ([velero docs](https://velero.io/docs/main/file-system-backup/)). The provisioner
    defaults to `hostPath`, so the StorageClass **must** set the annotation
    `defaultVolumeType: local`. Without it, every backup of a SQLite DB here finishes
    `Completed` having captured **zero** bytes — a silent, total loss of durability that
    looks healthy. This was caught by a restore drill, not in theory; the gate is a
    non-empty `PodVolumeBackup`, never a `Completed` status.

Embedded SQLite databases are tiny, so node eMMC is an acceptable medium even on a 16 GB
node. SQLite apps pin to the designated app-state node — `emerald (Node 2)`, label
`app-state: "true"` — rather than chasing the biggest disk: the 32 GB-eMMC nodes are ruby
(control plane) and topaz (NFS server + monitoring), both better kept free of app state.

## Relational databases — on the NAS, not the cluster

The relational apps (Nextcloud, Paperless, Reactive Resume → PostgreSQL; BookStack →
MariaDB) get a **shared database server on the NAS** — the same x86 Docker host that already
runs Immich's Postgres, so this extends an established pattern rather than inventing one.

Shape of the layer:

- **One PostgreSQL container** on the NAS (`10.0.20.50:5433`, Lab VLAN — port 5433 because
  UGOS runs its own internal Postgres on loopback 5432) with a **database + role per app** —
  not one Postgres per app.
- **One MariaDB container** for the MySQL-only apps (BookStack; optionally Ghost, Monica) —
  deferred until one of them actually deploys.
- Cluster apps connect over the Lab VLAN via `DATABASE_URL`, credentials from a `SealedSecret`.
- **Backups:** nightly `pg_dump` / `mysqldump` per database to Garage S3 (Backups), gated by a
  seeded restore drill. Move to pgBackRest / WAL archiving later if you want point-in-time
  recovery.
- **Access:** `pg_hba` allows each role only its own database, only from the four node IPs
  (pod egress is SNAT'd to node IPs); no superuser over TCP. It stays on the trusted Lab
  VLAN, never exposed.

[NAS PostgreSQL](nas-postgres.md) is the implementation.

!!! tip "Caches stay in-cluster"
    Redis/Valkey for Paperless and Reactive Resume is a **cache**, not a system of record.
    Run it as an ordinary in-cluster pod with no durable volume — there's nothing to back up
    and no reason to send it to the NAS.

### Why not CloudNativePG (in-cluster Postgres)?

[CloudNativePG](https://cloudnative-pg.io/) is genuinely good and **fully self-hosted** —
Apache-2.0, a CNCF Sandbox project, ships official `arm64` images, and backs up to *your own*
Garage S3 (no cloud account, no subscription; "cloud-native" means *Kubernetes-native*). It's
shelved here for a **hardware** reason, not a values one: its headline feature is **HA
PostgreSQL across 3+ replicas with independent per-node storage**, and this cluster has a single
shared SSD (a storage SPOF by design) and write-limited eMMC. You'd run it single-instance on the
wrong disk — paying operator complexity for the one capability the hardware can't provide.
Revisit CNPG only if the nodes ever gain per-node NVMe.

## Durability model

One real disk means durability is **backups, not redundancy** — applied consistently per tier:

| Data | Backed up by | Destination |
|---|---|---|
| Cluster objects (etcd) | k3s etcd snapshots | local + Garage S3 |
| `nfs-storage` + `local-path` PVCs | velero node-agent (filesystem) | Garage S3 |
| NAS relational DBs | `pg_dump` / `mysqldump` (or pgBackRest) | Garage S3 |

`local-path` is only velero-backable because its StorageClass sets `defaultVolumeType: local`
— see the [danger note above](#the-local-path-tier). `hostPath` volumes are silently skipped.

## Open cleanup items

These predate this record and should be reconciled to it as each app is touched:

- ~~Audit the App Catalog for SQLite apps still on `nfs-storage`~~ — **done (2026-07
  audit):** every SQLite app sits on `local-path` — Vaultwarden, lldap, linkding,
  Actual Budget, Donetick, ntfy, Woodpecker, and Forgejo's data volume (Forgejo's *repos*
  deliberately stay on `nfs-storage`: flat git files, not SQLite).
- **BookStack and Reactive Resume assume cluster-hosted databases** — revise their DB sections to the
  NAS server above when each app is deployed. (Nextcloud and Paperless-ngx are done —
  Nextcloud documents the four-tier decomposition this page uses as its worked example, and Paperless-ngx
  was trued up at Paperless's 2026-06 bring-up.)
