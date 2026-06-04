# Personal Cloud Stack — Complete Service Guide

A comprehensive review of every awesome-selfhosted category mapped against your specific hardware, existing stack, and day-to-day needs. Written against the Turing Pi 2 cluster, UGREEN DXP6800 Pro NAS, slate (a Late-2014 Mac mini running Home Assistant OS as a Proxmox VM), and 2× Pi Zero 2 W DNS nodes.

---

## Hardware Context

| Platform | Specs | Already Running |
|----------|-------|-----------------|
| **Cluster (4× CM4)** | ARM64, 32 GB total RAM, 1 TB NFS | Traefik, ArgoCD, Prometheus/Grafana/Loki, Vaultwarden, Nextcloud, Paperless-ngx, Forgejo, Woodpecker |
| **NAS (DXP6800 Pro)** | x86-64, **8 GB RAM** (expandable) | Plex, Immich |
| **Home Assistant node (slate — Mac mini)** | x86-64, 16 GB RAM / 256 GB SSD, Proxmox | Home Assistant OS (VM: 2 vCPU, 4 GB) |
| **DNS nodes (2× Pi Zero 2 W)** | ARM64, 512 MB RAM each | AdGuard Home (primary + secondary) |

> **NAS RAM is a meaningful constraint.** At 8 GB, Plex + Immich can consume 2.5–6 GB under load, leaving 2–5.5 GB headroom. Enough to run the Arr stack locally (if you prefer), but not enough for Ollama without a RAM upgrade. The DXP6800 Pro accepts standard DDR5 SO-DIMMs — upgrading to 16 GB (~$35–50) opens up AI workloads and makes the NAS more comfortable overall. This doc marks services that require the upgrade `⚠️ NAS RAM upgrade recommended`.

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
| 🖥️ **Cluster** | Run on k3s |
| 💾 **NAS** | Run via Docker Compose on DXP6800 |
| 🌐 **DNS nodes** | Run on Pi Zero 2 W (primary/secondary) |
| ⚠️ | Needs NAS RAM upgrade to run comfortably |
| 🚫 | Trap — don't self-host this |

---

## Part 1 — What You Already Have

| Service | Tool | Notes |
|---------|------|-------|
| File sync + Calendar + Contacts | Nextcloud ✅ | R13; also covers Notes app, Bookmarks app, Collabora |
| Document archive | Paperless-ngx ✅ | R14 |
| Photo/video library | Immich ✅ | R15 (NAS) |
| Smart home hub | Home Assistant ✅ | R16 (slate — Mac mini / Proxmox VM, Home Assistant OS) |
| Password manager + TOTP | Vaultwarden ✅ | R7; also generates 2FA codes natively — no separate app needed. If you want strict separation between passwords and 2FA, [2FAuth](https://2fauth.app) is the self-hosted option. |
| Media streaming | Plex ✅ | NAS |
| Music streaming | Plex ✅ | NAS — Plex Media Server serves audio; Plexamp is the client player, not a separate daemon |
| Git hosting | Forgejo ✅ | R11 |
| CI/CD | Woodpecker ✅ | R12 |
| GitOps | ArgoCD ✅ | R5 |
| Monitoring + metrics | Prometheus + Grafana + Loki ✅ | R9 |
| Backups | Restic + Velero ✅ | R10 |
| Object storage (backup target) | Garage ✅ | R10 — S3 store for etcd snapshots + Velero, runs on the NAS |
| Secrets management | Sealed Secrets ✅ | Cluster — encrypts secrets committed to Git for ArgoCD |
| VPN / remote access | Tailscale ✅ | R2 |
| HTTPS proxy | Traefik ✅ | R6 |

---

## Part 2 — Foundational Layer (Do These First)

These unlock everything else. Add them before the "good additions" below.

### 🔴 DNS Ad-Blocking — AdGuard Home
**Category:** DNS | **Run:** 🌐 2× Pi Zero 2 W (primary + secondary) | **RAM:** ~100 MB each | **ARM64:** ✅ Official multiarch | **Runbook:** [R17](17-adguard-home.md)

Running on dedicated Pi Zero 2 Ws is the correct pattern: DNS must stay up independently of your cluster, and two units give you redundancy. AdGuard Home has *no* native cross-instance sync — R17 runs the [`adguardhome-sync`](https://github.com/bakito/adguardhome-sync) container on the primary to keep the secondary's blocklists and config in lockstep.

The Pi Zero 2 W has 512 MB RAM — AdGuard Home uses ~100 MB, leaving plenty of headroom. Install via the official one-line installer on DietPi or Raspberry Pi OS Lite.

**Setup:** Configure your UDM to advertise both Pi Zero IPs as DNS servers via DHCP on each VLAN. Handing clients two resolvers gives only *partial* failover, though — stub resolvers retry inconsistently and can stall on a dead server for several seconds. For true HA, front the pair with a shared [keepalived](https://www.keepalived.org/) VIP. Note also that the Pi Zero 2 W is Wi-Fi-only (no Ethernet port); for DNS infrastructure a USB-Ethernet dongle is worth considering.

**Replaces:** Google's DNS, your ISP's DNS, in-browser ad blockers.

### 🔴 SSO / Auth Layer — Authelia
**Category:** Federated Identity | **Run:** 🖥️ Cluster | **RAM:** ~100–150 MB | **ARM64:** ✅ Official | **Runbook:** [R18](18-authelia.md)

You already have Traefik. Authelia adds a single login + 2FA layer, wired two different ways depending on the app (see R18):
- **OIDC** for apps with their own login (Nextcloud, Paperless, Forgejo, Vikunja, …) — they redirect to Authelia and manage their own session. **Do not** put forward-auth in front of these: their API clients (desktop sync, git CLI, mobile apps) send credentials directly and can't follow the redirect.
- **ForwardAuth** (Traefik middleware) only for apps with *no* login of their own (e.g. Homepage).
- 2FA (TOTP, WebAuthn) and single sign-on across all of the above in one place.
- LDAP-compatible user store via **lldap** (Rust, ~30 MB, ARM64 ✅ — lightweight LDAP backend), or flat file for simplicity.

**Why before everything else:** every new service you add will want auth. Doing it now means each future service gets SSO for free.

> **Authentik** is the heavier alternative (~600 MB Python). **Authelia** (Go, ~100 MB) is the right fit for your cluster.

### 🔴 Push Notifications — ntfy
**Category:** Communication | **Run:** 🖥️ Cluster | **RAM:** ~50 MB | **ARM64:** ✅ Official | **Runbook:** [R19](19-ntfy.md)

Self-hosted push notifications for everything: backup alerts, Woodpecker pipeline results, Home Assistant automations, Paperless document ingested, etc. Replaces paid notification services. Native Android/iOS app. Integrates with Home Assistant, Grafana, shell scripts.

---

## Part 3 — High Priority Gaps (Daily Drivers)

These are the services people use every day that currently rely on commercial clouds.

### 🔴 RSS / News Reader — FreshRSS or Miniflux
**Category:** Feed Readers | **Run:** 🖥️ Cluster | **RAM:** ~100 MB | **ARM64:** ✅ Both official | **Runbook:** [App Catalog](apps-catalog.md#freshrss)

**FreshRSS** — PHP, feature-rich, mobile-friendly, supports Fever/Google Reader API (works with Reeder, NetNewsWire, etc.)
**Miniflux** — Go binary, minimal, fast, Fever + Google Reader API, very low RAM.

Pick Miniflux if you prefer lightweight and fast. Pick FreshRSS if you want more features. Both are excellent.

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
**Category:** Task Management | **Run:** 🖥️ Cluster | **RAM:** ~150–200 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#vikunja)

The most complete self-hosted task app: lists, kanban, Gantt, calendar view, teams, labels, due dates, reminders. Has a mobile app and desktop clients. Go binary — very ARM-friendly.

**Replaces:** Todoist, TickTick, Apple Reminders, Notion tasks.

> **AppFlowy** is the Notion alternative but is significantly heavier and ARM builds are less polished. Skip unless you specifically want a Notion replacement rather than a task manager.

### 🔴 Finance — Actual Budget
**Category:** Money & Budgeting | **Run:** 🖥️ Cluster | **RAM:** ~100–200 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#actual-budget)

**Actual Budget** is the best-in-class local-first personal finance tool: zero-sum (envelope) budgeting, fast SQLite backend, no forced cloud sync. Syncs via a self-hosted server, and supports automatic bank sync via SimpleFIN (US) or GoCardless (EU/UK). Official mobile apps exist but trail the desktop/web experience.

**Alternative:** **Firefly III** if you prefer double-entry bookkeeping (PHP, more RAM, also ARM64).

**Replaces:** YNAB ($14/mo), Mint (defunct), spreadsheets.

### 🔴 Media Feeder Stack (Arr Suite + Downloader)
**Category:** Media Management | **Run:** 💾 NAS or 🖥️ Cluster (pointing at NAS NFS storage) | **ARM64:** ✅ All official | **Runbook:** [R21](21-arr-stack.md)

This is the automation layer that feeds Plex. Without it, adding new shows/movies/music is manual.

| Service | Purpose | RAM |
|---------|---------|-----|
| **Sonarr** | TV show automation | ~200–400 MB |
| **Radarr** | Movie automation | ~200–400 MB |
| **Lidarr** | Music automation | ~150–300 MB |
| **Prowlarr** | Indexer management (replaces Jackett) | ~100–200 MB |
| **Seerr** (Overseerr/Jellyseerr) | Request portal UI | ~200–300 MB |
| **qBittorrent** or **SABnzbd** | Actual downloading | ~100–300 MB |

**At 8 GB NAS RAM:** run on the cluster pointing to NFS — all four arr apps handle NFS paths fine and this keeps the NAS comfortable under load.

**At 16 GB NAS RAM (recommended):** run everything on the NAS via Docker Compose. This is the better architecture because Sonarr/Radarr can **hardlink** completed downloads directly into the Plex library — a second directory entry pointing at the same inode, with no data copied, rather than a full NFS transfer. Keep all paths under a single root (e.g. `/data/downloads/` and `/data/media/`) so hardlinks work across the same filesystem.

```
/data/
  downloads/complete/    ← qBittorrent drops files here
  media/movies/          ← Radarr library path
  media/tv/              ← Sonarr library path
  media/music/           ← Lidarr library path
```

**Replaces:** Manual downloading, commercial indexer subscriptions, PlexAmp searches that return nothing.

### 🔴 Audiobook + Podcast Server — Audiobookshelf
**Category:** Document Mgmt / Audio | **Run:** 🖥️ Cluster or 💾 NAS | **RAM:** ~200–400 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#audiobookshelf)

Streams all audio formats, syncs progress across devices, has native iOS/Android apps. Covers both audiobooks and podcasts in one app. Backed by your NFS storage.

**Replaces:** Audible (audiobooks), Pocket Casts/Overcast (podcasts).

### 🔴 E-Book Library — Kavita
**Category:** Document Mgmt / E-Books | **Run:** 🖥️ Cluster | **RAM:** ~100–200 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#kavita)

Web reader + OPDS server for comics, manga, PDF, epub. Clean UI. .NET-based but ARM64 binaries ship officially. Covers your full ebook library via NFS.

**Alternatives:** **Komga** (also excellent, more community, Java but ARM64 confirmed), **Calibre-Web** (if you use Calibre as your library manager on desktop).

**Replaces:** Kindle Unlimited, Libby (still use Libby for library loans), reading on a desktop app.

### 🔴 Personal Dashboard — Homepage (gethomepage)
**Category:** Personal Dashboards | **Run:** 🖥️ Cluster | **RAM:** ~150–200 MB | **ARM64:** ✅ Official | **Runbook:** [R20](20-homepage.md)

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
**Category:** Wikis | **Run:** 🖥️ Cluster | **RAM:** ~200–300 MB | **ARM64:** ✅ Official | **Runbook:** [R22](22-bookstack.md)

Books → Chapters → Pages hierarchy. Clean editor, search, image embedding, LDAP/SAML auth (ties into Authelia). Ideal for personal runbooks, home documentation, ISP info, car maintenance records, etc.

**Alternative:** **Wiki.js** (more powerful, heavier Node.js). BookStack is the right fit for a personal/family use case.

**Replaces:** Confluence, Notion wiki, Google Docs scattered notes.


### 🟡 Uptime Status Page — Uptime Kuma
**Category:** Monitoring | **Run:** 🖥️ Cluster | **RAM:** ~150–200 MB | **ARM64:** ✅ Official | **Runbook:** [App Catalog](apps-catalog.md#uptime-kuma)

Monitors HTTP, TCP, ping, DNS for all your services and shows a clean status page. Different job than Prometheus — this is user-facing "is it up?" vs infrastructure metrics. Pairs well with ntfy for alerts.

**Replaces:** UptimeRobot free tier (sends data to their cloud), StatusCake.

### 🟡 Location History — Dawarich
**Category:** Maps & GPS | **Run:** 🖥️ Cluster | **RAM:** ~300–500 MB | **ARM64:** Verify with `docker manifest inspect`

Privacy-first alternative to Google Timeline. Import your existing Google Location History export, then use OwnTracks on your phone to record ongoing location. You own the data. Useful for travel logging, life events, mileage records.

**Dependency:** Also deploy **OwnTracks Recorder** (C, ~50 MB, ARM64 ✅) for the phone-to-server side.

**Replaces:** Google Timeline / Maps location history.


### 🟡 Chore + Habit Tracker — Donetick
**Category:** Miscellaneous / Task Management | **Run:** 🖥️ Cluster | **RAM:** ~100 MB | **ARM64:** Verify with `docker manifest inspect` | **Runbook:** [App Catalog](apps-catalog.md#donetick)

Recurring task scheduler built around "due X days after last completion" rather than fixed calendar dates. Scheduling, family sharing, REST API + webhooks, and a native Home Assistant integration. Lightweight Go binary.

**Alternative:** **Habitica** — RPG-gamified habits. Heavier (Node.js + MongoDB). Fun if that motivation style suits you.

**Replaces:** Streaks (iOS), Habitify.

### 🟡 Resume Builder — Reactive Resume
**Category:** Miscellaneous | **Run:** 🖥️ Cluster | **RAM:** ~300–500 MB (app + PostgreSQL + Redis) | **ARM64:** ✅ Official | **Runbook:** [R25](25-reactive-resume.md)

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
**Category:** Generative AI | **Run:** 💾 NAS (x86) ⚠️ NAS RAM upgrade required | **Runbook:** [R26](26-ollama.md)

**Ollama** runs LLMs locally. **Open-WebUI** is the ChatGPT-like interface. The real constraint here isn't RAM — it's that the NAS's Pentium Gold 8505 has no usable GPU, so this is **CPU inference**: expect ~10–30 tok/s for a 3B model and meaningfully slower as models grow (R26 has the numbers).
- Small models (Llama 3.2 3B, Phi-3 mini): ~4–6 GB RAM, usable on CPU
- Larger models (Mistral 7B, Llama 3.1 8B): ~8–16 GB RAM, but slow on this CPU
- ARM CM4 can technically run Ollama but is even slower. NAS x86 is the better of the always-on hosts.

**Only practical after NAS RAM upgrade to 16 GB.**

> **Consider the desktop instead.** Your daily-driver desktop (RTX 3080) would run 7–13B models an order of magnitude faster than the NAS's CPU. You already deploy **Upsnap** (WOL) in this doc — wake the desktop on demand, reach it over Tailscale, and point a cluster-hosted Open-WebUI at it. Tradeoff: the desktop isn't always on, so it suits interactive use, not 24/7 background tasks.

**Replaces:** ChatGPT subscription (partially — local models are less capable but private).

---

## Part 5 — Category-by-Category Verdict (Full Coverage)

Every category from awesome-selfhosted, with a one-line verdict:

| Category | Verdict | Notes |
|----------|---------|-------|
| **Analytics** | ⚪ Skip | No public-facing site |
| **Archiving / DP** | ✅ Paperless-ngx covers docs; 🟡 ArchiveBox for URLs | ArchiveBox: ~300–500 MB, ARM64 verify |
| **Automation** | ✅ Home Assistant covers this; 🟡 Node-RED or n8n if you want flow-based automation on top | n8n is ARM64 ✅; Huginn is Ruby/heavy |
| **Backup** | ✅ Restic + Velero | Redirects to awesome-sysadmin — already solved |
| **Blogging** | 🟡 Ghost if you want a public blog | Ghost ARM64 ✅, 300–500 MB. WriteFreely for Fediverse |
| **Booking / Scheduling** | ⚪ Skip | Personal use only; Cal.com heavy |
| **Bookmarks** | 🔴 linkding | See Part 3 |
| **Calendar + Contacts** | ✅ Nextcloud (CalDAV/CardDAV) | Already in R13 |
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
| **DNS** | 🔴 AdGuard Home | See Part 2 — runs on 2× Pi Zero 2 W |
| **Document Mgmt** | ✅ Paperless-ngx | |
| **Document Mgmt — E-Books** | 🔴 Kavita + 🔴 Audiobookshelf | See Part 3 |
| **Document Mgmt — Library Systems** | ⚪ Skip | Institutional use |
| **E-Commerce** | ⚪ Skip | |
| **Federated Identity / Auth** | 🔴 Authelia | See Part 2 |
| **Feed Readers** | 🔴 FreshRSS or Miniflux | See Part 3 |
| **File Transfer — Distributed FS** | ⚪ Skip | NFS already serves the cluster |
| **File Transfer — Object Storage** | ✅ Garage (already deployed) | Runs on the NAS as the S3 backup target for etcd snapshots + Velero (R10); also available for Nextcloud external storage if needed |
| **File Transfer — P2P** | ⚪ Skip | |
| **File Transfer — Single-click Upload** | 🟡 copyparty if you want a quick file drop endpoint | ~100 MB, Python, ARM64 ✅ |
| **File Transfer — Web File Managers** | 🟡 FileBrowser for NAS web access | ~100 MB, Go, ARM64 ✅ |
| **File Transfer — Sync** | ✅ Nextcloud + 🟡 Syncthing | Syncthing (Go, ARM64 ✅) for P2P sync without server-round-trip — [R23](23-syncthing.md) |
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
| **Media Management** | 🔴 Arr stack (Sonarr/Radarr/Lidarr/Prowlarr + Seerr) | See Part 3 |
| **Media Streaming — Audio** | ✅ Plex/Plexamp, 🔴 Audiobookshelf | Navidrome redundant — Plexamp covers the same ground |
| **Media Streaming — Multimedia** | ✅ Plex; 🟡 Jellyfin NAS-side if transcoding limits hit | The 8505's Intel Quick Sync handles hardware transcoding for either Plex or Jellyfin |
| **Media Streaming — Video** | ✅ Plex | PeerTube only if you want public video hosting |
| **Miscellaneous** | See individual picks | Habitica/Donetick ✅, Reactive Resume ✅ |
| **Money + Budgeting** | 🔴 Actual Budget | See Part 3 |
| **Monitoring** | ✅ Prometheus/Grafana/Loki + 🟡 Uptime Kuma | |
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
| **Remote Access** | ✅ Tailscale + 🟡 RustDesk for graphical remote control | RustDesk server: Rust, ~50–100 MB, ARM64 ✅ — [R24](24-rustdesk.md); Tailscale handles network access, RustDesk handles the screen |
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

If starting fresh, add services in this sequence. AdGuard Home goes on the Pi Zero 2 Ws — everything else goes on the cluster unless marked 💾 NAS. Items tagged `(Rxx)` have a runbook; untagged items are recommended but not yet documented.

```
1.  AdGuard Home              (R17) → whole-network ad blocking immediately
2.  Authelia                  (R18) → SSO foundation for everything after
3.  Homepage                  (R20) → visual dashboard of what you're running
4.  ntfy                      (R19) → notifications for everything that follows
5.  Uptime Kuma               (R33) → status monitoring for your services
6.  FreshRSS / Miniflux       (R21) → daily news/RSS replacement
7.  linkding                  (R22) → bookmark manager
8.  Actual Budget             (R26) → financial tracking
9.  Vikunja                   (R24) → task management
10. Donetick                  (R25) → recurring chores and habits
11. Arr stack + Seerr         (R21) → Plex automation (Sonarr/Radarr/Lidarr/Prowlarr)
12. Audiobookshelf            (R28) → audiobooks + podcasts
13. Kavita                    (R29) → e-book library
14. Mealie                    (R30) → recipes
15. BookStack                 (R22) → personal wiki
16. Syncthing                 (R23) → P2P file sync
17. Collabora (Nextcloud app) (R34) → office suite
18. RustDesk                  (R24) → graphical remote control
19. Reactive Resume           (R25) → resume builder
20. Dawarich + OwnTracks            → location history (no runbook yet)
21. SearXNG                         → private search engine (no runbook yet)
22. Ollama + Open-WebUI       (R26) → AI assistant (NAS, post-RAM upgrade)
```

---

## Part 7 — NAS Upgrade Path

At 8 GB RAM, your NAS can sustain:
- Plex (0.5–2 GB)
- Immich (2–4 GB)
- **Total: 2.5–6 GB → 2–5.5 GB headroom**

The Arr stack (1–1.5 GB total) *can* run on the NAS at 8 GB, but it will be tight during Plex transcode + Immich ML processing spikes. Running arr on the cluster (pointing to NFS) is safer and keeps the NAS headroom available. Upgrading to **16 GB DDR5 SO-DIMM** ($35–50) makes everything comfortable and unlocks:
- Arr stack on NAS (direct disk access, no NFS hop for downloads)
- Ollama with small models (Phi-3 mini, Llama 3.2 3B)
- Jellyfin as a hardware-transcoding backup to Plex

Until then, run the Arr stack on the cluster pointing at NFS — it works fine.

---

## Summary: What You're Actually Missing

| Priority | Service | Replaces |
|----------|---------|---------|
| 🔴 High | AdGuard Home | ISP/Google DNS |
| 🔴 High | Authelia | Per-app login friction |
| 🔴 High | ntfy | Paid notification services |
| 🔴 High | FreshRSS or Miniflux | Feedly, Google News |
| 🔴 High | linkding | Browser cloud bookmarks |
| 🔴 High | Vikunja | Todoist, TickTick |
| 🔴 High | Actual Budget | YNAB |
| 🔴 High | Arr stack + Seerr | Manual Plex library management |
| 🔴 High | Audiobookshelf | Audible, podcast apps |
| 🔴 High | Kavita | Kindle, Calibre desktop |
| 🔴 High | Homepage | Browser tab chaos |
| 🟡 Good | Mealie | Paprika, recipe screenshots |
| 🟡 Good | BookStack | Confluence, scattered docs |
| 🟡 Good | Uptime Kuma | UptimeRobot |
| 🟡 Good | Donetick | Habit and chore tracking |
| 🟡 Good | Dawarich + OwnTracks | Google Timeline |
| 🟡 Good | RustDesk | TeamViewer, AnyDesk |
| 🟡 Good | Ollama + Open-WebUI | ChatGPT (post NAS upgrade) |
| 🟡 Good | Collabora (NC app) | Google Docs |
| 🟡 Good | SearXNG | Google search |
| 🚫 Trap | Email server | — don't; use Proton/Migadu |
