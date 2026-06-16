# Runbook 21: Arr Stack

Media automation: **Prowlarr** (indexer manager), **Sonarr** (TV), **Radarr**
(films), **Lidarr** (music), and **qBittorrent** behind a **gluetun** VPN. The
stack runs on the cluster and feeds **Plex**, which runs on the NAS.

!!! note "No request UI"
    This stack intentionally omits a request portal (Seerr/Jellyseerr) — content
    is added directly in the arr apps. A Plex-facing request UI can be added
    later if wanted: `seerr/seerr` (the maintained Overseerr+Jellyseerr merger,
    arm64, listens on 5055, SQLite config in `/app/config` → `local-path`) would
    drop in as another Deployment, with an **unauthenticated** HTTPRoute (it has
    its own Plex login).

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Runs On** | k3s (cluster); media + Plex on the NAS |
| **Depends On** | R5 (k3s), R6 (Traefik Gateway + wildcard TLS), R18 (Authelia — ForwardAuth), R8 (Cloudflare DNS), a VPN provider account |

!!! note "Source of truth is the manifest"
    The deployed state lives in `homelab-manifests/apps/arr/manifests/` and
    `bootstrap/arr.yaml`. This runbook is the *reasoning and the procedure*; it
    shows the non-obvious YAML (the VPN sidecar, the NFS PV, the ForwardAuth
    wiring) but doesn't restate every Deployment. The app's `README.md` is the
    operational companion.

## What it is

Prowlarr indexes torrent trackers / Usenet indexers in one place and pushes them
to the others. Sonarr/Radarr/Lidarr do search, grab, and post-processing, and you
add content directly in their UIs. qBittorrent downloads. All LinuxServer.io
images ship **arm64** ✅.

```
you ──add show/film──▶ Sonarr/Radarr/Lidarr ──grab via──▶ Prowlarr ──▶ indexers
                              │                                │
                              └────── add to ──▶ qBittorrent ──┘ (through gluetun VPN)
                                          │
                       hardlink completed download into the Plex library
                                          │
                                        Plex (NAS) ──streams──▶ you
```

**Auth:** Authelia **ForwardAuth** protects every arr UI — they have no
meaningful SSO and their API keys are for integrations, not users.

## Architecture decisions (read before deploying)

This runbook was rewritten against the live deployment; earlier revisions had
several traps now corrected.

### Storage — `/config` and `/data` go to different tiers

The single most important decision. Each app has **two** kinds of state:

- **`/config` (SQLite) → `local-path`.** Every Servarr app keeps an embedded
  SQLite DB in `/config` (`sonarr.db`, `radarr.db`, `lidarr.db`, `prowlarr.db`).
  **SQLite corrupts on NFS** — the
  [Servarr wiki FAQ](https://wiki.servarr.com/sonarr/faq) is explicit: *"a common
  cause of database corruption is putting the database on a network share… the
  AppData folder must be on local storage."* So `/config` uses the node-local
  `local-path` class, pinned to the app-state node (emerald) with
  `nodeSelector: app-state=true` + `strategy: Recreate`. **An earlier revision
  put every `/config` on `nfs-storage` — that was wrong** and is the same trap
  documented for linkding / Actual Budget / Donetick. See
  [Storage & Data Architecture](storage-architecture.md).
- **`/data` (downloads + media) → one RWX NFS export on the NAS.** Bulk media is
  TB-scale and Plex reads it locally on the NAS, so the media library lives on
  the NAS 6-bay array, exported over NFS to the cluster — **not** on the topaz
  `nfs-storage` SSD. It's a *static* `PersistentVolume` (`pv-data.yaml`) bound
  1:1 to its PVC by `claimRef`, distinct from the dynamic `nfs-storage` class.

### Hardlinks need one filesystem

The Servarr wiki's strongest recommendation is a **unified `/data` layout**:
downloads and the library under a single root so completed grabs are *hardlinked*
into place (a second directory entry for the same inode — instant, no bytes
copied, keeps seeding) instead of copy+delete. That requires `downloads/` and
`media/` to be **siblings on one filesystem**, so they share one NFS export and
each app mounts it at `/data`:

```
/data/
├── downloads/      ← qBittorrent
└── media/
    ├── tv/         ← Sonarr
    ├── movies/     ← Radarr
    └── music/      ← Lidarr
```

NFS supports `link()`, so hardlinks work across this export with the apps on the
cluster and Plex on the NAS reading the same files locally. (A future move of the
whole stack to NAS-side Docker would make hardlinks node-local with no NFS hop —
an optimisation, not a requirement; see the end of this runbook.)

### Download client routes through a VPN

qBittorrent runs with a **gluetun** sidecar in the same pod. gluetun brings up a
WireGuard/OpenVPN tunnel and a kill-switch; both containers share the pod network
namespace, so **all** of qBittorrent's traffic exits via the VPN and nothing
leaks to the home WAN IP if the tunnel drops. This replaces R21's original
no-VPN qBittorrent.

### Manifests + Application layout

Raw manifests in `apps/arr/manifests/` (one `Deployment`+`Service` per app, the
shared PVCs, the NFS PV, the `Middleware`, the `HTTPRoute`s, the gluetun
`SealedSecret`), driven by a single ArgoCD `Application` (`bootstrap/arr.yaml`)
with `prune: false` and **no ServerSideApply** (the Gateway controller mutates
the HTTPRoutes post-apply → SSA causes permanent OutOfSync). This replaces the
old loose `apps/arr/*.yaml`.

### Pinned, arm64 images

`:latest` is never used. Verified multi-arch tags at the 2026-06 bring-up:

| Component | Image |
|---|---|
| Prowlarr | `lscr.io/linuxserver/prowlarr:version-2.4.0.5397` |
| Sonarr | `lscr.io/linuxserver/sonarr:version-4.0.17.2952` |
| Radarr | `lscr.io/linuxserver/radarr:version-6.2.1.10461` |
| Lidarr | `lscr.io/linuxserver/lidarr:version-3.1.0.4875` |
| qBittorrent | `lscr.io/linuxserver/qbittorrent:version-5.2.2_v2.0.13` |
| gluetun | `qmcgaw/gluetun:v3.41.1` |

`TZ=Etc/UTC` everywhere (was Europe/London). All LinuxServer apps take
`PUID=1000`, `PGID=1000`, `UMASK=022`.

## Step 1 — NAS NFS export (prerequisite)

On the NAS (`10.0.20.50`):

1. Create the unified layout above under one shared folder (e.g.
   `/volume1/data`), `downloads/` and `media/{tv,movies,music}` as siblings.
2. Export it over NFS to the four node IPs `10.0.20.10-13` (Lab VLAN),
   read-write, mapping/allowing **UID/GID 1000** so the apps can write.
3. Set `spec.nfs.path` in `pv-data.yaml` to the **actual** export path (the
   committed `/volume1/data` is the documented default — confirm it against your
   UGOS layout).
4. In Plex, add libraries from the local paths
   `/volume1/data/media/{tv,movies,music}`.

The static PV (illustrative — full file in the repo):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: arr-data
spec:
  capacity: { storage: 4Ti }     # nominal; NFS enforces no quota here
  accessModes: [ReadWriteMany]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  mountOptions: [nfsvers=4.1]
  nfs:
    server: 10.0.20.50
    path: /volume1/data          # confirm against the NAS
  claimRef: { namespace: arr, name: arr-data }
```

## Step 2 — VPN credentials (gluetun)

qBittorrent's traffic is useless (and unsafe) without the tunnel, so the
`gluetun-secrets` SealedSecret is a hard prerequisite. Until it's resealed with
real values the qbittorrent pod stays Pending. Capture values into shell vars and
seal (WireGuard example; adjust per provider):

```fish
set GLUETUN_PROVIDER 'your-provider'           # mullvad, protonvpn, pia, …
set GLUETUN_WG_KEY   'your-wireguard-private-key'
set GLUETUN_WG_ADDR  'your-wireguard-address'  # e.g. 10.2.0.2/32
set GLUETUN_COUNTRY  'Netherlands'

kubectl create secret generic gluetun-secrets --namespace arr \
  --from-literal=VPN_SERVICE_PROVIDER="$GLUETUN_PROVIDER" \
  --from-literal=VPN_TYPE=wireguard \
  --from-literal=WIREGUARD_PRIVATE_KEY="$GLUETUN_WG_KEY" \
  --from-literal=WIREGUARD_ADDRESSES="$GLUETUN_WG_ADDR" \
  --from-literal=SERVER_COUNTRIES="$GLUETUN_COUNTRY" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets --format yaml \
  > apps/arr/manifests/gluetun-sealed.yaml
```

OpenVPN providers use `VPN_TYPE=openvpn` + `OPENVPN_USER`/`OPENVPN_PASSWORD`. For
inbound peer connectivity add `VPN_PORT_FORWARDING=on` (PIA/ProtonVPN). Save the
plaintext to Vaultwarden; add the gitleaks fingerprints to `.gitleaksignore`.

### The sidecar networking gotcha

gluetun blocks everything by default and owns the pod netns, so the qBittorrent
WebUI is unreachable unless you punch two holes:

```yaml
# gluetun container env
- { name: FIREWALL_INPUT_PORTS,     value: "8080" }                       # WebUI inbound
- { name: FIREWALL_OUTBOUND_SUBNETS, value: "10.42.0.0/16,10.43.0.0/16" } # pod+svc CIDRs
```

`FIREWALL_OUTBOUND_SUBNETS` lets Sonarr/Radarr/Lidarr and Traefik reach
`qbittorrent:8080`; without it the HTTPRoute 502s and download-client tests fail.
gluetun needs `securityContext.capabilities.add: [NET_ADMIN]` (it creates
`/dev/net/tun` itself and falls back to wireguard-go — no host device mount or
kernel module on the CM4s). Its `livenessProbe` runs gluetun's own healthcheck so
a dead tunnel restarts the sidecar and re-arms the kill-switch.

## Step 3 — Auth (ForwardAuth)

One `authelia-forwardauth` Middleware in the `arr` namespace; each protected
HTTPRoute references it with an `ExtensionRef` filter (the
[deploy-pattern recipe](apps-deploy-pattern.md#forwardauth-authelia-gates-the-app-at-the-proxy)).
The route fails closed.

```yaml
# every arr route:
rules:
  - filters:
      - type: ExtensionRef
        extensionRef: { group: traefik.io, kind: Middleware, name: authelia-forwardauth }
    backendRefs:
      - { name: sonarr, port: 8989 }
```

## Step 4 — DNS

Add the subdomains to `var.services` in the Cloudflare module and `tofu apply`
**before** they'll resolve (one A record each at the Traefik LB):
`sonarr`, `radarr`, `lidarr`, `prowlarr`, `qbt`.

## Step 5 — Deploy

Commit `apps/arr/` + `bootstrap/arr.yaml` via a branch/PR and let ArgoCD sync.
The app won't go fully Healthy until Step 1 (NAS export) and Step 2 (gluetun
secret) are done.

## Step 6 — First-run wiring

1. **qBittorrent password** — the image prints a *random temporary* admin
   password to its logs on first start:
   `kubectl -n arr logs deploy/qbittorrent -c qbittorrent | grep -i password`.
   Log in at `https://qbt.alivenda.dev`, set a permanent one (Settings → Web UI),
   save to Vaultwarden, set the save path to `/data/downloads`.
2. **Prowlarr → apps** — Settings → Apps; add each with its in-cluster URL +
   API key: `http://sonarr.arr.svc.cluster.local:8989`,
   `http://radarr.arr.svc.cluster.local:7878`,
   `http://lidarr.arr.svc.cluster.local:8686`.
3. **Download client** — in each arr app, Settings → Download Clients → add
   qBittorrent at `http://qbittorrent.arr.svc.cluster.local:8080`.
4. **Root folders** — Sonarr `/data/media/tv`, Radarr `/data/media/movies`,
   Lidarr `/data/media/music`; confirm "Use Hardlinks instead of Copy" is on.
5. Save every app's API key to Vaultwarden.

## Verification

- [ ] `kubectl get pods -n arr` — all Running (qbittorrent has 2/2: gluetun + app).
- [ ] ArgoCD shows `arr` Synced / Healthy.
- [ ] Each UI serves over the wildcard cert and **challenges with Authelia**
      before loading.
- [ ] gluetun's public IP is the VPN's, not home:
      `kubectl -n arr exec deploy/qbittorrent -c gluetun -- wget -qO- https://ipinfo.io/ip`.
- [ ] Prowlarr → Settings → Apps shows Sonarr/Radarr/Lidarr **connected**.
- [ ] A test grab completes and lands (hardlinked) in the right
      `/data/media/<type>` folder; Plex sees it.
- [ ] `PodVolumeBackup` for each `config` volume shows **bytes > 0** after the
      04:00 UTC velero run (not just `Completed`).
- [ ] All API keys + the qBittorrent password + the VPN config in Vaultwarden.

## Optional future — move the stack to the NAS

If you later run the arr stack in NAS-side Docker (e.g. after a NAS RAM upgrade),
hardlinks become node-local with no NFS hop. Stop the cluster Deployments, copy
the `local-path` configs to NAS local storage, recreate the unified `/data`
layout in a compose file, and front each NAS UI through Traefik with a
selector-less `Service` + `EndpointSlice` (the Immich pattern), re-creating the
ForwardAuth `Middleware` in that namespace. Not required — the cluster + NFS
design above works today.
