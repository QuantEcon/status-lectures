# status-lectures

Central dashboard for environment/config reporting across the QuantEcon
`lecture-*` series (umbrella: QuantEcon/meta#321).

- **[PLAN.md](PLAN.md) is the authoritative design.** Read it before
  proposing architecture. Key decisions: single pull-based producer (a
  composite action in `QuantEcon/actions` writes `qe.status.json` into each
  published site — no dispatch, no scrape/backstop, no credentials); static
  client-side dashboard; history/monitoring are future bolt-ons.
- [docs/tooling-audit-notes-2026-06.md](docs/tooling-audit-notes-2026-06.md)
  is **historical design input** — evidence, not current design. Where it
  conflicts with PLAN.md, PLAN.md wins (see its status banner).
- Each PLAN phase is reviewed with @mmcky before implementation — don't
  build ahead of the agreed phase.
- Sibling repos for this initiative are cloned under the gitignored
  `repos/` folder (see PLAN.md "Development workspace").
