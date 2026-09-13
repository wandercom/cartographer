# Cartographer

Stack adoption and compatibility tool for the [Constrain](https://github.com/wandercom/constrain) software engineering stack.

Cartographer answers two questions:

1. **Discovery** -- given an existing codebase, running services, and live data backends, what draft artifacts can be produced for Constrain, Pact, Ledger, Arbiter, Baton, and Sentinel?

2. **Compatibility check** -- given a codebase built without the stack (or with an earlier version), what is missing, what is wrong, and what needs to change?

Cartographer does not make changes. It discovers, drafts, and reports.

## Install

```bash
pip install cartographer
```

Or from source:

```bash
git clone https://github.com/wandercom/cartographer.git
cd cartographer
pip install -e ".[dev]"
```

## Quick Start

```bash
# Initialize configuration
cartographer init

# Discover what exists in your codebase
cartographer discover --no-live

# Check compliance against stack requirements
cartographer check

# Review draft artifacts
cartographer drafts list

# Show a specific draft
cartographer drafts show pact auth_service_draft.yaml
```

## Discovery

Cartographer scans source code using AST-based analysis (Python) and regex-based detection (TypeScript, JavaScript, Ruby, Go, Java) to find:

- **ORM models** -- SQLAlchemy, Django ORM, Prisma, TypeORM, Sequelize, Mongoose, Drizzle, ActiveRecord, Sequel, GORM, sqlx, JPA/Hibernate
- **API routes** -- Flask, FastAPI, Express, Rails, net/http, Gin, Spring
- **PACT keys** -- existing instrumentation markers
- **Backend connection hints** -- environment variables suggesting databases, caches, queues
- **Sensitive fields** -- PII, financial, auth, and compliance field patterns

Every discovered item has a confidence level (HIGH, MEDIUM, LOW) and a note explaining the evidence.

```bash
cartographer discover                    # full discovery
cartographer discover --only ledger      # just ledger drafts
cartographer discover --only pact,baton  # subset
cartographer discover --no-live          # source only, no live probing
```

## Compatibility Check

Evaluates an existing codebase against each stack tool's requirements. Works in CI without any running services.

```bash
cartographer check                       # check everything
cartographer check --tool pact           # check one tool
cartographer check --strict              # exit non-zero on any warning
cartographer check --format json         # machine-readable output
```

## Compliance Posture

`cartographer compliance` verifies compliance posture against declarative
controls for CJIS, ISO 27001, SOC 2, HIPAA, CCPA/CPRA, GDPR, FedRAMP, and PCI
DSS. It is
project-agnostic: project-specific facts come from `cartographer.yaml`, source
scanning, evidence documents, and the on-disk Ledger registry. Cartographer does
not reclassify data; Ledger remains the data-classification source of truth.

```bash
cartographer compliance scan --format json
cartographer compliance scan --strict
cartographer compliance gen-tests --out tests/compliance
cartographer compliance baseline --update
cartographer compliance verify
cartographer compliance add-risk --framework gdpr --describe "New risk..." --dry-run
```

`scan --strict` and `verify` are fail-closed CI gates. LLM judgment controls are
skipped by default and require `--with-llm` plus `cartographer[llm]`.

Controls with contextual scope skip as informational until the project declares
the relevant `compliance.project_tags`; for example, PCI DSS controls run when a
project declares `payment_data`.

Generated validation tests go to `compliance.tests_dir` in `cartographer.yaml`
(`tests/compliance` by default), or to `--out` for one-off runs.

See [Compliance Scan Results](docs/compliance-scan-results.md) for the current
Cartographer validation run and first read-only MEA posture snapshot.

### Checks by Tool

| Tool | Checks | What it validates |
|------|--------|-------------------|
| Pact | 11 | Contracts have data_access/authority, source has PACT keys and handlers |
| Baton | 7 | Schema version current, nodes have data_access/authority, OpenAPI specs |
| Ledger | 8 | Fields classified, no annotation conflicts, GDPR erasure methods |
| Arbiter | 2 | Access graph and trust ledger present and readable |
| Sentinel | 6 | Manifest entries have paths, source keys match manifest |

### Example Report

```
CARTOGRAPHER COMPATIBILITY REPORT
generated: 2026-03-15T14:32:00Z

PACT COMPATIBILITY
  contracts/auth_module.yaml      PASS   data_access present, authority present
  contracts/payment.yaml          FAIL   missing authority field
  src/payment.py                  FAIL   no event_handler, no log_handler

BATON COMPATIBILITY
  baton.yaml schema version       FAIL   version 1.0, requires 2.0
  node: auth_service              PASS

OVERALL SCORE: 61% compliant (18/29 checks passing)
  FAIL: 6   WARN: 5   PASS: 18

RECOMMENDED NEXT STEPS (in order):
  1. Update baton.yaml to schema version 2.0
  2. Add PACT keys to: src/payment.py
  3. Add event_handler and log_handler to payment.py
```

## Adoption Workflow

```bash
# Step 1: Discover
cartographer discover --no-live

# Step 2: Check current compliance
cartographer check

# Step 3: Review and register drafts (per tool)
cartographer adopt --confidence high --dry-run
cartographer adopt --confidence high

# Step 4: Check again -- score should improve
cartographer check
```

## Configuration

Create `cartographer.yaml` in your project root:

```yaml
version: "1.0"

targets:
  source:
    dirs: ["./src", "./lib", "./app"]
    languages: [python, typescript, javascript, ruby, go, java]
    exclude: ["node_modules", ".venv", "dist", "build"]

  infrastructure:
    backends: []

  services:
    base_urls: []
    openapi_paths: []

stack:
  pact_project_dir: null
  baton_config: null
  ledger_registry: null
  arbiter_registry: null
  sentinel_manifest: null

output_dir: ".cartographer/drafts/"

compatibility:
  pact_key_format: "PACT:[a-zA-Z0-9_]+:[a-zA-Z0-9_]+"  # See https://github.com/wandercom/pact/blob/main/PACT_KEY_STANDARD.md
  min_baton_schema_version: "2.0"
```

## Draft Artifacts

All drafts include `_draft: true` and `_generated_by: cartographer` markers. Tools strip these before registration.

| Tool | Draft output |
|------|-------------|
| Constrain | prompt, constraints, component_map, trust_policy, schema_hints |
| Pact | Per-component contracts + task description |
| Ledger | Per-backend + per-model schemas, connection hints |
| Baton | Complete topology (nodes + edges) |
| Sentinel | PACT key manifest |

## The Stack

Cartographer is part of a contract-first software engineering stack:

- **[Constrain](https://github.com/wandercom/constrain)** -- constraint discovery through guided interview
- **[Pact](https://github.com/wandercom/pact)** -- contract-first multi-agent development
- **[Ledger](https://github.com/wandercom/ledger)** -- schema registry and data obligation manager
- **Arbiter** -- trust and authority management
- **[Baton](https://github.com/wandercom/baton)** -- cloud-agnostic circuit orchestration
- **[Sentinel](https://github.com/wandercom/sentinel)** -- production attribution and contract tightening
- **Cartographer** -- adoption and compatibility (this tool)

## License

MIT
