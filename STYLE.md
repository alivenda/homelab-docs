# Documentation style guide

How pages in `docs/` are written. The goal is a **wiki you skim**, not a
narrative you read: facts land in tables at the top, prose is reserved for
*why*, and long reference material collapses out of the way. Two pages are the
reference implementations — copy their shape, not just their rules:

- **[`docs/backups.md`](docs/backups.md)** — infrastructure chapter (multi-system, schedules, DR order)
- **[`docs/forgejo.md`](docs/forgejo.md)** — service/app runbook (deploy steps, SSO, verification)

## Page anatomy

Service and app pages, in this order:

1. `# Title` — the service name, nothing else.
2. Status banner: `!!! success "Status — Live"` (or Planned / Shelved / Retired).
3. One-sentence description of what it is and why it's here.
4. **Facts table** — the headerless `| | |` table. Include every fact a
   returning operator looks up: URL, namespace, chart + pinned version,
   storage (class + node pin), auth method, ArgoCD Application names,
   dependencies. If a reader might come back for it, it goes in this table.
5. Body: numbered deploy steps, config artifacts, rationale.
6. `## Verification` — checkbox list of observable outcomes, each with the
   command and its expected output.

Infrastructure chapters (backups, networking, storage…) swap the status banner
for whatever orientation fits, but still open with the facts table and — where
the chapter describes recurring jobs — a **quick-reference table** ("what runs
when") before any prose.

The landing page (`index.md`) is the one **showcase** page: at-a-glance
tables — hardware, the five-repo map, pointers into the catalog — belong
there. Service pages stay operational; the shop-window formatting doesn't
spread.

## Core rules

### Facts go in tables, prose explains why

If a sentence's payload is a value — a port, path, schedule, version, bucket
name — it belongs in a table or a code block, not a paragraph. Prose is for
rationale, trade-offs, and failure modes: the things a table can't carry.
A paragraph that survives the rewrite should answer *why*, not *what*.

The best shape for why-prose is a heading that asks the question the reader
would ask — "Why not a TCPRoute?", "Why Garage, not MinIO" — and answers it
directly. Pre-empt the misconception; don't narrate the decision history.

Keep paragraphs to roughly four rendered lines. No throat-clearing
("In this section we will…"); start with the point.

### Admonitions carry the traps

- `!!! danger` — irreversible loss if ignored (data loss, lockout).
- `!!! warning` — gotchas that cost real debugging time; things that look
  right but aren't.
- `!!! note` — context worth having that would interrupt the flow as prose.
- `!!! tip` — optional shortcuts and nice-to-haves.

Never put a step the reader **must execute** inside an admonition — they read
as skippable. Avoid stacking more than two in a row; if you have three
warnings back to back, the section needs restructuring, not more boxes.

### Collapse long reference material

Use `??? note "…"` / `??? example "…"` (pymdownx.details) for content a
reader only needs while executing, not while skimming: full systemd units,
long config files, command transcripts, sample outputs. Rule of thumb:
**more than ~20 lines that a skimmer scrolls past → collapse it.**

Never collapse: short verification commands, values that differ from
defaults, or anything the reader must notice to avoid a trap.

### Repetition collapses into a pattern

When two or more sections differ only in names, paths, and times, restructure:
state the shared pattern once, put the differences in a delta table, then give
each instance a short subsection holding only its specifics (see the NAS
rclone jobs in `backups.md`). Shared warnings are stated **once**, at the
pattern level — not copied into every instance.

Content tabs (`=== "Label"`) are available for true either/or alternatives
(two ways to do the same step). Don't use them for parallel instances a
reader needs to find via the page TOC or a link — tab labels don't produce
headings or anchors.

### Headings and anchors — the `--strict` contract

`mkdocs build --strict` fails the build on broken internal links **and
anchors**. Heading text generates the anchor, so renaming a heading silently
breaks every inbound link to it. Before renaming any heading:

```sh
grep -rn "pagename.md#" docs/
```

If the old anchor is linked anywhere, pin it on the renamed heading with
`attr_list`:

```markdown
## New heading text { #old-anchor-id }
```

Demoting or promoting a heading (`##` ↔ `###`) keeps its anchor as long as
the text is unchanged — restructure freely, rename carefully.

### This repo is public

- The external password manager holding the age key is always
  "your (externally-hosted) password manager" — never the product name.
- Hostnames in runbook examples use `yourdomain.com` / `auth.yourdomain.com`
  etc.; real usernames and email addresses never appear.
- Internal RFC1918 IPs (`10.0.20.50`, `10.0.20.200`, …) are fine — they're
  meaningless outside the LAN and the runbooks need them to be copy-pasteable.

### Restructuring is not editing

A style rewrite moves facts; it never drops them. Every command, value,
caveat, and rationale in the old page must survive into the new one. If a
fact turns out to be *wrong*, that's an accuracy fix — call it out in the PR
description as its own change, don't fold it silently into the restructure.

## Workflow

1. Branch, then edit. Never commit to `main` — including for one-line fixes.
2. Build locally before pushing:

   ```sh
   pip install -r requirements.txt   # once, in a venv
   mkdocs build --strict
   ```

3. PRs are AGit pushes to Forgejo (not GitHub — that's a read-only mirror):

   ```sh
   git push origin HEAD:refs/for/main -o topic=<short-topic>
   ```

4. The Woodpecker `mkdocs-strict` pipeline runs the same build on the PR;
   it must be green before merge.
