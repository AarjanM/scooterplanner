# ScooterPlanner

Free, open-source (AGPL-3.0) route planner and turn-by-turn navigation for Dutch
mopeds. Status: planning. Read `GLOSSARY.md` before anything else and use its
terms (Bromfiets, Snorfiets, Vehicle class, Bromfietspad, Legal route, ...).

## Agent skills

### Issue tracker

GitHub Issues on `AarjanM/scooterplanner`. The wayfinder map is issue #1
(label `wayfinder:map`); tickets are its sub-issues with native blocking.
See `docs/agents/issue-tracker.md`.

### Domain docs

Single-context: `GLOSSARY.md` at the root, ADRs under `docs/adr/`,
research notes under `docs/research/`. See `docs/agents/domain.md`.

## Working on this repo

- Planning happens through `/wayfinder` against the map; one ticket per session.
- Research findings land on `research/<name>` branches, never directly on `main`.
- Implementation has not started; do not scaffold application code until the
  v1 spec (`docs/spec.md`) exists.
