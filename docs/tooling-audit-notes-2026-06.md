# Design notes from the May–June 2026 tooling audits

**Date:** 2026-06-11
**Context:** Two manual tooling audits of the QuantEcon org were run recently — a
baseline on 2026-05-27 (34 tier-1 repos) and a delta refresh on 2026-06-11.
Together they are, in effect, two hand-run iterations of exactly what
`status-lectures` is being built to automate. These notes transfer what the
manual process needed, found, and got wrong into design input for the manifest
schema, the drift model, and the daily backstop.

---

## 1. Fields the manual audit actually needed

The README's tracked-field list (runner/container, versions, execution settings,
features, build health) matches what the audit used. Field-level detail worth
capturing explicitly:

### Versions (per lecture series)

- `python` and the **`anaconda=YYYY.MM` metapackage pin** — the metapackage pin
  is the student-environment contract; surfacing it per series shows immediately
  when one series lags an Anaconda release.
- `jupyter-book` — **including the specifier string, not just the resolved
  version**. The org's convention is a defensive `<2.0` upper bound (jb 2.x is
  the MyST-MD rewrite). A repo whose specifier is missing the bound is a policy
  drift even when the resolved version is identical. Same logic applies to the
  theme bound (`<0.22` currently).
- `quantecon-book-theme` — resolved version *and* specifier.
- The standard sphinx-extension set: `sphinx-tojupyter`, `sphinxext-rediraffe`,
  `sphinx-exercise`, `sphinx-proof`, `sphinxcontrib-youtube`,
  `sphinx-togglebutton`, `sphinx-reredirects`.
- **Declared vs effective:** `environment.yml` says X; `conda list` / `pip list`
  at build time says Y. The publish-push manifest is the only reliable source of
  "effective" — the daily backstop can only see "declared".

### Runner / container

- Container image **and tag/digest** (`quantecon` vs `quantecon-build` vs
  no-container/inline-conda) — the org currently has all three patterns live.
- GPU runner spec (the `runs-on=...` string) for the JAX/python.myst-style repos.
- `conda-incubator/setup-miniconda` version for non-container repos (v3/v4 are
  both live today).

### Workflow & action pins (suggested addition)

The audits spent significant time on fields not currently in the README list:

- **Pinned versions of `QuantEcon/actions/*` composite actions** consumed by
  the repo (today: `@v0.6.0` where used at all).
- **Pinned versions of internal QuantEcon actions** (e.g. `action-style-guide`)
  vs the action's latest release. Real example: the only production consumer of
  `action-style-guide` pins `@v0.5` while the action is at `v0.7.2`.
- A small set of high-churn third-party actions (`checkout`,
  `upload-artifact` / `download-artifact`, `codecov-action`,
  `actions-netlify`) — org spread in June 2026 was checkout v4–v6,
  codecov v5–v7, netlify v3/v3.0/v4.0.

### Repo-hygiene presence flags (suggested addition)

Boolean (or version) fields that the audit had to check by hand per repo:

- `dependabot.yml` present + which ecosystems (`github-actions`, `conda`) + the
  `jupyter-book >=2.0` ignore present. See §3 for why this is the single most
  predictive health field.
- `linkcheck.yml` present **and parseable and recently green** (see §4).
- style-guide workflow present + which action version it pins.

---

## 2. Drift classes the dashboard should surface

Concrete examples from the June delta, each a class of drift the dashboard
should make visible without manual digging:

1. **Per-package version spread across the family.** In June, the lecture
   series simultaneously ran theme `0.15.1` / `0.20.2` / `0.20.3` / `0.21.0` —
   and 0.21.0 ships an intentional layout change, so the `*.quantecon.org`
   sites visibly diverge for readers. A "max spread per package" view (or just
   sorting the family table by version) makes this a one-glance finding.
2. **Producer/consumer lag for internal actions.** `action-style-guide` v0.7.2
   released; consumer pinned `@v0.5`. Needs both sides: the action's latest
   release (cheap API call) and the consumer's pin (from the manifest).
3. **Declared-pin policy drift.** A repo missing the `jupyter-book<2.0` or
   theme upper bound — detectable from the specifier string alone.
4. **Stale-pin outliers.** `lecture-dp` pinned theme `==0.15.1` (3 years old)
   because its `environment.yml` was copied from an old repo. Flag any package
   where a series is >1 minor behind the family median.
5. **Uncoordinated runtime waves.** Node 20 vs 24 chosen independently by three
   repos in the same fortnight. Low priority for lecture repos specifically,
   but the same mechanism as (1).

---

## 3. Dependabot presence is the most predictive health field

The clearest single finding from the delta audit: **nearly every tooling bump
in the two-week window landed via Dependabot, not humans.** The four Python
lecture repos with `dependabot.yml` (github-actions + conda) received theme,
conda-action, and sphinx-extension bumps automatically; `lecture-dp` — the one
series without it — received none and silently fell further behind. Dependabot
even delivered to one repo the exact bumps another repo had open as manual
migration tasks.

Implication for the dashboard: don't just track versions — track **whether the
update engine is connected**. A series without Dependabot will drift
monotonically; that's a "warning" health state regardless of how current its
versions are today.

---

## 4. "Workflow exists" ≠ "workflow runs"

Two traps the manual audit fell into that the dashboard design should avoid:

1. **File presence is not health.** One lecture repo's `linkcheck.yml` had been
   *syntactically broken* (unquoted glob) for months — present in the tree,
   never parsed by Actions. If the daily backstop scrapes for file existence,
   it would have reported green. Recommendation: where a workflow matters,
   record its **last run conclusion + timestamp** (one `gh api
   .../actions/workflows/<file>/runs?per_page=1` call), not its presence.
2. **`pushed_at` is misleading for staleness.** One Julia repo's `pushed_at`
   moved because of *unmerged* Dependabot branches; default-branch HEAD was
   months old. The daily backstop should read default-branch HEAD (or the
   workflow-run history), never repo-level `pushed_at`.

---

## 5. Notes on the data-arrival design

- The **publish-push manifest via `QuantEcon/actions`** is the right primary
  mechanism, with one scheduling caveat: `QuantEcon/actions` has had no release
  since v0.6.0 (2026-02-09) and only one lecture repo consumes any of it in
  production today. Until the rollout resumes, the **daily backstop is
  effectively the primary source** for every series — worth designing it to
  carry the full field set (minus effective-versions, which only the
  publish-push path can know), not as a thin fallback.
- The backstop's scrape targets (`_config.yml` + `environment.yml`) cover
  versions and execution settings but not the workflow/pin fields in §1 — those
  come from `.github/workflows/*.yml` and `dependabot.yml`, which the backstop
  can fetch in the same pass (2–3 extra raw-content calls per repo).
- Worth keeping each manifest **append-only with a timestamp** rather than
  latest-only: the weekly diff consumer (`action-activity-report`) needs
  week-over-week pairs, and the two manual audits showed point-in-time
  snapshots become valuable precisely when you can difference them.

---

## 6. Suggested baseline seed

The two manual audits produced a hand-collected version table for the five main
Python lecture series (theme, jupyter-book, python, conda-action, runner — as
of 2026-06-11). Seeding the data store with that snapshot would give the
dashboard a day-one "before automation" reference point and an immediate
demonstration diff when the first real manifests arrive. Happy to contribute it
as JSON once the manifest schema is settled.
