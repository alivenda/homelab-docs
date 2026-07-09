# Personal Cloud Stack — Complete Service Guide

A comprehensive review of every awesome-selfhosted category mapped against your specific hardware, existing stack, and day-to-day needs — written against the Turing Pi 2 cluster, UGREEN DXP6800 Pro NAS, slate (a Late-2014 Mac mini running Home Assistant OS as a Proxmox VM), and a dedicated Raspberry Pi for DNS (`pyrite`).

!!! note "Decision record, not the status page"
    This page is the gap analysis that chose the stack (2026-05/06) — it stays as
    the record of what was decided and why. Priority markers (🔴/🟡) describe how
    much a service was *wanted at that moment*, not what's deployed today; for
    current status see the [App Catalog](apps-catalog.md) and the
    Live/Planned/Retired table on [the home page](index.md). When a decision
    below has since changed — a service shelved, a fork picked over its rival —
    the entry says so rather than being silently rewritten.

---

## Hardware Context

| Platform | Specs | Already Running |
|----------|-------|-----------------|
| **Cluster (4× CM4)** | ARM64, 32 GB total RAM, 1 TB NFS | Traefik, ArgoCD, Prometheus/Grafana/Loki, Vaultwarden, Nextcloud, Paperless-ngx, Forgejo, Woodpecker |
| **NAS (DXP6800 Pro)** | x86-64, **8 GB RAM** (expandable) | Plex, Immich |
| **Home Assistant node (slate — Mac mini)** | x86-64, 16 GB RAM / 256 GB SSD, Proxmox | Home Assistant OS (VM: 2 vCPU, 4 GB) |
| **DNS node (`pyrite` — Pi 3 Model B)** | ARM64, 1 GB RAM | AdGuard Home (optional 2nd, `marcasite`, for failover) |

!!! note "NAS RAM is a meaningful constraint"
    At 8 GB, Plex + Immich can consume 2.5–6 GB under load, leaving 2–5.5 GB
    headroom — comfortable for Audiobookshelf (the one service from this stack
    that landed NAS-side), but not enough for Ollama without a RAM upgrade. The
    DXP6800 Pro accepts standard DDR5 SO-DIMMs — upgrading to 16 GB (~$35–50)
    opens up AI workloads and makes the NAS more comfortable overall. Services
    that need the upgrade are marked `⚠️ NAS RAM upgrade recommended`.

### Cluster RAM Budget

| Layer | Steady-state RAM |
|-------|-----------------|
| k3s system (4 nodes) | 1.2–2.0 GB |
| Traefik + ArgoCD + Sealed Secrets | 0.8–1.4 GB |
| Prometheus + Loki + Grafana | 1.0–1.6 GB |
| Woodpecker server + agent | 0.3–0.5 GB |
| **Existing apps** (Forgejo, Vaultwarden, Nextcloud, Paperless) | 1.5–2.5 GB |
| **Subtotal (current)** | **~5.8–8.0 GB** |
| Recommended new services (this doc) | ~2.5–4.0 GB |
| **Projected total** | **~8–12 GB of 32 GB** |

Headroom is comfortable. OCR (Paperless) and CI builds (Woodpecker) are still the cluster's spike events, not the new additions.

---

## Legend

| Symbol | Meaning |
|--------|---------|
| ✅ **Already covered** | In your existing stack |
| 🔴 **High priority gap** | Common daily-driver cloud replacement |
| 🟡 **Good addition** | Worthwhile, lower urgency |
| ⚪ **Skip / edge case** | Niche, heavy, ARM issues, or not relevant |
| ⬛ **Shelved / retired** | Decided against or removed after this page was written — see the entry for why |
| 🖥️ **Cluster** | Run on k3s |
| 💾 **NAS** | Run via Docker Compose on DXP6800 |
| 🌐 **DNS** | Run on a dedicated Raspberry Pi (`pyrite`; optional 2nd) |
| ⚠️ | Needs NAS RAM upgrade to run comfortably |
| 🚫 | Trap — don't self-host this |

---

## Part 1 — What You Already Have

| Service | Tool | Notes |
|---------|------|-------|
| File sync + Calendar + Contacts | Nextcloud ✅ | Nextcloud; also covers Notes app, Bookmarks app, Collabora |
| Document archive | Paperless-ngx ✅ | Paperless-ngx |
| Photo/video library | Immich ✅ | Immich (NAS) |
| Smart home hub | Home Assistant ✅ | Home Assistant (slate — Mac mini / Proxmox VM, Home Assistant OS) |
| Password manager + TOTP | Vaultwarden ✅ | Vaultwarden; also generates 2FA codes natively — no separate app needed. If you want strict separation between passwords and 2FA, [2FAuth](https://2fauth.app) is the self-hosted option. |
| Media streaming | Plex ✅ | NAS |
| Music streaming | Plex ✅ | NAS — Plex Media Server serves audio; Plexamp is the client player, not a separate daemon |
| Git hosting | Forgejo ✅ | Forgejo |
| CI/CD | Woodpecker ✅ | Woodpecker |
| GitOps | ArgoCD ✅ | Kubernetes |
| Monitoring + metrics | Prometheus + Grafana + Loki + Alloy ✅ | Observability |
| Backups | Velero + Garage ✅ | Backups |
| Object storage (backup target) | Garage ✅ | Backups — S3 store for etcd snapshots + Velero, runs on the NAS |
| Secrets management | Sealed Secrets ✅ | Cluster — encrypts secrets committed to Git for ArgoCD |
| VPN / remote access | Tailscale ✅ | Networking |
| HTTPS proxy | Traefik ✅ | Traefik |

---

## Part 2 — Foundational Layer (Do These First)

These unlock everything else. Add them before the "good additions" below.

### 🔴 DNS Ad-Blocking — AdGuard Home
**Category:** DNS | **Run:** 🌐 Raspberry Pi — `pyrite` (optional 2nd, `marcasite`, for failover) | **RAM:** ~100 MB | **ARM64:** ✅ Official multiarch | **Runbook:** [AdGuard Home](adguard-home.md)

Running on a dedicated Pi is the correct pattern: DNS must stay up independently of your cluster. One Pi is enough; a second gives you redundancy. AdGuard Home has *no* native cross-instance sync — when you run two, AdGuard Home runs the [`adguardhome-sync`](https://github.com/bakito/adguardhome-sync) container on the primary to keep the secondary's blocklists and config in lockstep.

This build runs `pyrite`, a Pi 3 Model B (1 GB RAM) — AdGuard Home uses ~100 MB, leaving plenty of headroom. Install via DietPi-Software (ID 126) or the official one-line installer on DietPi / Raspberry Pi OS Lite.

**Setup:** Configure your UDM to advertise `pyrite`'s IP (`10.0.0.20`) as the DNS server via DHCP on each VLAN — and add a second IP once you run `marcasite`. Handing clients two resolvers gives only *partial* failover, though — stub resolvers retry inconsistently and can stall on a dead server for several seconds. For true HA, front the pair with a shared [keepalived](https://www.keepalived.org/) VIP. Note the Pi Zero 2 W (a common pick for the second node) is Wi-Fi-only with no Ethernet port — for DNS infrastructure a USB-Ethernet dongle is worth it.

**Replaces:** Google's DNS, your ISP's DNS, in-browser ad blockers.

### 🔴 SSO / Auth Layer — Authelia
**Category:** Federated Identity | **Run:** 🖥️ Cluster | **RAM:** ~100–150 MB | **ARM64:** ✅ Official | **Runbook:** [Authelia](authelia.md)

You already have Traefik. Authelia adds a single login + 2FA layer, wired two different ways depending on the app (see Authelia):
- **OIDC** for apps with their own login (Nextcloud, Paperless, Forgejo, Vikunja, …) — they redirect to Authelia and manage their own session. **Do not** put forward-auth in front of these: their API clients (desktop sync, git CLI, mobile apps) send credentials directly and can't follow the redirect.
- **ForwardAuth** (Traefik middleware) only for apps with *no* login of their own (e.g. Homepage).
- 2FA (TOTP, WebAuthn) and single sign-on across all of the above in one place.
- LDAP-compatible user store via **lldap** (Rust, ~30 MB, ARM64 ✅ — lightweight LDAP backend), or flat file for simplicity.

**Why before everything else:** every new service you add will want auth. Doing it now means each future service gets SSO for free.

!!! tip "Authentik is the heavier alternative"
    ~600 MB Python vs. Authelia's ~100 MB Go — Authelia is the right fit for your cluster.

### 🔴 Push Notifications — ntfy
**Category:** Communication | **Run:** 🖥️ Cluster | **RAM:** ~50 MB | **ARM64:** ✅ Official | **Runbook:** [ntfy](ntfy.md)

Self-hosted push notifications for everything: backup alerts, Woodpecker pipeline results, Home Assistant automations, Paperless document ingested, etc. Replaces paid notification services. Native Android/iOS app. Integrates with Home Assistant, Grafana, shell scripts.

---

## Part 3 — High Priority Gaps (Daily Drivers)

These are the services people use every day that currently rely on commercial clouds.

### 🔴 RSS / News Reader — Miniflux
**Category:** Feed Readers | **Run:** 🖥️ Cluster | **RAM:** ~50 MB | **ARM64:** ✅ Official | **Runbook:** [Miniflux](miniflux.md)

**Miniflux** — Go binary, minimal, fast, Fever + Google Reader API (works with Reeder, NetNewsWire, etc.), very low RAM. PostgreSQL-only, so it rides the shared [NAS Postgres](nas-postgres.md) and runs fully stateless in-cluster.

Deployed choice. FreshRSS remains the feature-rich alternative if your needs outgrow Miniflux.

**Replaces:** Feedly, Inoreader, Google News.

### 🔴 Bookmarks — linkding
**Category:** Bookmarks | **Run:** 🖥️ Cluster | **RAM:** ~100–150 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#linkding)

Minimal, clean, fast. Browser extension for one-click saving. Tags, search, archive via Wayback Machine. Simpler than Wallabag (which is more of a read-later app). Docker ARM64 image confirmed.

**Alternatives:** LinkWarden (more features, adds screenshot archiving) or Karakeep (AI tagging).

**Replaces:** Browser bookmarks synced to Firefox/Chrome cloud, Pocket, Raindrop.

### 🔴 Notes — Joplin (sync via Nextcloud) or TriliumNext
**Category:** Note-Taking | **Run:** 🖥️ Cluster (TriliumNext server) or sync only via Nextcloud | **ARM64:** ✅ | **Runbook:** [App Catalog](apps-catalog.md#triliumnext-notes)

Two options depending on what you want:

**Option A (Nextcloud already does this):** Install the Nextcloud Notes app. Files stored as Markdown in Nextcloud. Syncs to Joplin on desktop/mobile using the Nextcloud sync server. No new service to deploy. This is the clean path.

**Option B:** **TriliumNext** server (~200–300 MB) if you want a hierarchical knowledge base with backlinks, code blocks, and relational notes. Think Obsidian Publish but private.

**Replaces:** Apple Notes, Google Keep, Notion (partially), Obsidian Sync.

### 🔴 Tasks / To-Do — Vikunja
**Category:** Task Management | **Run:** 🖥️ Cluster | **RAM:** ~150–200 MB | **ARM64:** ✅ Official | **Runbook:** [Vikunja](vikunja.md)

The most complete self-hosted task app: lists, kanban, Gantt, calendar view, teams, labels, due dates, reminders. Has a mobile app and desktop clients. Go binary — very ARM-friendly.

**Replaces:** Todoist, TickTick, Apple Reminders, Notion tasks.

!!! tip "AppFlowy is the Notion alternative"
    Significantly heavier and ARM builds are less polished. Skip unless you specifically want a Notion replacement rather than a task manager.

### 🔴 Finance — Actual Budget
**Category:** Money & Budgeting | **Run:** 🖥️ Cluster | **RAM:** ~100–200 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#actual-budget)

**Actual Budget** is the best-in-class local-first personal finance tool: zero-sum (envelope) budgeting, fast SQLite backend, no forced cloud sync. Syncs via a self-hosted server, and supports automatic bank sync via SimpleFIN (US) or GoCardless (EU/UK). Official mobile apps exist but trail the desktop/web experience.

**Alternative:** **Firefly III** if you prefer double-entry bookkeeping (PHP, more RAM, also ARM64).

**Replaces:** YNAB ($14/mo), Mint (defunct), spreadsheets.

### ⬛ Media Feeder Stack (Arr Suite + Downloader) *(shelved here)*
**Category:** Media Management | **Run:** 💾 NAS or 🖥️ Cluster (pointing at NAS NFS storage) | **ARM64:** ✅ All official | **Runbook:** [Arr Stack](arr-stack.md)

!!! warning "Decision revised — shelved 2026-06-16, never deployed"
    Scoped here as a 🔴 high-priority gap; on reflection it doesn't fit the actual
    need. The library is physical-media-first (ripped straight into Plex, which
    handles its own metadata), and the only downloads wanted are titles the public
    torrent indexers reachable without a private tracker don't carry. Manifests
    were written but never applied. The full reasoning, the revival decision
    tree, and the as-designed shape (**without** a Seerr request portal) live in
    [Arr Stack](arr-stack.md).

This is the automation layer that feeds Plex. Without it, adding new shows/movies/music is manual.

| Service | Purpose | RAM |
|---------|---------|-----|
| **Sonarr** | TV show automation | ~200–400 MB |
| **Radarr** | Movie automation | ~200–400 MB |
| **Lidarr** | Music automation | ~150–300 MB |
| **Prowlarr** | Indexer management (replaces Jackett) | ~100–200 MB |
| ~~**Seerr**~~ (Overseerr/Jellyseerr) | Request portal UI — dropped from the eventual design | ~200–300 MB |
| **qBittorrent** or **SABnzbd** | Actual downloading | ~100–300 MB |

**At 8 GB NAS RAM:** run on the cluster pointing to NFS — all four arr apps handle NFS paths fine and this keeps the NAS comfortable under load.

**At 16 GB NAS RAM:** run everything on the NAS via Docker Compose. This is the better architecture because Sonarr/Radarr can **hardlink** completed downloads directly into the Plex library — a second directory entry pointing at the same inode, with no data copied, rather than a full NFS transfer. Keep all paths under a single root (e.g. `/data/downloads/` and `/data/media/`) so hardlinks work across the same filesystem.

```
/data/
  downloads/complete/    ← qBittorrent drops files here
  media/movies/          ← Radarr library path
  media/tv/              ← Sonarr library path
  media/music/           ← Lidarr library path
```

**Replaces:** Manual downloading, commercial indexer subscriptions, PlexAmp searches that return nothing.

### 🔴 Audiobook + Podcast Server — Audiobookshelf
**Category:** Document Mgmt / Audio | **Run:** 💾 NAS (Docker) | **RAM:** ~200–400 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#audiobookshelf)

Streams all audio formats, syncs progress across devices, has native iOS/Android apps. Covers both audiobooks and podcasts in one app. Library lives next to it on the NAS media share.

**Replaces:** Audible (audiobooks), Pocket Casts/Overcast (podcasts).

### 🔴 E-Book Library — Kavita
**Category:** Document Mgmt / E-Books | **Run:** 🖥️ Cluster | **RAM:** ~100–200 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#kavita)

Web reader + OPDS server for comics, manga, PDF, epub. Clean UI. .NET-based but ARM64 binaries ship officially. Covers your full ebook library via NFS.

**Alternatives:** **Komga** (also excellent, more community, Java but ARM64 confirmed), **Calibre-Web** (if you use Calibre as your library manager on desktop).

**Replaces:** Kindle Unlimited, Libby (still use Libby for library loans), reading on a desktop app.

### 🔴 Personal Dashboard — Homepage (gethomepage)
**Category:** Personal Dashboards | **Run:** 🖥️ Cluster | **RAM:** ~150–200 MB | **ARM64:** ✅ Official | **Runbook:** [Homepage](homepage.md)

YAML-configured, Docker-native, integrates with ~100 services (Plex, Sonarr, Nextcloud, Proxmox, etc.) to show live status and stats. The best-maintained dashboard option. Replaces opening 12 browser tabs.

**Alternative:** **Homarr** has a web-based drag-and-drop editor if you prefer no-YAML config.

**Replaces:** Bookmarking each service URL.

---

## Part 4 — Good Additions (After the Above Are Stable)

### 🟡 Recipe Manager — Mealie
**Category:** Recipe Management | **Run:** 🖥️ Cluster | **RAM:** ~200–300 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#mealie)

Material Design UI, import recipes from any URL, meal planning, shopping list generation. Python + Vue. Very popular in homelab circles.

**Replaces:** Paprika, Yummly, browser recipe tabs, screenshotting recipes.

### 🟡 Wiki — BookStack
**Category:** Wikis | **Run:** 🖥️ Cluster | **RAM:** ~200–300 MB | **ARM64:** ✅ Official | **Runbook:** [BookStack](bookstack.md)

Books → Chapters → Pages hierarchy. Clean editor, search, image embedding, LDAP/SAML auth (ties into Authelia). Ideal for personal runbooks, home documentation, ISP info, car maintenance records, etc.

**Alternative:** **Wiki.js** (more powerful, heavier Node.js). BookStack is the right fit for a personal/family use case.

**Replaces:** Confluence, Notion wiki, Google Docs scattered notes.


### ⬛ Uptime Status Page — Uptime Kuma *(retired here)*
**Category:** Monitoring | **Run:** 🖥️ Cluster | **RAM:** ~150–200 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#uptime-kuma)

Retired from this build (2026-06): for a solo operator a status page has no audience, and the function moved to declarative blackbox-exporter `Probe`s + Prometheus `PublicEndpointDown` alerts → ntfy. Still a fine pick if you want a user-facing status page — monitors HTTP, TCP, ping, DNS and pairs well with ntfy.

**Replaces:** UptimeRobot free tier (sends data to their cloud), StatusCake.

### 🟡 Location History — Dawarich
**Category:** Maps & GPS | **Run:** 🖥️ Cluster | **RAM:** ~300–500 MB | **ARM64:** Verify with `docker manifest inspect`

Privacy-first alternative to Google Timeline. Import your existing Google Location History export, then use OwnTracks on your phone to record ongoing location. You own the data. Useful for travel logging, life events, mileage records.

**Dependency:** Also deploy **OwnTracks Recorder** (C, ~50 MB, ARM64 ✅) for the phone-to-server side.

**Replaces:** Google Timeline / Maps location history.


### 🟡 Chore + Habit Tracker — Donetick
**Category:** Miscellaneous / Task Management | **Run:** 🖥️ Cluster | **RAM:** ~100 MB | **ARM64:** ✅ (v0.1.75 multi-arch) | **Runbook:** [App Catalog](apps-catalog.md#donetick)

Recurring task scheduler built around "due X days after last completion" rather than fixed calendar dates. Scheduling, family sharing, REST API + webhooks, and a native Home Assistant integration. Lightweight Go binary.

**Alternative:** **Habitica** — RPG-gamified habits. Heavier (Node.js + MongoDB). Fun if that motivation style suits you.

**Replaces:** Streaks (iOS), Habitify.

### 🟡 Resume Builder — Reactive Resume
**Category:** Miscellaneous | **Run:** 🖥️ Cluster | **RAM:** ~300–500 MB (app + PostgreSQL + Redis) | **ARM64:** ✅ Official | **Runbook:** [Reactive Resume](reactive-resume.md)

Export-ready, ATS-friendly resume builder. Keep your resume on your own server.

**Replaces:** LinkedIn Resume, Canva, Zety.


### 🟡 Read-Later / Web Archive — Readeck
**Category:** Bookmarks / Archiving | **Run:** 🖥️ Cluster | **RAM:** ~100 MB | **ARM64:** ✅ Official

Saves articles as clean readable copies. Combines bookmark + read-later in one app. Newer and cleaner than Wallabag.

**Alternative:** **Wallabag** — more mature, heavier PHP, also ARM64.

**Replaces:** Pocket, Instapaper, Safari Reading List.

### 🟡 Network Scanner — NetAlertX
**Category:** Network Utilities | **Run:** 🖥️ Cluster or 💾 NAS | **RAM:** ~100 MB | **ARM64:** Verify

Scans your network for unknown devices and alerts you (via ntfy). Useful for IoT VLAN auditing — you'll know immediately when a new device appears.

### 🟡 Wake-on-LAN — Upsnap
**Category:** Network Utilities | **Run:** 🖥️ Cluster | **RAM:** ~50 MB | **ARM64:** ✅

Web dashboard to wake sleeping machines (your desktop, NAS if asleep, etc.) via WOL. Small, Go binary.

### 🟡 AI Assistant — Ollama + Open-WebUI
**Category:** Generative AI | **Run:** 💾 NAS (x86) ⚠️ NAS RAM upgrade required | **Runbook:** [Ollama](ollama.md)

**Ollama** runs LLMs locally. **Open-WebUI** is the ChatGPT-like interface. The real constraint here isn't RAM — it's that the NAS's Pentium Gold 8505 has no usable GPU, so this is **CPU inference**: expect ~10–30 tok/s for a 3B model and meaningfully slower as models grow (Ollama has the numbers).
- Small models (Llama 3.2 3B, Phi-3 mini): ~4–6 GB RAM, usable on CPU
- Larger models (Mistral 7B, Llama 3.1 8B): ~8–16 GB RAM, but slow on this CPU
- ARM CM4 can technically run Ollama but is even slower. NAS x86 is the better of the always-on hosts.

**Only practical after NAS RAM upgrade to 16 GB.**

!!! tip "Consider the desktop instead"
    Your daily-driver desktop (RTX 3080) would run 7–13B models an order of
    magnitude faster than the NAS's CPU. You already deploy **Upsnap** (WOL) in
    this doc — wake the desktop on demand, reach it over Tailscale, and point a
    cluster-hosted Open-WebUI at it. Tradeoff: the desktop isn't always on, so
    it suits interactive use, not 24/7 background tasks.

**Replaces:** ChatGPT subscription (partially — local models are less capable but private).

---

## Part 5 — Category-by-Category Verdict (Full Coverage)

Every category from awesome-selfhosted, with a one-line verdict:

| Category | Verdict | Notes |
|----------|---------|-------|
| **Analytics** | ⚪ Skip | No public-facing site |
| **Archiving / DP** | ✅ Paperless-ngx covers docs; 🟡 ArchiveBox for URLs | ArchiveBox: ~300–500 MB, ARM64 verify |
| **Automation** | ✅ Home Assistant covers this; 🟡 Node-RED or n8n if you want flow-based automation on top | n8n is ARM64 ✅; Huginn is Ruby/heavy |
| **Backup** | ✅ Velero + Garage | Redirects to awesome-sysadmin — already solved |
| **Blogging** | 🟡 Ghost if you want a public blog | Ghost ARM64 ✅, 300–500 MB. WriteFreely for Fediverse |
| **Booking / Scheduling** | ⚪ Skip | Personal use only; Cal.com heavy |
| **Bookmarks** | 🔴 linkding | See Part 3 |
| **Calendar + Contacts** | ✅ Nextcloud (CalDAV/CardDAV) | Already in Nextcloud |
| **Communication — Chat** | 🟡 Matrix/Synapse if family needs | See note below |
| **Communication — Email** | 🚫 Do NOT self-host primary email | See note below |
| **Communication — IRC** | ⚪ Skip | |
| **Communication — SIP** | ⚪ Skip | |
| **Communication — Social / Forums** | ⚪ Skip | Personal use; heavy |
| **Communication — Video Conf** | 🟡 Jitsi Meet if you need it | Jitsi is Java + heavy; often better to use free tier |
| **Communication — XMPP** | ⚪ Skip | Matrix is the better choice if you need chat |
| **Community Supported Agriculture** | ⚪ Skip | |
| **Conference Management** | ⚪ Skip | |
| **CMS** | ⚪ Skip | Nextcloud covers docs; BookStack covers wiki; Ghost covers blog |
| **CRM** | ⚪ Skip | Personal use; Monica (personal CRM) is 🟡 if you track relationships |
| **Database Management** | 🟡 Adminer or pgAdmin | Already in stack via Nextcloud/Paperless; Adminer is ~50 MB |
| **DNS** | 🔴 AdGuard Home | See Part 2 — runs on a dedicated Pi (`pyrite`) |
| **Document Mgmt** | ✅ Paperless-ngx | |
| **Document Mgmt — E-Books** | 🔴 Kavita + 🔴 Audiobookshelf | See Part 3 |
| **Document Mgmt — Library Systems** | ⚪ Skip | Institutional use |
| **E-Commerce** | ⚪ Skip | |
| **Federated Identity / Auth** | 🔴 Authelia | See Part 2 |
| **Feed Readers** | 🔴 Miniflux | See Part 3 |
| **File Transfer — Distributed FS** | ⚪ Skip | NFS already serves the cluster |
| **File Transfer — Object Storage** | ✅ Garage (already deployed) | Runs on the NAS as the S3 backup target for etcd snapshots + Velero (Backups); also available for Nextcloud external storage if needed |
| **File Transfer — P2P** | ⚪ Skip | |
| **File Transfer — Single-click Upload** | 🟡 copyparty if you want a quick file drop endpoint | ~100 MB, Python, ARM64 ✅ |
| **File Transfer — Web File Managers** | 🟡 FileBrowser for NAS web access | ~100 MB, Go, ARM64 ✅ |
| **File Transfer — Sync** | ✅ Nextcloud + 🟡 Syncthing | Syncthing (Go, ARM64 ✅) for P2P sync without server-round-trip — [Syncthing](syncthing.md) |
| **Games** | ⚪ Skip | |
| **Games — Admin Panels** | ⚪ Skip | No game servers in this setup |
| **Genealogy** | ⚪ Skip | |
| **Generative AI** | 🟡 Ollama + Open-WebUI | NAS only, after RAM upgrade |
| **Groupware** | ✅ Nextcloud is your groupware | Don't add a second groupware suite |
| **Health + Fitness** | 🟡 FitTrackee (AGPL, ARM64 verify) | Python app, ~200 MB; good if you track workouts |
| **HRM** | ⚪ Skip | |
| **Identity Mgmt** | 🔴 Authelia + lldap | See Part 2 |
| **IoT** | ✅ Home Assistant | Covers this category entirely |
| **Inventory Mgmt** | 🟡 Snipe-IT or Shelf | Shelf (AGPL, ARM64 verify) for tracking home assets with QR labels |
| **Knowledge Mgmt** | 🔴 TriliumNext or Nextcloud Notes | See Part 3 |
| **Learning / Courses** | ⚪ Skip | Personal use; heavy platforms (Canvas) |
| **Manufacturing** | ⚪ Skip | |
| **Maps + GPS** | 🟡 Dawarich + OwnTracks | See Part 4 |
| **Media Management** | ⬛ Arr stack — shelved, never deployed | See Part 3; [Arr Stack](arr-stack.md) has the full reasoning |
| **Media Streaming — Audio** | ✅ Plex/Plexamp, 🔴 Audiobookshelf | Navidrome redundant — Plexamp covers the same ground |
| **Media Streaming — Multimedia** | ✅ Plex; 🟡 Jellyfin NAS-side if transcoding limits hit | The 8505's Intel Quick Sync handles hardware transcoding for either Plex or Jellyfin |
| **Media Streaming — Video** | ✅ Plex | PeerTube only if you want public video hosting |
| **Miscellaneous** | See individual picks | Habitica/Donetick ✅, Reactive Resume ✅ |
| **Money + Budgeting** | 🔴 Actual Budget | See Part 3 |
| **Monitoring** | ✅ Prometheus/Grafana/Loki + blackbox uptime Probes | |
| **Network Utilities** | 🟡 NetAlertX + Upsnap | See Part 4 |
| **Note-taking + Editors** | 🔴 Joplin via Nextcloud or TriliumNext server | See Part 3 |
| **Office Suites** | 🟡 Nextcloud + Collabora Online | Enable as Nextcloud app; ~500 MB extra — [App Catalog](apps-catalog.md#collabora-online) |
| **Password Managers** | ✅ Vaultwarden | |
| **Pastebins** | 🟡 PrivateBin (~50 MB) if you share code snippets | |
| **Personal Dashboards** | 🔴 Homepage (gethomepage) | See Part 3 |
| **Photo Galleries** | ✅ Immich | |
| **Polls + Events** | 🟡 Rallly (Doodle alt, ~200 MB) if you schedule with others | |
| **Proxy** | ✅ Traefik handles reverse proxy | |
| **Recipe Management** | 🟡 Mealie | See Part 4 |
| **Remote Access** | ✅ Tailscale + 🟡 RustDesk for graphical remote control | RustDesk server: Rust, ~50–100 MB, ARM64 ✅ — [RustDesk](rustdesk.md); Tailscale handles network access, RustDesk handles the screen |
| **Resource Planning** | ⚪ Skip | ERP territory; overkill |
| **Search Engines** | 🟡 SearXNG if you want private web search | ~200 MB, Python, ARM64 ✅ |
| **Self-hosting Solutions** | ✅ Your ArgoCD/k3s stack is this | CasaOS/Tipi are consumer panels; you've outgrown them |
| **Software Dev — API Mgmt** | ⚪ Skip | |
| **Software Dev — CI/CD** | ✅ Woodpecker | |
| **Software Dev — FaaS** | ⚪ Skip | |
| **Software Dev — Feature Toggle** | ⚪ Skip | |
| **Software Dev — IDE + Tools** | 🟡 code-server or Coder for a cloud IDE | Forgejo doesn't provide a hosted IDE; heavy — skip unless you specifically need remote coding |
| **Software Dev — Low Code** | ⚪ Skip | |
| **Software Dev — Project Mgmt** | 🟡 Plane (Jira alt) or Gitea-native if Forgejo issues suffice | |
| **Software Dev — Testing** | ⚪ Skip | |
| **Static Site Generators** | ⚪ Skip | Build tools, not servers |
| **Task Management** | 🔴 Vikunja | See Part 3 |
| **Ticketing** | ⚪ Skip | Personal use; Forgejo issues serve this |
| **Time Tracking** | 🟡 Kimai (~200 MB) if freelance; Wakapi for coding stats | |
| **URL Shorteners** | 🟡 Shlink if you share links publicly | ~100 MB, PHP, ARM64 ✅ |
| **Video Surveillance** | ✅ Ubiquiti Protect (UDM + dedicated drive) | Handled at the network layer; no self-hosted NVR needed |
| **VPN** | ✅ Tailscale | Redirects to awesome-sysadmin — solved |
| **Web Servers** | ✅ Traefik | |
| **Wikis** | 🟡 BookStack | See Part 4 |

---

## Special Topic Notes

### Email — Don't Self-Host Primary

Self-hosting your own mail server (Mailcow, docker-mailserver, Stalwart) is the most common homelab trap:
- Residential/datacenter IPs are blacklisted by default for outbound SMTP
- Deliverability requires DKIM, SPF, DMARC, reverse DNS — all require a clean IP reputation that takes months to build
- Mailcow alone wants ~4–6 GB RAM and is x86-optimised
- One misconfiguration = your emails land in spam at every major provider

**Recommendation:** Use a privacy-focused email provider (Proton Mail, Fastmail, Migadu) for primary. If you want email aliases without exposing your address, **AnonAddy** or **SimpleLogin** (AGPL, ARM64 ✅, ~200 MB) are self-hostable and let you create throwaway addresses that forward to your real inbox. That is the 90% solution.

### Chat — Matrix/Synapse (If You Need It)

If you want a self-hosted chat replacing iMessage/WhatsApp for friends/family: **Matrix Synapse** (Python, ~400–800 MB, ARM64 ✅) + **Element Web** client. It federates so your contacts can be on other Matrix servers.

If it's just for yourself: Signal is fine. Don't deploy a chat server unless you have people who'll use it.


### Collabora Online (Office Suite)

The Nextcloud Collabora integration turns Nextcloud into a full Google Docs replacement. It adds ~500 MB RAM to the cluster but unlocks real-time collaborative editing of `.docx`, `.xlsx`, `.pptx` directly in the browser. Install it as a Nextcloud app (CODE — Collabora Online Development Edition). ARM64 ✅ official.

### Personal CRM — Monica

**Monica** (~300 MB, PHP, ARM64 verify) is a "personal relationship manager" — tracks people you know, notes from conversations, birthdays, how you met them. Useful if you want to be more intentional about maintaining relationships.

---

## Part 6 — Recommended Deployment Order

If starting fresh, add services in this sequence. AdGuard Home goes on the dedicated DNS Pi (`pyrite`) — everything else goes on the cluster unless marked 💾 NAS. Each item notes where it's documented: a full **runbook**, the shared **catalog**, or **none yet**.

```
1.  AdGuard Home              (runbook)        → whole-network ad blocking immediately
2.  Authelia                  (runbook)        → SSO foundation for everything after
3.  Homepage                  (runbook)        → visual dashboard of what you're running
4.  ntfy                      (runbook)        → notifications for everything that follows
5.  Blackbox uptime Probes    (Observability)  → status checks for your public services
6.  Miniflux                  (runbook)        → daily news/RSS replacement
7.  linkding                  (catalog)        → bookmark manager
8.  Actual Budget             (catalog)        → financial tracking
9.  Vikunja                   (runbook)        → task management
10. Donetick                  (catalog)        → recurring chores and habits
11. Arr stack                 (shelved)        → decided against 2026-06-16, see runbook
12. Audiobookshelf            (catalog)        → audiobooks + podcasts
13. Kavita                    (catalog)        → e-book library
14. Mealie                    (catalog)        → recipes
15. BookStack                 (runbook)        → personal wiki
16. Syncthing                 (runbook)        → P2P file sync
17. Collabora (Nextcloud app) (catalog)        → office suite
18. RustDesk                  (runbook)        → graphical remote control
19. Reactive Resume           (runbook)        → resume builder
20. Dawarich + OwnTracks      (none yet)       → location history
21. SearXNG                   (none yet)       → private search engine
22. Ollama + Open-WebUI       (runbook)        → AI assistant (NAS, post-RAM upgrade)
```

---

## Part 7 — NAS Upgrade Path

At 8 GB RAM, your NAS can sustain:
- Plex (0.5–2 GB)
- Immich (2–4 GB)
- **Total: 2.5–6 GB → 2–5.5 GB headroom**

The Arr-stack RAM math below predates its shelving (see Part 3) — kept in case revival is ever back on the table. Ollama is the upgrade's actual live driver today. Upgrading to **16 GB DDR5 SO-DIMM** ($35–50) unlocks:
- Ollama with small models (Phi-3 mini, Llama 3.2 3B)
- Jellyfin as a hardware-transcoding backup to Plex
- Arr stack on NAS, if ever revived (direct disk access, no NFS hop for downloads; at 8 GB it would have been tight during Plex transcode + Immich ML spikes, so cluster+NFS was the planned fallback)

Without the upgrade, Ollama isn't practical and the NAS stays at Plex + Immich (+ Audiobookshelf) only.

---

## Summary: What You're Actually Missing

| Priority | Service | Replaces |
|----------|---------|---------|
| 🔴 High | AdGuard Home | ISP/Google DNS |
| 🔴 High | Authelia | Per-app login friction |
| 🔴 High | ntfy | Paid notification services |
| 🔴 High | Miniflux | Feedly, Google News |
| 🔴 High | linkding | Browser cloud bookmarks |
| 🔴 High | Vikunja | Todoist, TickTick |
| 🔴 High | Actual Budget | YNAB |
| ⬛ Shelved | Arr stack | Manual Plex library management — decided against, see [Arr Stack](arr-stack.md) |
| 🔴 High | Audiobookshelf | Audible, podcast apps |
| 🔴 High | Kavita | Kindle, Calibre desktop |
| 🔴 High | Homepage | Browser tab chaos |
| 🟡 Good | Mealie | Paprika, recipe screenshots |
| 🟡 Good | BookStack | Confluence, scattered docs |
| ⬛ Retired | Uptime Kuma → blackbox Probes | UptimeRobot |
| 🟡 Good | Donetick | Habit and chore tracking |
| 🟡 Good | Dawarich + OwnTracks | Google Timeline |
| 🟡 Good | RustDesk | TeamViewer, AnyDesk |
| 🟡 Good | Ollama + Open-WebUI | ChatGPT (post NAS upgrade) |
| 🟡 Good | Collabora (NC app) | Google Docs |
| 🟡 Good | SearXNG | Google search |
| 🚫 Trap | Email server | — don't; use Proton/Migadu |
