# PostgreSQL skills — Elysium / Opereta selection

Fork of [timescale/pg-aiguide](https://github.com/timescale/pg-aiguide),
used as a **storage of agent skills** (Claude format) for the Elysium Java base
(PostgreSQL 18, Flyway) and for the Opereta application. The upstream MCP server
and its tooling are not used here.

## Branches

| Branch | Content |
|---|---|
| `main` | exact mirror of the upstream `main`; never committed to |
| `elysium-stack` | skills selected and adapted for the Elysium Java base (incl. PostGIS) |
| `opereta-stack` | `elysium-stack` plus what is specific to Opereta (no PostGIS) |

Changes flow one way: upstream → `main` → `elysium-stack` → `opereta-stack` (by merge).
Fixes that are useful for everyone go to a topic branch from `main` and upstream as a pull request.

## Layout

- `.claude/skills/<skill>/` — the skills in use (`SKILL.md`, `references/`).
- `_unused/` — everything else from upstream, moved (not deleted) so that upstream changes keep merging:
  `_unused/skills/` (skills not selected, e.g. TimescaleDB, pgvector), `_unused/root/` (MCP server,
  tooling, original README), `_unused/elysium-only/` (on `opereta-stack` only).

Moves are always separate commits without content changes; content changes follow in their own commits.

## License

Apache License 2.0, see [LICENSE](LICENSE) and [NOTICE](NOTICE). Original work by Timescale, Inc.;
the original README is kept in `_unused/root/README.md`.
