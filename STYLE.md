# Documentation style guide

Two standards govern this repo. The
[Google developer documentation style guide](https://developers.google.com/style)
sets the prose — voice, word choice, headings, procedures. This file sets the page
structure Google doesn't cover: the facts table, the status banner, the admonition
semantics, and the anchor contract that `mkdocs build --strict` enforces.

The goal is a wiki you skim, not a narrative you read. Facts land in tables at the top,
prose explains *why*, and long reference material collapses out of the way. Two pages
are the reference implementations — copy their shape, not just their rules:

- **[`docs/build/backups.md`](docs/build/backups.md)** — infrastructure guide (multi-system, schedules, DR order)
- **[`docs/deploy/forgejo.md`](docs/deploy/forgejo.md)** — service runbook (deploy steps, SSO, verification)

## Prose style

Read the Google guide for the full picture. These are the rules that change almost
every sentence:

- Address the reader as *you*. Use the imperative for instructions.
- Use active voice and present tense. Name who performs each action: "ArgoCD syncs the
  app," not "the app is synced."
- Put the condition, circumstance, or goal before the instruction: "To free the port,
  stop the container."
- Put the most important information first, in the sentence and on the page.
- Use `must`, `can`, or `might`. Avoid `should`, `could`, `may`, and `would` — `should`
  hides whether an action is required.
- Cut filler: *simply*, *easy*, *just*, *please*, *note that*, *in order to*, *via*,
  *leverage*, *utilize*, *allows you to*, *e.g.*, *i.e.*
- Don't claim more than you verified. Avoid *best*, *fastest*, *guarantees*. A
  prohibition is not a claim — "never put ForwardAuth in front of an app with its own
  API clients" stays.
- Keep sentences under about 26 words and paragraphs to one idea.
- Use contractions, especially negations: *don't*, *isn't*, *can't*.
- Write timelessly. Drop *currently*, *now*, *new*, *latest*, and *soon* when describing
  how something works. A live reading — "the pod is in `CrashLoopBackOff`" — is fine.

### Headings

Use sentence case. Capitalize only the first word and proper nouns, and don't end a
heading with a period.

Don't number a heading to convey sequence. `## Install the NFS provisioner` beats
`## Step 7: Install NFS Storage Provisioner` — page order already carries the sequence,
and a numbered heading breaks the moment a step is inserted. Where the steps are a true
sequence, put them in one numbered list under a single task heading.

Numbers that identify rather than order are fine: `## Scenario 4 — ruby, the control
plane` names a failure mode, and `## v21` names a release.

Task headings start with a bare infinitive ("Configure the gateway"). Conceptual
headings are noun phrases. A heading that asks the reader's own question —
"Why Garage, not MinIO?" — is the best shape for why-prose. Pre-empt the misconception;
don't narrate the decision history.

### Code font

Use code font for what the reader types verbatim or reads back from a machine:
filenames, commands, flags, values, class and method names, and command output.

Don't use code font for product names, domain names, IP addresses, or URLs. Write
10.0.20.50 as plain text and a browser URL as a plain link. Add a qualifying noun to a
bare keyword: "the `values.yaml` file," not "`values.yaml`."

Bold is for UI elements and run-in headings. Italics introduce a term. Nothing else
gets either.

## Page anatomy

Service and app pages, in this order:

1. `# Title` — the service name, nothing else.
2. Status banner: `!!! success "Status — Live"` (or Planned / Shelved / Retired).
3. One-sentence description of what it is and why it's here.
4. **Facts table** — the headerless `| | |` table. Include every fact a returning
   operator looks up: URL, namespace, chart and pinned version, storage (class and node
   pin), auth method, ArgoCD `Application` names, dependencies.
5. Body: deploy steps, config artifacts, rationale.
6. `## Verification` — a checkbox list of observable outcomes, each with its command and
   expected output.

Infrastructure guides (backups, network, storage) swap the status banner for whatever
orientation fits, but still open with the facts table and — where the guide describes
recurring jobs — a quick-reference table before any prose.

The landing page (`docs/index.md`) is the one showcase page. At-a-glance tables for the
hardware build, the repo map, and the catalog belong there. Service pages stay
operational; the shop-window formatting doesn't spread.

## Facts go in tables, prose explains why

If a sentence's payload is a value — a port, path, schedule, version, bucket name — it
belongs in a table or a code block, not a paragraph. Prose carries rationale,
trade-offs, and failure modes: the things a table can't hold. A paragraph that survives
the rewrite answers *why*, not *what*.

Keep paragraphs to roughly four rendered lines. Start with the point; no throat-clearing.

## Admonitions carry the traps

- `!!! danger` — irreversible loss if ignored (data loss, lockout).
- `!!! warning` — gotchas that cost real debugging time; things that look right but aren't.
- `!!! note` — context that belongs on the page but breaks the flow as prose.
- `!!! tip` — optional shortcuts and nice-to-haves.

Never put a step the reader must execute inside an admonition — they read as skippable.
A mandatory requirement isn't a tip: `!!! tip "HTTPS is mandatory"` belongs in the body
or in a warning. Avoid stacking more than two in a row; three warnings back to back mean
the section needs restructuring, not more boxes.

Give every admonition a title. An untitled box makes the reader open it to find out
whether it matters.

## Collapse long reference material

Use `??? note "…"` or `??? example "…"` (pymdownx.details) for content a reader needs
only while executing: full systemd units, long config files, command transcripts, sample
output. Rule of thumb: more than about 20 lines that a skimmer scrolls past, collapse it.

Never collapse short verification commands, values that differ from defaults, or
anything the reader must notice to avoid a trap.

## Repetition collapses into a pattern

When two or more sections differ only in names, paths, and times, restructure: state the
shared pattern once, put the differences in a delta table, then give each instance a
short subsection holding only its specifics. State shared warnings once, at the pattern
level.

One procedure gets one home. The `kubeseal` pipeline lives in
[`docs/deploy/index.md`](docs/deploy/index.md); OIDC client registration lives with
Authelia. Link to them rather than restating them.

Content tabs (`=== "Label"`) are for true either/or alternatives. Don't use them for
parallel instances a reader needs to find in the page TOC — tab labels produce no
headings and no anchors.

## Headings and anchors — the `--strict` contract

`mkdocs build --strict` fails the build on broken internal links **and anchors**. Heading
text generates the anchor, so renaming a heading breaks every inbound link to it. Before
renaming any heading:

```sh
grep -rn "pagename.md#" docs/
```

If the old anchor is linked anywhere, pin it on the renamed heading with `attr_list`:

```markdown
## New heading text { #old-anchor-id }
```

Demoting or promoting a heading (`##` to `###`) keeps its anchor as long as the text is
unchanged. Restructure freely; rename carefully.

Renaming a page title is a three-part change: the `# H1`, the `mkdocs.yml` nav entry, and
every cross-reference that uses the old name as link text. Doing one or two of the three
leaves the docs contradicting themselves.

## This repo is public

- The external password manager holding the age key is always "your (externally-hosted)
  password manager" — never the product name.
- Hostnames in examples use `yourdomain.com` and `auth.yourdomain.com`. Real usernames
  and email addresses never appear.
- Internal RFC1918 addresses (10.0.20.50, 10.0.20.200) are fine. They're meaningless
  outside the LAN and the runbooks need them.

## Restructuring is not editing

A style rewrite moves facts; it never drops them. Every command, value, caveat, and
rationale in the old page survives into the new one. If a fact turns out to be *wrong*,
that's an accuracy fix — call it out in the PR description as its own change rather than
folding it silently into the restructure.

To check that nothing vanished, compare fenced-code-block counts before and after:

```sh
git show main:docs/build/backups.md | grep -c '^```'
grep -c '^```' docs/build/backups.md
```

## Workflow

1. Branch, then edit. Never commit to `main`, including for one-line fixes.
2. Build locally before pushing:

   ```fish
   python -m venv .venv                      # once
   source .venv/bin/activate.fish
   pip install -r requirements.txt
   mkdocs build --strict
   ```

3. Optional: lint the prose. CI runs this too, so a local run just shortens the loop:

   ```fish
   sudo pacman -S vale                       # once
   vale sync                                 # fetches the Google package
   vale --minAlertLevel=warning docs/
   ```

4. Open the PR with an AGit push to Forgejo. GitHub is a read-only mirror:

   ```fish
   git push origin HEAD:refs/for/main -o topic=<short-topic>
   ```

   The topic is the PR's identity. Reusing it updates that PR; a different topic opens a
   duplicate.

5. The Woodpecker pipeline runs `mkdocs build --strict` and `vale`. The build must be
   green before merge.

### What Vale checks, and what it doesn't

`.vale.ini` loads Vale's official Google package plus the project-local `Homelab` style
in `styles/Homelab/`. The Homelab rules cover three things the Google package misses:
`should`/`could`/`may`/`would`, the filler word list, and sentence-case headings with
this project's product names exempted.

Vale is advisory while the style rewrite is in flight — the `vale` step in
`.woodpecker.yml` carries `failure: ignore`. It becomes blocking once the corpus reaches
zero warnings.

Four Google rules are turned off or downgraded in `.vale.ini`, each with its measured hit
count in a comment. Vale can't judge whether a sentence is *true*, whether a fact belongs
in a table, or whether a page tells a story instead of describing a system. Those stay a
human job.
