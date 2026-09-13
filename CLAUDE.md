# Cartographer

Stack adoption and compatibility tool for Constrain/Pact/Ledger/Arbiter/Baton/Sentinel.

## Architecture

Two modes:
- **discover** — scan source code, live backends, and running services; produce draft artifacts for each stack tool
- **check** — evaluate existing projects against stack requirements; produce compatibility reports

Read-only invariant: never writes to backends, never registers artifacts without human approval.

## Implementation

- Python 3.12+, click CLI, pydantic models, pyyaml
- Source scanner is AST-based (Python) with regex fallback (TS/JS/Ruby/Go/Java)
- Plugin architecture: one scanner per language, one drafter per tool, one checker per tool
- Classification patterns from `DEFAULT_PATTERNS` dict in `sensitive_fields.py` (extendable via YAML)
- Live backend introspection: postgres, redis, mongodb, kafka (all read-only)
- Service probing: HTTP-based OpenAPI spec discovery + file-based spec parsing
- HTTP API via Starlette/uvicorn (optional `[api]` dependency)

## Key Files

- `src/cartographer/cli/main.py` — CLI entry point (click), 7 commands + drafts subgroup
- `src/cartographer/discovery/scanner.py` — Source scanner orchestrator
- `src/cartographer/discovery/plugins/` — Language-specific scanners (6 languages)
- `src/cartographer/discovery/live/` — Live backend introspectors (postgres, redis, mongodb, kafka)
- `src/cartographer/discovery/service_prober.py` — OpenAPI service probing
- `src/cartographer/drafters/` — Artifact drafters (constrain, pact, ledger, baton, sentinel)
- `src/cartographer/compatibility/` — Checkers per tool + runner (5 tools, 34 checks)
- `src/cartographer/report/generator.py` — Text and JSON report formatting
- `src/cartographer/api/server.py` — HTTP API (Starlette ASGI)
- `src/cartographer/config/loader.py` — cartographer.yaml parsing
- `src/cartographer/models.py` — Core data models

## Commands

```bash
cartographer init                                          # create cartographer.yaml
cartographer discover [--only <tools>] [--no-live]         # scan and draft
cartographer discover --service http://localhost:8001       # probe a service
cartographer discover --openapi ./specs/api.yaml           # parse OpenAPI spec
cartographer check [--tool <tool>] [--strict] [--format json]
cartographer report                                        # show latest report
cartographer adopt [--confidence <level>] [--dry-run]
cartographer drafts list [--tool <tool>]
cartographer drafts show <tool> <artifact>
cartographer serve [--host 0.0.0.0] [--port 8090]          # HTTP API
```

## Testing

```bash
python3 -m pytest tests/smoke/ -v    # 47 smoke tests
```

## References

- Constrain session: `.constrain/sessions/` (artifacts: prompt.md, constraints.yaml, trust_policy.yaml, component_map.yaml, schema_hints.yaml)
- Baton skeleton: `baton.yaml`
- Brief: `brief.md`
- Landing page: `docs/index.html` (GitHub Pages at wandercom.github.io/cartographer)
