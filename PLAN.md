# PLAN — status-lectures

Implementation plan for [QuantEcon/status-lectures#1](https://github.com/QuantEcon/status-lectures/issues/1),
part of the centralized environment & config reporting initiative
([QuantEcon/meta#321](https://github.com/QuantEcon/meta/issues/321)).

This repo is the **dashboard** (and, later, the history store — see *Future
improvements*). The data **producer** lives in
[QuantEcon/actions](https://github.com/QuantEcon/actions)
([QuantEcon/actions#30](https://github.com/QuantEcon/actions/issues/30)); the
two share a versioned schema. Design input from the May–June 2026 manual
tooling audits is in
[docs/tooling-audit-notes-2026-06.md](docs/tooling-audit-notes-2026-06.md).

> **Process:** each phase below is reviewed in detail (implementation
> approach agreed with @mmcky) before any code is written.

## Architecture: one source, pull-based

`QuantEcon/actions` is in active development and is the planned future for
unified lecture configuration. There is **a single data path**, and it is
**pull, not push**:

```
lecture repo publish (publish-gh-pages, QuantEcon/actions)
        │  writes qe.status.json into the site artifact it already deploys
        ▼
https://<series-site>/qe.status.json    (e.g. intro.quantecon.org/qe.status.json)
        │
        │ fetched live, client-side
        ▼
   Pages dashboard
```

Design properties:

- **The manifest is just a static file on each lecture site.** The producer's
  entire job is to write one JSON file into the `_build/html/` artifact that
  `publish-gh-pages` already deploys. No `repository_dispatch`, no payload
  caps, no cross-repo credentials, no intake workflow, no commit races.
- **No new credentials anywhere.** Manifests are public URLs and the
  dashboard is a static page — nothing in this repo needs write access to
  anything.
- **No scrape/backstop pipeline** (supersedes the "daily backstop" in
  meta#321): the publish workflow has the full repo checked out, so
  workflow/action pins, `dependabot.yml`, and specifier strings are collected
  by the producer in the same run — plus the effective resolved versions only
  a real build can know. A series that doesn't publish a manifest renders as
  **"awaiting migration"**; the dashboard doubles as the migration tracker.
- **History is a bolt-on, not core.** A single-purpose scheduled action can
  later collect manifests into `data/history/` for week-over-week diffs (see
  *Future improvements*) — it copies what the builds wrote, never synthesizes
  data. Until then this repo has no scheduled jobs at all.
- "Declared vs effective" drift is an **intra-record** check: the manifest
  carries both the specifier (from `environment.yml`) and the resolved
  version (from the built env), from one build.

## The dashboard

A **single table for quick cross-comparison** of all lecture series:

- **Columns:** one per series (from the registry in `data/index.json`);
  translations render as sub-columns grouped under their parent (see
  *Monitored data* g).
- **Rows:** one per tracked field — last publish (date + "X days old"),
  lecture execution statistics (documents run / failed, run time — the
  aggregate of each site's `status.html`, linked), runner/container, versions
  (python, anaconda pin, jupyter-book, theme, sphinx extensions), execution
  settings, features, hygiene flags.
- **Row is green when all series match**; divergent rows are highlighted with
  outlier cells marked. Version spread, stale pins, and missing policy bounds
  (audit notes §2) fall out of the row-match rule.
- Series not yet publishing a manifest render as **"awaiting migration"**.

**Rendering: a static HTML page with client-side JS** that fetches
`data/index.json` plus each series' live `qe.status.json` (GitHub Pages serves
`Access-Control-Allow-Origin: *`). The dashboard is therefore current the
moment any series publishes — no rebuild trigger, no build pipeline, no
ingestion. A series whose manifest fetch fails renders an explicit error
state (upgrading to "last seen <date>" once the history collector lands).

## Repo layout

```
data/
  index.json                  # registry: series → site URL, manifest URL, migrated flag;
                              # dashboard settings (e.g. stale_after_days: 90)
schema/
  manifest.schema.json        # JSON Schema, versioned via schema_version
.github/workflows/
  pages.yml                   # deploy site/ + data/ to Pages (only when they change)
site/
  index.html                  # the dashboard (static, client-side rendering)
  dashboard.js / dashboard.css
```

(`data/history/` and its collector are a future bolt-on — see *Future
improvements*.)

There is no `data/series/*.json` "latest" store: **the lecture sites
themselves are the canonical latest**. Consumers read
`https://<series-site>/qe.status.json` directly — a plain HTTPS GET,
discovered via `data/index.json`.

## Monitored data

Everything the manifest carries, by source. Package rows always carry both
the **specifier** (declared) and the **resolved** version (effective, from
the built environment) — declared-vs-effective drift is read off one record.

### a) Environment — `environment.yml` + built env (`conda list` / `pip list`)

| Field | Manifest path | Notes |
|---|---|---|
| python version | `environment.python` | resolved at build |
| anaconda metapackage pin | `environment.anaconda_pin` | `anaconda=YYYY.MM` — the student-environment contract |
| jupyter-book | `environment.jupyter_book` | specifier + resolved; `<2.0` policy bound checked from specifier |
| quantecon-book-theme | `environment.theme` | specifier + resolved; `<0.22` policy bound |
| sphinx extensions | `environment.sphinx_extensions` | standard set: `sphinx-tojupyter`, `sphinxext-rediraffe`, `sphinx-exercise`, `sphinx-proof`, `sphinxcontrib-youtube`, `sphinx-togglebutton`, `sphinx-reredirects` |

### b) Build configuration — `_config.yml` (jb<2) / `myst.yml` (jb≥2)

The producer normalizes both formats into the same manifest fields and
records **which file it read** — that field doubles as the jb1→jb2 migration
signal on the dashboard.

| Field | Manifest path | Notes |
|---|---|---|
| config format | `config.source_file` | `_config.yml` or `myst.yml` |
| execute mode | `config.execute_mode` | cache / auto / force / off |
| execution timeout | `config.execute_timeout` | seconds |
| `only_build_toc_files` | `config.only_build_toc_files` | jb<2 concept; `null` under jb≥2 |
| launch buttons | `config.launch_buttons` | colab, etc. |
| intersphinx targets | `config.intersphinx` | |

### c) Lecture execution statistics — the execution cache (the data behind `status.html`)

**Decided:** read the execution cache directly — the `.jupyter_cache` SQLite
database under jb<2 (myst-nb), mystmd's execute cache under jb≥2 — an
engine-specific branch in the collector, parallel to the
`_config.yml`/`myst.yml` config branch. Parsing the rendered `status.html`
stays available as a fallback only if a cache format proves awkward.

| Field | Manifest path | Notes |
|---|---|---|
| documents / succeeded / failed | `execution.documents` etc. | failed documents listed by name when > 0 |
| total run time | `execution.total_run_time_seconds` | |
| slowest document | `execution.slowest` | name + run time |
| status page link | `execution.status_page` | dashboard links through for per-lecture detail |

### d) Repo settings & hygiene — `.github/dependabot.yml`, workflow presence + Actions API

| Field | Manifest path | Notes |
|---|---|---|
| dependabot | `hygiene.dependabot` | presence + ecosystems (`github-actions`, `conda`) + `jupyter-book>=2.0` ignore; the most predictive health field (audit notes §3) |
| linkcheck | `hygiene.linkcheck` | **last run conclusion + timestamp**, not file presence (audit notes §4) |

### e) Publish workflow & CI pins — `.github/workflows/*.yml`

| Field | Manifest path | Notes |
|---|---|---|
| runner | `build.runner` | raw `runs-on` string + parsed provider (github-hosted / RunsOn / self-hosted), EC2 instance family, custom machine image, GPU flag — e.g. lecture-jax's `family=g4dn.2xlarge/image=quantecon_ubuntu2404` |
| container vs conda | `build.container` | image + tag + digest, or `null` for inline conda |
| environment source | `environment.source` | `environment.yml` / `container` / `machine-image` — tracks the planned transition to regularly updated images |
| setup-miniconda pin | `build.setup_miniconda` | v3/v4 both live in the org today |
| cache strategy | `build.cache.strategy` | |
| QuantEcon/actions pins | `pins.quantecon_actions` | consumer pin vs latest release = producer/consumer lag |
| internal action pins | `pins.internal_actions` | e.g. `action-style-guide` |
| third-party action pins | `pins.third_party_actions` | high-churn set: checkout, upload/download-artifact, codecov, netlify |

### f) Publish run metadata — Actions context at build time

| Field | Manifest path | Notes |
|---|---|---|
| publish timestamp | `collected_at` | drives the "X days old" badge |
| workflow run id / url / file | `workflow_run` | hook for the future publish-failure monitoring bot |
| build duration | `build.duration_seconds` | |
| cache hit | `build.cache.hit` | |

### g) Series identity & translations — derived from each repo's own checkout

| Field | Manifest path | Notes |
|---|---|---|
| repo | `repo` | source repository |
| language | `language` | `target-language` from `.translate/config.yml` where present; `en` otherwise |
| translation of | `translation_of` | parent series, derived from the repo-name suffix (`.translate/config.yml` does not name the source repo); `null` for parents |
| translations | `translations` | parents: language list derived from their `sync-translations-<lang>.yml` workflows; `[]` otherwise |
| translation tooling | `pins.internal_actions` | recorded as `action-translation`: from the sync-workflow pin on parents, from `.translate` `tool-version` on translation repos |

Everything stays **self-described from the repo's own checkout** — the
parent's sync workflows *are* its declaration that translations exist, and
the `.translate` folder marks a repo as a translation; no cross-repo config
is hand-maintained. (Upgrade path: if `action-translation` ever adds a
`source-repo:` key to `.translate/config.yml`, `translation_of` switches
from name convention to data. A formal versioned schema for that file isn't
warranted — it has one writer and `tool-version` already identifies the
format.) The dashboard can render translations as sub-columns of
their parent, and cross-check that the two directions agree (parent claims
`zh-cn` ⇄ a `.zh-cn` repo reports the matching `translation_of`).

## Who builds the manifest: an action, not a MyST extension

`qe.status.json` is assembled by a **composite action in `QuantEcon/actions`**
(a post-build, pre-deploy step in `publish-gh-pages`), not by a Sphinx/MyST
extension running inside the build. Three reasons:

1. **Half the data is invisible from inside the build.** Runner/instance
   details, action pins, build duration, cache hit/miss, `dependabot.yml` —
   workflow-level facts (categories d–f) that no build extension can know.
   An extension would still need an action to finish the manifest: two
   producers and merge semantics, which we've already rejected.
2. **It survives jb1→jb2.** Sphinx extensions don't run under mystmd (jb≥2),
   so an extension-based producer would stop reporting at exactly the
   migration the dashboard tracks. The action reads checked-out files and
   built output, so it is build-tool-agnostic — `_config.yml` vs `myst.yml`
   is just a parsing branch.
3. **Zero per-repo footprint.** No package to add to every series'
   `environment.yml` + `_config.yml`; the collector ships, versions, and
   rolls out with the publish flow itself — consistent with
   `QuantEcon/actions` as the unification point.

Escape hatch: if extracting execution statistics from `.jupyter_cache` /
`status.html` ever proves brittle, a minimal in-build hook may emit an
intermediate exec-stats JSON into `_build/html/` for the action to fold in —
but the action always owns assembling and writing `qe.status.json`.

## Manifest schema (sketch — finalized with actions#30 in Phase 1)

Served as **`qe.status.json`** at each site root — flat and dot-namespaced,
following the mystmd convention for machine-readable endpoints
(`myst.xref.json`, `myst.search.json`). The `qe.` prefix keeps it clear of
anything a build tool will ever emit; future endpoints (e.g. for custom
books) become `qe.*.json` siblings; and because consumers discover URLs via
the registry, the manifest can still be broken apart — or moved under a
folder — later without breaking anyone. Conventions follow
`action-activity-report`'s `report-data.json` (integer `schema_version`,
snake_case, ISO 8601). Fields incorporate audit notes §1:

```jsonc
{
  "schema_version": 1,
  "repo": "lecture-python-intro",
  "language": "en",                   // .translate/config.yml target-language, else "en"
  "translation_of": null,             // parent series (from repo-name suffix); null for parents
  "translations": ["zh-cn", "fa"],    // parents: derived from sync-translations-<lang>.yml
  "collected_at": "2026-06-11T02:00:00Z",
  "workflow_run": { "id": 123, "url": "...", "workflow_file": "publish.yml" },
  "build": {
    "runner": {
      // raw runs-on string preserved verbatim; the rest parsed from it, e.g.
      // "runs-on=<run_id>/family=g4dn.2xlarge/image=quantecon_ubuntu2404/disk=large"
      "runs_on": "ubuntu-latest",
      "provider": "github-hosted",      // github-hosted | runson | self-hosted
      "instance_family": null,          // EC2 family for RunsOn, e.g. "g4dn.2xlarge"
      "machine_image": null,            // custom AMI, e.g. "quantecon_ubuntu2404"
      "gpu": false
    },
    "container": null,                  // or { image, tag, digest }
    "setup_miniconda": "v4",            // conda-incubator/setup-miniconda pin, if used
    "cache": { "strategy": "...", "hit": true },
    "duration_seconds": 1234
  },
  "environment": {
    // where the build environment comes from — environment.yml today; a
    // regularly updated container/AMI is the planned future (see Future
    // improvements). Transitioning is a data change, not a schema break.
    "source": "environment.yml",        // environment.yml | container | machine-image
    "python": "3.13.2",
    "anaconda_pin": "anaconda=2025.06",
    // specifier (declared) + resolved (effective) per key package:
    "jupyter_book": { "specifier": ">=1.0,<2.0", "resolved": "1.0.4" },
    "theme":        { "name": "quantecon-book-theme",
                      "specifier": "<0.22", "resolved": "0.21.0" },
    "sphinx_extensions": { "sphinx_exercise": { "specifier": null, "resolved": "1.0.1" }, ... }
  },
  "config": {
    "source_file": "_config.yml",       // or "myst.yml" under jb>=2 — the migration signal
    "execute_mode": "cache",
    "execute_timeout": 600,
    "only_build_toc_files": true,
    "launch_buttons": ["colab"],
    "intersphinx": ["..."]
  },
  // lecture-level execution statistics — aggregate of the execution cache
  // (jb<2: .jupyter_cache; jb>=2: mystmd execute cache — see Monitored data c):
  "execution": {
    "status_page": "https://python.quantecon.org/status.html",
    "documents": 110,
    "succeeded": 110,
    "failed": 0,
    "failed_documents": [],             // names, when failed > 0
    "total_run_time_seconds": 3456,
    "slowest": { "document": "amss2", "run_time_seconds": 312 }
  },
  "pins": {
    "quantecon_actions": "v0.6.0",      // QuantEcon/actions/* refs in workflows
    "internal_actions": { "action-style-guide": "v0.5", "action-translation": "0.15.0" },
    "third_party_actions": { "actions/checkout": "v6", "codecov/codecov-action": "v7" }
  },
  "hygiene": {
    "dependabot": { "present": true, "ecosystems": ["github-actions", "conda"],
                    "jb2_ignore": true },   // most predictive health field (notes §3)
    "linkcheck": { "present": true, "last_conclusion": "success",
                   "last_run_at": "2026-06-10T03:00:00Z" }  // run state, not file presence (notes §4)
  }
}
```

All of `pins` and `hygiene` come from the producer reading its own checkout
and the Actions API during the publish run — no second collection path.

## Edge cases & mitigations

Failure modes of the pull model, considered up front:

1. **Failed builds are invisible to the manifest.** A failed publish never
   deploys, so `qe.status.json` silently stays at the last *successful* build.
   **Accepted:** publish failures are observed at the point of merge, and the
   dashboard renders the last publish date + age ("X days old") prominently,
   with `collected_at` older than `stale_after_days` showing amber — a
   dashboard parameter in `data/index.json`, **default 90 days**, overridable
   per series later if publish cadences diverge. Active run-status monitoring
   is deferred — see *Future improvements*.
2. **Manifest vanishes on rollback/manual publish.** Each deploy replaces the
   whole site; a publish from a non-migrated workflow (or a manual gh-pages
   push) deletes `qe.status.json`. Mitigation: the registry carries a
   **`migrated` flag**, so a missing manifest on a migrated series renders as
   **"manifest missing"** — a visible regression — rather than silently
   reverting to "awaiting migration". (The history collector later upgrades
   this to "manifest missing — last seen <date>".)
3. **CDN/browser caching.** GitHub Pages sits behind a CDN (~10 min TTL) and
   browsers cache JSON. The dashboard fetches with a cache-busting query
   param; worst case the table is minutes stale, which is acceptable.
4. **CORS depends on staying on GitHub Pages.** Pages serves
   `Access-Control-Allow-Origin: *`; if a series ever moves to another host,
   its manifest must keep permissive CORS or that series renders as a fetch
   error. All production lecture sites are on Pages today; the registry
   records the host so this is auditable.
5. **Mixed schema versions across series.** Series migrate at different
   times, so the dashboard will routinely see different `schema_version`s
   simultaneously. The renderer must tolerate older versions (render known
   fields, mark unknown) rather than assuming uniformity.
6. **Site URL mapping.** Custom domains and subpath sites (translations)
   mean manifest URLs can't be derived from repo names — the registry stores
   the explicit manifest URL per series, and it's the single place to update
   if a domain changes.
7. **Information exposure.** Every schema field is already public (public
   repo files, public Actions logs/runs, the published `status.html`) — the
   manifest only adds permanence and aggregation. The risk is implementation,
   not schema: the producer must build the manifest from an **explicit
   allowlist of fields** and never serialize context wholesale (no
   `os.environ` dumps, no `toJSON(github)`, no raw runner/API responses) —
   that's how IPs, AWS account identifiers, or tokens would end up in a file
   published to a CDN. Enforce in the Phase 2 review and producer tests.

## Tracked series

Source lecture repos only — `.notebooks` output mirrors excluded. Initial
registry: `lecture-python-intro`, `lecture-python-programming` (+ `.fa`,
`.zh-cn`), `lecture-python.myst` (+ `.zh-cn`), `lecture-python-advanced.myst`,
`lecture-jax`, `lecture-dp`, `lecture-datascience.myst`, `lecture-julia.myst`,
`lecture-intro.zh-cn`, `lecture-stats`, `lecture-wasm`. The registry lives in
`data/index.json` (series → site/manifest URL + `migrated` flag); series not
yet publishing a manifest render as "awaiting migration".

## Consumers

- `QuantEcon.manual` — features table + per-series env block
  ([QuantEcon.manual#94](https://github.com/QuantEcon/QuantEcon.manual/issues/94)):
  reads each series' live `qe.status.json` (current state) and `data/index.json`
  (registry).
- `action-activity-report` — week-over-week env diffs
  ([action-activity-report#25](https://github.com/QuantEcon/action-activity-report/issues/25)):
  **activates once the history collector lands** (see *Future improvements*);
  it will read `data/history/<repo>.ndjson` raw URLs and difference the
  records bracketing the report window.

## Development workspace

This repo is the **primary working repo** for the initiative. The sibling
repos are cloned into a gitignored `repos/` folder so the schema contract and
producer/consumer code can be developed side-by-side:

```
repos/
  actions/                  # producer (QuantEcon/actions#30)
  QuantEcon.manual/         # manual views (#94) + naming convention (#95)
  action-activity-report/   # weekly diffs (#25)
  meta/                     # umbrella issue lives here
```

Each clone keeps its own git remote; changes to those repos are committed and
PR'd from within `repos/<name>/` as usual.

## Phases

*Each phase: agree implementation approach in review first, then build.*

### Phase 1 — Manifest schema + registry
- [ ] Review: field-by-field walkthrough of the schema sketch against actions#30 and the audit-notes field list; settle `schema_version` semantics. (Filename decided: `qe.status.json` at the site root.)
- [ ] Write `schema/manifest.schema.json` (v1) + `docs/manifest-schema.md`.
- [ ] Create `data/index.json` with the initial registry (explicit manifest URL per series, `migrated` flags all false, dashboard settings incl. `stale_after_days: 90`).

### Phase 2 — Producer (in `repos/actions/`, lands via actions#30)
- [ ] Review: where manifest generation hooks into `publish-gh-pages` (post-build, pre-deploy, into `_build/html/`); failure behavior (manifest step must never fail the publish).
- [ ] Implement collection (config parse, resolved env from the built environment, execution statistics from the execution cache — `.jupyter_cache` SQLite under jb<2, mystmd execute cache under jb≥2 — pins/hygiene from checkout + Actions API) → write `qe.status.json` into the artifact.
- [ ] PR to `QuantEcon/actions`.

### Phase 3 — Dashboard
- [ ] Review: exact row set, match/highlight rules (strict equality vs normalized comparison per field type), and the three non-reporting states (awaiting migration / manifest missing / fetch error).
- [ ] `site/` — static page, client-side fetch of registry + manifests, cross-comparison table with green/divergent rows, manifest-age badges.
- [ ] Test against a hand-placed fixture manifest before the producer ships.
- [ ] `pages.yml` + enable Pages (deploy-from-Actions).

### Phase 4 — First series end-to-end
- [ ] Wire one series (suggest `lecture-python-intro`) through publish → `qe.status.json` → dashboard; set its `migrated` flag in the registry.
- [ ] Hand off URLs to QuantEcon.manual#94; note on action-activity-report#25 that diffs activate with the history collector.
- [ ] Update this repo's README and note the architecture refinements (pull model, no backstop, history as a bolt-on) on meta#321.

## Future improvements

- **Manifest history collector.** A single-purpose scheduled action in this
  repo: fetch every registry manifest (daily), append changed ones to
  `data/history/<repo>.ndjson`, record per-series fetch failures — committing
  with its own `GITHUB_TOKEN`, so still no new credentials. One series'
  fetch failure must not abort the rest, and commits happen only on change
  (no daily noise); growth is negligible (~2–4 KB per changed snapshot).
  This is what unblocks the `action-activity-report` weekly diffs (#25), the
  "manifest missing — last seen <date>" dashboard state, and is where the
  hand-collected 2026-06-11 baseline (audit notes §6) gets seeded as the
  pre-automation reference record.
- **Other manifest consumers — e.g. custom books.** `qe.status.json` +
  `data/index.json` together form a machine-readable, single-source-of-truth
  metadata endpoint for every series. Beyond the dashboard, that enables
  things like a "custom book" assembled from multiple series — fetching each
  source's effective environment and config to validate compatibility (same
  python/theme/extension stack) before composing, or to pin the combined
  build to what actually built its parts. No design commitment here; the
  point is that `schema_version` discipline keeps the manifest safe to build
  such consumers against.
- **Container-based environments.** As `QuantEcon/actions` develops, the org
  may transition from per-repo `environment.yml` files to a regularly updated
  container image (the GPU runners' custom AMI `quantecon_ubuntu2404` is
  already this pattern). The schema is shaped so this is a **data change, not
  a schema break**: `environment.source` flips to `container`/`machine-image`,
  `build.container` (tag + digest) becomes the student-environment contract
  the way `anaconda_pin` is today, and specifiers go `null` while resolved
  versions keep coming from the built env — so cross-series comparison rows
  keep working through the transition, and a mixed fleet (some series on
  containers, some on `environment.yml`) renders honestly. The image's own
  update cadence (tag age) becomes a new monitorable once the transition
  starts.
- **Publish-failure monitoring bot.** The pull model can't see failed builds
  (edge case 1). If merge-time observation ever proves insufficient, add one
  single-purpose bot: a scheduled job querying each series' publish-workflow
  last run conclusion via the Actions API (`GITHUB_TOKEN`, public repos) and
  opening/pinging an issue on failure. Deliberately separate from the
  dashboard — it monitors runs, not configuration. The
  manifest's `workflow_run.workflow_file` field already records which
  workflow to query, so no schema change is needed to add this later.

## Open questions

- Whether translations (`.zh-cn`, `.fa`) publish to subpaths of the parent
  domain or their own — affects only registry URLs.
- **Non-Python series.** The `environment` block is conda/Python-shaped, but
  the registry includes `lecture-julia.myst` (Project.toml / Manifest.toml)
  and `lecture-wasm`. Proposed: keep them in the registry (they render as
  "awaiting migration" until their publish flows migrate anyway), make the
  Python-specific fields nullable, and add ecosystem-specific environment
  blocks in a later schema version when those series actually migrate.
