# Arr stack shelved

**Date:** 2026-06
**Status:** Active

## Context

The service selection gap analysis (2026-05) scoped the Arr stack (Sonarr,
Radarr, Prowlarr, qBittorrent + gluetun VPN) as a high-priority gap for
automated media acquisition. Manifests were written and documented in full but
never applied.

## Decision

Shelve the Arr stack. Don't deploy it.

The media library is physical-media-first — ripped straight into Plex, which
handles its own metadata. The only downloads wanted are titles the public
torrent indexers (reachable without a private tracker) don't carry. That doesn't
justify the operational weight of four always-on containers, VPN routing, and
the indexer maintenance that comes with them.

## Consequences

- The manifests and the full deployment runbook remain in `deploy/arr-stack.md`
  with a decision tree for revival.
- Plex continues to serve the library from locally ripped media. No automated
  download pipeline exists.
- The NAS RAM budget freed by not running the Arr stack is available for Ollama
  (the live driver for the 16 GB DDR5 upgrade).
- Revival conditions are documented: if a private tracker invitation or a use
  case beyond physical media changes the equation, the manifests are ready to
  apply.
