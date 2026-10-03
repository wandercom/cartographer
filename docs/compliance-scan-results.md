# Compliance Scan Results

Point-in-time results from the first Cartographer compliance-posture run.

Date: 2026-06-01

Status: draft feature validation, not an attestation.

## Cartographer Validation

These checks validate the generic compliance feature in Cartographer.

| Check | Result |
|------|--------|
| `python3 -m pytest -q` | 66 passed |
| `python3 -m pytest tests/compliance -q` | 19 passed |
| `python3 -m pytest tests/smoke -q` | 47 passed |
| `python3 -m compileall -q src/cartographer` | Passed |
| `git diff --check` | Passed |
| `cartographer compliance scan --framework pcidss --format json` | Exited 0 |
| `cartographer compliance gen-tests --out /tmp/cartographer-compliance-tests` | 12 artifacts |

Generated contract fixtures under `tests/src_cartographer_*` are retained on
disk as reference artifacts, but are quarantined from the default pytest gate
until they are regenerated against Cartographer's current public APIs. They
previously blocked collection through duplicate `contract_test.py` module names
and stale private API expectations.

PCI DSS self-scan status:

| Status | Count |
|--------|-------|
| PASS | 0 |
| WARN | 0 |
| FAIL | 0 |
| INFO | 7 |
| Scored controls | 0 |
| Percent | 100.0 (no scored controls) |

PCI DSS controls are informational for Cartographer itself because Cartographer
does not declare the `payment_data` project tag and has no Ledger financial
registry. The same controls run as scored checks in payment-data projects.

## MEA Adoption Scan

Command shape:

```bash
PYTHONPATH=<workspace>/cartographer/src \
  python3 -m cartographer.cli.main compliance scan --format json
```

The MEA scan was run from `<workspace>/MEA` using the uncommitted
Cartographer compliance feature. The current posture baseline was written to
`<workspace>/MEA/.cartographer/compliance/baseline.json`.

Overall score:

| Metric | Value |
|--------|-------|
| Score | 100.0% |
| PASS | 49 |
| WARN | 0 |
| FAIL | 0 |
| Scored controls | 49 |

Framework breakdown:

| Framework | PASS | WARN | FAIL | INFO |
|-----------|------|------|------|------|
| CCPA/CPRA | 6 | 0 | 0 | 0 |
| CJIS | 6 | 0 | 0 | 0 |
| FedRAMP | 6 | 0 | 0 | 0 |
| GDPR | 6 | 0 | 0 | 0 |
| HIPAA | 6 | 0 | 0 | 0 |
| ISO 27001 | 6 | 0 | 0 | 0 |
| PCI DSS | 7 | 0 | 0 | 0 |
| SOC 2 | 6 | 0 | 0 | 0 |

Remediation applied:

- MEA `cartographer.yaml` now wires `compliance.evidence_index` to existing
  `docs/compliance/*` artifacts.
- MEA `cartographer.yaml` declares intentional public/bootstrap route patterns.
- Ledger PII, COMPLIANCE, and FINANCIAL fields now carry `encrypted_at_rest`.
- Fields marked `gdpr_erasable` now carry `erasure_method`.

These results show the scanner is live and that the current repository posture
passes its mechanical checks. They do not certify CJIS, ISO 27001, SOC 2, HIPAA,
CCPA/CPRA, GDPR, FedRAMP, or PCI DSS readiness.
