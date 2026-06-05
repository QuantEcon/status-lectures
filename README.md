# status-lectures

Status and configuration dashboard for QuantEcon lecture series.

This repository is the central **data store + dashboard** for environment and
build-configuration reporting across the `lecture-*` series. It is GitHub-native:
the data lives in-repo as JSON and the dashboard is published via GitHub Pages.

## What it tracks

For each lecture series: build runner (container vs conda, container tag/digest,
GPU), Python and key package versions (jupyter-book, theme, sphinx extensions),
execution settings (execute mode, timeout, cache), enabled features (launch
buttons, intersphinx), and build health (cache hit, duration).

## How data arrives

- **Publish-push (primary):** each lecture's publish workflow (via
  `QuantEcon/actions`) emits an effective-environment manifest and sends it here
  via `repository_dispatch` on every publish.
- **Daily backstop:** a scheduled job scrapes `_config.yml` + `environment.yml`
  for not-yet-migrated repos and flags declared-vs-effective drift.

## Consumers

- `QuantEcon.manual` — cross-cutting features table + per-series environment block
- `QuantEcon/action-activity-report` — week-over-week environment *diffs* in the
  weekly report

## Status

Scaffolding in progress. Tracked under the environment & config reporting
initiative in `QuantEcon/meta`.
