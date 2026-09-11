# mkdocs-redirects dropped

**Date:** 2026-07
**Status:** Active

## Context

When the 30 numbered runbook files (`00-prerequisites.md` through
`29-miniflux.md`) were renamed to slug-based filenames in v20, the
`mkdocs-redirects` plugin was added to keep the old `/NN-name/` URLs resolving.

The plugin pulled an unpinned fork of MkDocs by a departed maintainer into the
dependency tree — four pins instead of three. The site was never published, so
the redirects served no live URL.

## Decision

Drop `mkdocs-redirects`, its pin, and the redirect map.

## Consequences

- `mkdocs build --strict` runs against three dependency pins instead of four.
- The unpinned transitive MkDocs fork is removed from the dependency tree.
- Old `/NN-name/` bookmarks no longer resolve. Because the site was never
  published, no external link breaks.
- If the site is published in the future, any needed redirects can be handled at
  the web-server layer (Traefik middleware or static redirect rules) rather than
  through a build plugin.
