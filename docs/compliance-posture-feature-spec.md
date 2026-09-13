I now have full, verified knowledge of the real APIs. I have everything needed to write a concrete, buildable spec. Here it is.

---

# IMPLEMENTATION-READY SPEC: `cartographer compliance` — Generic Compliance-Posture Verification

Status: ready for Codex. Target package: **Cartographer** (`~/Code/cartographer`, v0.1.2, MIT, Python 3.12+, Click + Pydantic v2 + PyYAML). Project-agnostic; MEA is the first consumer. All APIs cited below are verified against current source.

---

## 1. CHOSEN PACKAGE + RATIONALE

**Package: Cartographer.** The feature lands as a new sibling subpackage `src/cartographer/compliance/` that reuses four existing Cartographer subsystems verbatim. It does **not** live in Ledger.

Why Cartographer and not the others (verified):
- Cartographer's entire charter is *static, read-only, CI-runnable, extensible PASS/WARN/FAIL scanning with scoring* — exactly the shape of compliance-posture verification. It is read-only by invariant ("never writes to backends, never registers artifacts without human approval").
- The four reused subsystems already exist and are load-bearing:
  1. **Checker ABC + registry** — `compatibility/base.py:12` `CompatibilityChecker` (abstract `tool_name` property + `check(config, base_dir) -> list[CheckResult]`); registration is a one-line append to `ALL_CHECKERS` in `compatibility/runner.py:17`.
  2. **Result + scored report** — `models.py:123` `CheckResult{check_id, target, severity, status, message, tool}`; `models.py:132` `CompatibilityReport` with `score_pct` and `has_failures`; `report/generator.py:12` `build_report`, `format_json`, `save_report` (timestamped + `report_latest.{json,txt}` baseline).
  3. **Fail-closed CI gate** — `cli/main.py:159-162`: `if report.has_failures: sys.exit(1)` and `if strict and report.score_warn > 0: sys.exit(1)`.
  4. **Draft → review → adopt loop** — `drafters/base.py` `Drafter` ABC + `DraftArtifact` (`models.py:163`) + `adopt`/`adopt --dry-run` (`cli/main.py:189`).
- `LedgerChecker` (`compatibility/ledger_checker.py`) is a worked control template: it walks a Ledger registry and emits PASS/WARN/FAIL per data-protection rule (`all_fields_have_classification`, `no_annotation_conflicts`, `gdpr_erasable_has_erasure_method`, `audit_fields_have_retention_policy`). The compliance feature generalizes that exact pattern under a framework label.
- The classification-pattern extension model is the template for the control registry: `discovery/sensitive_fields.py:13` `DEFAULT_PATTERNS` (ships a `COMPLIANCE` tier) + `load_patterns(custom_path)` (`:95`) which **merges** a team's YAML over defaults. The compliance control registry uses the identical "ship defaults, merge user YAML" mechanism.

Rejected homes (verified): **Ledger** has zero control/risk/evidence primitives and answers "what is the data and its rules," not "is control X satisfied" — it is the *evidence source*, not the home. **Arbiter** is a runtime evidence loop (needs live OTLP spans). **Sentinel** is post-deploy attribution, status "Not started." **Covenant** validates one payload shape, too narrow.

### How it composes with Ledger's PII classification (reuse, don't reinvent)

The compliance feature **consumes** Ledger as the authoritative data-classification source; it never reimplements classification. Two integration surfaces, both already present:

1. **Ledger registry on disk** — `compliance/` reuses `ledger_checker._find_registry()` (`ledger_checker.py:127`, resolves `config.stack.ledger_registry` → `.ledger/registry` → `.ledger`) to read field tiers (`PUBLIC/PII/FINANCIAL/AUTH/COMPLIANCE`) and obligation annotations (`gdpr_erasable`, `encrypted_at_rest`, `audit_field`, `soft_delete_marker`, `immutable`, `pii_field`). Data-protection controls are expressed as assertions over these fields, e.g. "GDPR Art.17 right-to-erasure → every `gdpr_erasable` field carries an `erasure_method`" (Cartographer already emits this exact check). This is the privacy half of the registry, made generic by Ledger rather than hardcoded.
2. **Ledger export adapters** — Ledger's `src/export/export.py` emits derived obligations (masking, severity, retention hints) to Baton/Sentinel/Arbiter. A control may assert those are wired into the consuming configs; Cartographer's existing `BatonChecker`/`SentinelChecker` already verify those config files exist, so framework controls reuse them.

When a project needs a new data obligation to drive a control (e.g. `hipaa_phi`, `pci_scope`), it is added in Ledger via `ledger.yaml` `custom_annotations:` (config-only, no Ledger code change) — and the compliance control simply references the annotation `name`. Clean separation: **Ledger = field-level data obligations (source of truth); Cartographer = control-posture verification across frameworks (this feature).**

---

## 2. ARCHITECTURE: Extensible Risk/Control Registry

### 2.1 Declarative registry — "ship defaults, merge user YAML" (mirrors `load_patterns`)

The registry is a set of YAML files, one per framework, under a package-default directory plus an optional project override directory. Loading **merges** project YAML over packaged defaults (the exact mechanism of `sensitive_fields.load_patterns`, `:95-107`):

- Packaged defaults: `src/cartographer/compliance/frameworks/{cjis,iso27001,soc2,hipaa,ccpa,gdpr,fedramp}.yaml`
- Project overrides / additions: `compliance/controls/*.yaml` (path set via `cartographer.yaml` → `compliance.controls_dir`, default `./compliance/controls`)

A framework is a closed `framework` slug; controls are open and user-extensible by dropping a YAML control into the controls dir. No code change is required to add a control or a project's framework profile.

### 2.2 Control record schema (the actual shape)

Each control is one document in a framework YAML. Pydantic model `ControlDef` (new, in `compliance/registry.py`). `detection` is a tagged union keyed by `method`. The schema is a superset of MEA's `constraints.yaml` (`id/condition/violation/severity/classification_tier/affected_components`) plus the missing cross-links the investigation flagged (`frameworks[]`, `evidence_refs[]`).

```yaml
# src/cartographer/compliance/frameworks/gdpr.yaml  (packaged default — excerpt)
framework: gdpr                      # one of: cjis|iso27001|soc2|hipaa|ccpa|gdpr|fedramp|<custom>
framework_version: "2016/679"
controls:
  - id: GDPR-ART17-ERASURE                 # globally unique control id
    family: "Data Subject Rights"          # framework family/theme (Annex A theme, TSC, safeguard, etc.)
    framework_ref: "Art. 17"               # citation within the framework
    title: "Right to erasure is enforceable for all erasable PII"
    description: >
      Every field classified PII and annotated gdpr_erasable in the Ledger
      registry must declare an erasure_method, and an erasure execution path
      must exist in source.
    severity: must                         # must | should  (maps must->FAIL, should->WARN on miss)
    classification_tier: PII               # PUBLIC|PII|FINANCIAL|AUTH|COMPLIANCE|null (Ledger tier this keys off)
    applicability:                         # control applies only if ALL match (else SKIPPED, not FAIL)
      requires_ledger_registry: true       # control needs a Ledger registry present
      data_tiers: [PII]                    # applies when project handles these tiers
      tags: [eu_data_subjects]             # arbitrary project capability tags from cartographer.yaml
    detection:                             # tagged union on `method`
      method: ledger_obligation            # see detection methods table below
      tier: PII
      required_annotation: gdpr_erasable
      required_field_key: erasure_method   # field that must accompany the annotation
    corroborating_detection:               # optional secondary mechanical check (AND-ed)
      - method: source_grep
        any_of: ["execute_erasure", "def .*eras"]   # regex, OR-semantics within any_of
        include_glob: "**/*.py"
    validation_test:                       # template the generator renders into an executable test
      style: static_assertion             # static_assertion | goodhart_json
      assertion: ledger_annotation_has_companion_field
      params: {annotation: gdpr_erasable, companion: erasure_method, tier: PII}
    evidence_owner: null                   # null => code-detectable control (no manual evidence)
    status: auto                           # auto => derived by scan; or pinned: control-ready|audit-ready|attested|authorized|certified
    references: ["classification_registry.yaml", "services/compliance/src/data_rights_service.py"]

  - id: GDPR-ART35-DPIA                     # an ORG (evidence-owner) control
    family: "Accountability"
    framework_ref: "Art. 35"
    title: "DPIA performed and current for high-risk processing"
    description: "A Data Protection Impact Assessment exists, is owned, and is within review cadence."
    severity: must
    classification_tier: null
    applicability: {tags: [high_risk_processing]}
    detection:
      method: evidence_doc                 # ORG control: presence + freshness of an evidence artifact
      evidence_path_key: dpia              # resolved via project evidence index (see 2.4)
      max_age_days: 365
    validation_test: {style: static_assertion, assertion: evidence_present_and_fresh,
                      params: {evidence_path_key: dpia, max_age_days: 365}}
    evidence_owner: "DPO"                   # non-null => human-attested; scan reports PRESENT/STALE/MISSING only
    status: auto
    references: ["docs/compliance/dpia.md"]
```

### 2.3 Detection methods (tagged union on `detection.method`)

Mechanical-first; LLM only for judgment controls. Each maps to a deterministic Python evaluator in `compliance/detectors/`, returning `present | partial | missing | skipped`.

| `method` | Evaluator does | Emits |
|---|---|---|
| `ledger_obligation` | Read Ledger registry via `_find_registry`; for each field of `tier`, assert `required_annotation` present and `required_field_key` accompanies it (generalizes `gdpr_erasable_has_erasure_method`) | present if all, partial if some, missing if none |
| `ledger_tier_encrypted` | Every field of `tier` carries `encrypted_at_rest` annotation (HIPAA §164.312, GDPR Art.32, CJIS 5.10) | present/partial/missing |
| `source_grep` | `_grep`-style scan (regex, glob) — presence (`any_of`) or absence (`none_of`) of a pattern across source | present/missing |
| `route_guard` | Assert every discovered route (reuse `discovery.scanner` `DiscoveredRoute`) has an auth guard pattern (`require_role`/`Depends`) | present/partial |
| `config_present` | A config file exists and contains a required key (TLS min version, HSTS header, CSP without `unsafe-inline`) | present/missing |
| `companion_config` | A Cartographer sibling-tool config exists (reuse `BatonChecker`/`SentinelChecker` presence checks) | present/missing |
| `evidence_doc` | ORG control: resolve `evidence_path_key` via project evidence index; assert file exists and mtime within `max_age_days` | present/stale/missing |
| `llm_judgment` | Mechanical pre-filter narrows candidate files, then an LLM verdict prompt classifies present/partial/missing with rationale + cited lines (only when no deterministic signal exists, e.g. "purpose limitation is documented per processing activity") | present/partial/missing + rationale |

`applicability` is evaluated first; a control that does not apply is reported `SKIPPED` and excluded from scoring (never a false FAIL).

### 2.4 Evidence index (ORG controls)

Mirrors MEA's Product Evidence Index. Project supplies an `evidence_path_key → path` map in `cartographer.yaml` (default-empty); ingested from `docs/compliance/compliance-readiness-matrix.md` when present (parser in `compliance/ingest.py`). `evidence_doc` detection resolves keys through this map, so ORG controls carry an owner + a path + a freshness window and report PRESENT/STALE/MISSING — never a green pass on absent evidence.

### 2.5 `cartographer.yaml` additions

Add one optional block (loader is permissive — `CartographerConfig(**raw)`; add a `ComplianceConfig` model on `CartographerConfig`):

```yaml
compliance:
  frameworks: [gdpr, hipaa, cjis, soc2]      # active target frameworks
  controls_dir: ./compliance/controls         # project overrides/additions (merged over packaged)
  project_tags: [eu_data_subjects, cji_data]  # capability tags driving applicability
  evidence_index:                             # evidence_path_key -> path (ORG controls)
    dpia: docs/compliance/dpia.md
    poam: docs/compliance/poam.md
  baseline: .cartographer/compliance/baseline.json   # regression baseline (see §4)
```

---

## 3. THE SCAN

### 3.1 Runnable command

New command group `cartographer compliance` in `cli/main.py` (Click subgroup, same shape as existing `drafts`). `scan` is the verb; it composes a new `compliance/runner.py` (mirroring `compatibility/runner.py`) into the existing `build_report`/`format_*`/`save_report` pipeline.

```bash
# Mechanical scan against active frameworks; writes a coverage report + baseline-able JSON
cartographer compliance scan --framework gdpr --framework hipaa --format json

# All configured frameworks, human report
cartographer compliance scan

# Fail-closed CI gate (reuses cli/main.py:159-162 exit semantics)
cartographer compliance scan --strict

# Include LLM judgment controls (requires ANTHROPIC_API_KEY; default: mechanical only, llm controls -> SKIPPED+flagged)
cartographer compliance scan --with-llm
```

Mechanical path is fully deterministic and offline (reads source, configs, Ledger registry, evidence files). `--with-llm` enables `llm_judgment` detectors. Output is a per-control coverage report grouped by framework→family, each control PRESENT/PARTIAL/MISSING/SKIPPED, with `score_pct` and recommendations from `report/generator.py`. Reports saved to `.cartographer/compliance/reports/` (timestamped + `report_latest`).

`compliance/runner.py` builds one `FrameworkChecker(CompatibilityChecker)` per active framework. Each `FrameworkChecker` loads its merged control set, runs the detector per control, and emits one `CheckResult` per control with `tool="<framework>"`, `check_id="<control.id>"`, `status` mapped from detector verdict (present→PASS, partial→WARN, missing→FAIL when `severity=must` else WARN, skipped→omitted/INFO). This drops straight into the existing scoring + JSON + exit-code machinery with zero changes to `models.py`, `report/generator.py`, or the exit logic.

### 3.2 AI SCAN PROMPT (verbatim — for `llm_judgment` controls and the optional full-codebase pass)

Use this as the system+user prompt sent per LLM-judgment control (or, in a "full pass" mode, once per framework with the full control list). The mechanical pre-filter supplies `candidate_files` so the model reasons over a bounded context. The model returns strict JSON that maps 1:1 onto `CheckResult`.

```
You are a compliance-posture verification engine. You do NOT write or modify code.
Your only job: decide, for each control given to you, whether it is PRESENT, PARTIAL,
MISSING, or SKIPPED in the codebase, citing concrete evidence. Be conservative: if you
cannot find positive evidence, the control is MISSING. Never infer compliance from
intent, comments, TODOs, or documentation prose alone — require an enforcing mechanism
in code, config, or a present-and-fresh evidence artifact.

INPUTS
- framework: <slug, e.g. gdpr>  framework_version: <version>
- controls: a JSON array; each item is the control record from the registry
  (id, family, framework_ref, title, description, severity, classification_tier,
   applicability, detection, references).
- project_facts: machine-derived facts you must treat as ground truth:
    - data_classification: the Ledger registry tiers + annotations per field
      (this is the AUTHORITATIVE map of what data is sensitive; do not re-guess PII).
    - discovered_routes: list of {path, method, handler, has_auth_guard}.
    - present_configs: list of config files + the security-relevant keys found in each.
    - evidence_index: {evidence_path_key: {path, exists, age_days}}.
    - candidate_files: for each control id, a pre-filtered list of file paths +
      the matching line ranges the mechanical scanner found relevant.
- file_excerpts: the actual text of the candidate_files' relevant line ranges.

RULES OF JUDGMENT
1. APPLICABILITY FIRST. If the control's applicability conditions are not met by
   project_facts (missing required data tier, missing required tag, no Ledger
   registry when requires_ledger_registry is true), return status "SKIPPED" with a
   one-line reason. SKIPPED never counts against the score.
2. DATA CONTROLS KEY OFF CLASSIFICATION. For any control with a classification_tier,
   enumerate exactly the fields of that tier from data_classification and verify the
   required obligation/handling for EACH. Coverage gap on any field => PARTIAL (or
   MISSING if none covered).
3. PRESENT requires an enforcing mechanism you can cite: a code path, a guard on every
   relevant route, a config key with the required value, or a present-and-fresh evidence
   artifact (age_days <= the control's max_age_days).
4. PARTIAL means the mechanism exists but does not cover all in-scope targets (e.g. some
   routes unguarded, some tier fields unencrypted, evidence present but STALE).
5. MISSING means no enforcing mechanism found in the provided evidence.
6. For ORG/evidence controls (detection.method = evidence_doc): PRESENT only if exists
   AND fresh; STALE-but-present => PARTIAL; absent => MISSING. Report the owner.
7. NEVER reward "teaching to the test": a test file or a string literal naming the
   control does not by itself make the control PRESENT — the enforcing logic must exist.

OUTPUT — return ONLY this JSON, no prose, no fences:
{
  "results": [
    {
      "check_id": "<control.id>",
      "target": "<the most specific file:line or artifact path that is the evidence locus>",
      "status": "PASS|WARN|FAIL|SKIP",       // PASS=PRESENT, WARN=PARTIAL, FAIL=MISSING(if severity must) , SKIP=SKIPPED
      "severity": "FAIL|WARN|INFO",           // FAIL if severity=must, WARN if should, INFO if skipped
      "message": "<= 200 chars: the verdict + the single strongest cited evidence (path:line) or the gap>",
      "tool": "<framework slug>",
      "evidence": ["path:line", "..."],       // all loci you relied on; [] for SKIP/MISSING
      "uncovered_targets": ["<field or route that lacks the control>"],  // [] unless PARTIAL
      "confidence": "high|medium|low"
    }
  ]
}
```

A thin adapter in `compliance/llm.py` maps each `results[]` item onto a `CheckResult` (dropping `evidence/uncovered_targets/confidence` into `message`, or persisting them in the saved JSON report's `checks[].detail`). `status:"SKIP"` items are excluded from scoring exactly like mechanical skips.

---

## 4. THE VALIDATION TESTS + REGRESSION GATE

### 4.1 Generating executable tests for in-place controls

After a scan, the feature generates an executable test for every control whose verdict is PRESENT (and optionally PARTIAL, to pin the current covered set). This mirrors MEA's portable pattern: a control's `validation_test` block is the portable spec, rendered into a language-specific test file.

```bash
# Render executable tests for all PRESENT controls into the project's test dir
cartographer compliance gen-tests --out tests/compliance/

# Render only one framework
cartographer compliance gen-tests --framework gdpr --out tests/compliance/
```

Two render styles (`compliance/render/`), selected by `validation_test.style`:

- **`static_assertion`** → a pytest file in the style of MEA's `tests/test_constraints.py`: pure-Python, no app boot, using a small grep/registry toolkit (`_read`, `_grep`, `_load_ledger_registry`). One `class Test_<control.id>` per control; the docstring quotes the control; the method realizes the named `assertion` with `params`. The renderer ships a fixed library of named assertions (`assertions.py`): `ledger_annotation_has_companion_field`, `ledger_tier_all_encrypted`, `source_pattern_present`, `source_pattern_absent`, `every_route_has_guard`, `config_key_equals`, `evidence_present_and_fresh`, `control_id_referenced_in_code` (the generalized traceability check from MEA's `TestConstraintReferences`). These are deterministic templates — no LLM needed.
- **`goodhart_json`** → a portable `<control.id>_suite.json` (MEA's `goodhart_test_suite.json` schema: `component_id, contract_version, test_cases:[{id, description, function, category, assertions:[...]}]`) plus a rendered pytest. Because the `assertions` array is natural-language predicates, this render is the one place an LLM (or a human) is needed to turn predicates into code; the JSON suite is committed as the source-of-truth artifact and regenerated deterministically thereafter.

Generated tests are written under the project's chosen `--out` and are committed by the project. They are ordinary pytest files — they run under the project's existing `pytest`/`make test`.

### 4.2 Single regression command — "verify nothing compromised my compliance posture"

A baseline-diff gate. The scan writes a machine baseline; `verify` re-scans and fails if any previously-PASS control is no longer PASS, or the score drops.

```bash
# 1. Pin the current posture as the baseline (after a known-good scan)
cartographer compliance baseline --update      # writes compliance.baseline (per cartographer.yaml)

# 2. The regression gate (CI): re-scan, diff against baseline, fail-closed on regression
cartographer compliance verify
```

`verify` semantics (in `compliance/verify.py`):
1. Run `compliance scan` (mechanical; `--with-llm` honored if set).
2. Load `baseline.json` (the saved per-control statuses from the last `baseline --update`).
3. **Regression = any control PASS-in-baseline that is now WARN/FAIL/absent, OR `score_pct` < baseline `score_pct`.** Newly-added controls that are MISSING do *not* fail `verify` (they surface in `scan`) — `verify` guards against *regression*, not against work-not-yet-done. New PASS controls auto-promote the baseline only with `--update`.
4. Exit `1` on any regression (reuse the `sys.exit(1)` pattern); print a diff (control id, was→now). Exit `0` if posture held or improved.

This is the fail-closed CI gate. Wire it as a Make target in consumer projects (see §7 MEA adoption). Combined with the generated pytest tests from §4.1, a project gets two layers: the rendered tests assert each control's mechanism in the project's own suite, and `compliance verify` asserts the *aggregate posture never regressed*.

---

## 5. RISK REGISTRATION

Turns a described new risk exposure into: a registry control entry + a detection method + a generated test, appended to the project's controls dir (never overwriting packaged defaults). Reuses the draft→review→adopt loop conceptually: the AI drafts a `ControlDef`, a human reviews, `--adopt` appends it.

```bash
# Interactive / piped: describe a risk, get a drafted control written to the controls dir
cartographer compliance add-risk \
  --framework gdpr \
  --describe "We just started storing applicants' immigration status, which is GDPR special-category data; it must be encrypted at rest and excluded from CSV exports." \
  --dry-run                 # prints the drafted ControlDef YAML; does not write

# Adopt: append the reviewed draft as a new control + regenerate its test
cartographer compliance add-risk --framework gdpr --describe "..." --adopt --gen-test
```

Behavior (`compliance/add_risk.py`):
1. Gather project_facts (Ledger tiers/annotations, discovered routes, present configs, evidence index) — same facts the scan uses.
2. Send the description + facts + the `ControlDef` schema to the LLM with the prompt below.
3. Validate the returned YAML against the `ControlDef` Pydantic model. Reject (exit 2) if it does not validate or if `id` collides with an existing control.
4. `--dry-run`: print the YAML. `--adopt`: append to `compliance/controls/<framework>.local.yaml` (merged at load like a `load_patterns` override). `--gen-test`: run the §4.1 renderer for just the new control.
5. The new control flows into the next `scan`/`verify` automatically (no code change).

### AI PROMPT for risk registration (verbatim)

```
You are a compliance control author. Convert a described risk exposure into ONE
machine-checkable control record that conforms EXACTLY to the ControlDef schema given
below. You do not write application code; you produce a control definition and, where
possible, a MECHANICAL detection method so the control can be verified without a human
and without an LLM at scan time. Prefer mechanical detection; only choose llm_judgment
when no deterministic source/config/Ledger signal could possibly evidence the control.

INPUTS
- framework: <slug> (framework_version: <version>)
- risk_description: free text describing the new exposure.
- project_facts (ground truth — do not re-guess sensitive data):
    - data_classification: Ledger registry tiers + annotations per field.
    - discovered_routes, present_configs, evidence_index (as in the scan prompt).
    - existing_control_ids: ids already in the registry (your id MUST NOT collide).
- ControlDef schema (authoritative; your output must validate against it):
    id, family, framework_ref, title, description, severity (must|should),
    classification_tier (PUBLIC|PII|FINANCIAL|AUTH|COMPLIANCE|null),
    applicability {requires_ledger_registry?, data_tiers?, tags?},
    detection {method + method-specific fields}, corroborating_detection? [],
    validation_test {style (static_assertion|goodhart_json), assertion, params},
    evidence_owner (null for code-detectable; a role name for ORG controls),
    status (auto), references [].
- detection methods you may use and their required fields:
    ledger_obligation {tier, required_annotation, required_field_key}
    ledger_tier_encrypted {tier}
    source_grep {any_of? [regex], none_of? [regex], include_glob}
    route_guard {guard_patterns [regex]}
    config_present {path, required_key, required_value?}
    companion_config {tool}
    evidence_doc {evidence_path_key, max_age_days}
    llm_judgment {question}
- named assertions you may reference in validation_test.assertion:
    ledger_annotation_has_companion_field {annotation, companion, tier}
    ledger_tier_all_encrypted {tier}
    source_pattern_present {pattern, include_glob}
    source_pattern_absent {pattern, include_glob}
    every_route_has_guard {guard_patterns}
    config_key_equals {path, key, value}
    evidence_present_and_fresh {evidence_path_key, max_age_days}
    control_id_referenced_in_code {control_id}

RULES
1. Derive a stable, descriptive, UNIQUE id: <FRAMEWORK>-<SHORT-TOPIC> in caps, not
   colliding with existing_control_ids.
2. If the risk concerns specific data, bind classification_tier and detection to the
   ACTUAL fields/tiers in data_classification. If the relevant field is not yet in the
   Ledger registry, say so in the description and set detection to source_grep / config
   over the handling code, and add a corroborating ledger_obligation that will start
   passing once the field is registered.
3. severity = must when the risk is a hard legal/security obligation; should otherwise.
4. Choose the SIMPLEST detection that truly evidences the control. Encryption-at-rest of
   a tier => ledger_tier_encrypted. "Must not appear in exports/logs" => source_grep
   none_of over the export/log code. Route protection => route_guard. A required policy
   doc => evidence_doc with a sane max_age_days (<=365) and a named evidence_owner.
5. validation_test must match detection: choose the named assertion + params that re-
   express the same check as an executable test. Use static_assertion unless the control
   is behavioral/stateful, in which case use goodhart_json.
6. references: cite the specific files (from project_facts) the control governs.

OUTPUT — return ONLY the control as YAML (a single list item under `controls:`), no prose,
no code fences. It must parse and validate against ControlDef on the first try.
```

---

## 6. EXTENSIBILITY & GENERICITY

- **New control**: drop a YAML control document into `compliance/controls/*.yaml` (or use `add-risk`). Merged over packaged defaults at load (same merge mechanism as `load_patterns`). No code change.
- **New framework**: add `<slug>.yaml` (packaged or project) with `framework:` + `controls:`. The `--framework <slug>` filter and per-framework `FrameworkChecker` instantiation are data-driven from the loaded set — no new class needed for a framework that uses existing detection methods. All seven required frameworks ship as packaged defaults: **CJIS (FBI Security Policy v6.0), ISO/IEC 27001:2022 (Annex A), SOC 2 (Trust Services Criteria), HIPAA Security Rule, CCPA/CPRA (CPPA), GDPR, FedRAMP Rev 5.**
- **New detection method**: add an evaluator function to `compliance/detectors/` and register it in the `DETECTORS` dict (one-line append, mirrors `ALL_CHECKERS`). This is the only path that touches code, and only for genuinely new *kinds* of mechanical check.
- **New project**: provide a `cartographer.yaml` `compliance:` block (frameworks, controls_dir, tags, evidence_index). Controls reference Ledger tiers/annotations and project tags — never service names, never `C0NN` ids, never MEA paths.
- **Genericity guardrails (enforced, not aspirational)**: zero MEA-specific identifiers in packaged code or default frameworks. No hardcoded service names, no `services/<svc>/src/<svc>` convention, no `mea.*` event names, no `C0NN` ids. All project specifics arrive via config + the Ledger registry + the evidence index. The packaged framework YAMLs cite *framework* clauses (Art.17, §164.312, A.8.24, CC6, TSC, AC/IA/AU/SC) and generic detection, not MEA code.

**Explicit statement: this feature is project-agnostic. MEA is its first consumer, not its definition.** MEA supplies the data (its Ledger registry, its `classification_registry.yaml`-derived tiers, its evidence docs, its `cartographer.yaml`); Cartographer supplies the registry schema, the scan, the scoring, the test renderers, the regression gate, and the risk-registration loop.

---

## 7. ACCEPTANCE CRITERIA + CODEX HANDOFF

### 7.1 Definition of Done

1. New subpackage `src/cartographer/compliance/` with: `registry.py` (`ControlDef`, `FrameworkProfile`, merge-loader mirroring `load_patterns`), `runner.py` (`FrameworkChecker(CompatibilityChecker)` + framework iteration), `detectors/` (the 8 detection-method evaluators + `DETECTORS` registry), `render/` (`assertions.py` named-assertion library + static-assertion and goodhart-json renderers), `verify.py` (baseline diff), `add_risk.py`, `ingest.py` (readiness-matrix parser), `llm.py` (prompt invocation + `CheckResult` adapter).
2. Packaged default framework YAMLs for all 7 frameworks under `compliance/frameworks/`, each with ≥6 controls split across CODE-detectable and ORG/evidence controls per the framework coverage in the findings.
3. CLI: `cartographer compliance {scan, gen-tests, baseline, verify, add-risk}` with the flags in §3–§5; `scan`/`verify` reuse `build_report`/`format_*`/`save_report` and the `sys.exit(1)` fail-closed semantics.
4. `compliance scan` runs fully offline/mechanical with no API key; `--with-llm` is the only path that requires `ANTHROPIC_API_KEY`; LLM controls are SKIPPED+flagged (never FAIL) when LLM is off.
5. Ledger composition: `ledger_obligation`/`ledger_tier_encrypted` detectors resolve the registry via the existing `ledger_checker._find_registry` (refactor it to a shared `compliance`-importable helper or import directly). No reimplementation of classification.
6. `compliance verify` exits non-zero iff a baseline-PASS control regresses or `score_pct` drops; exits 0 if posture holds/improves; prints a was→now diff.
7. `add-risk --adopt` produces a `ControlDef`-valid entry appended to `controls_dir`, rejects schema-invalid or id-colliding drafts, and `--gen-test` renders the matching test.
8. Tests: unit tests for each detector (fixtures: a tiny synthetic Ledger registry, a synthetic source tree, present/absent/stale evidence files); a test that every packaged framework YAML loads and validates against `ControlDef`; a round-trip test that a rendered `static_assertion` test file is importable and passes against its fixture; a `verify` regression test (baseline pass → mutate → expect exit 1). All under `~/WanderRepos/repos/cartographer/tests/`, runnable via `pytest`.
9. No regression in existing Cartographer tests; `models.py`, `report/generator.py`, and the `check` exit logic are reused unchanged (new code only).
10. `prompt.md`/README updated with the two AI prompts and the `compliance` command reference.

### 7.2 Codex handoff

- **Repo / package**: `~/Code/cartographer` (GitHub: as configured for that repo, MIT). Branch off `main`; do not commit/push until the user asks.
- **File layout to create**:
  ```
  src/cartographer/compliance/
    __init__.py
    registry.py            # ControlDef, FrameworkProfile, load_frameworks(merge)
    runner.py              # FrameworkChecker(CompatibilityChecker), run_compliance(config, base_dir, frameworks)
    detectors/__init__.py  # DETECTORS registry + 8 evaluators
    render/__init__.py
    render/assertions.py   # named-assertion library (static)
    render/static.py       # pytest renderer (test_constraints.py style)
    render/goodhart.py      # *_suite.json + pytest renderer
    verify.py              # baseline diff gate
    add_risk.py
    ingest.py              # readiness-matrix + constraints.yaml + classification_registry.yaml parsers
    llm.py                 # prompt calls + CheckResult adapter
    frameworks/{cjis,iso27001,soc2,hipaa,ccpa,gdpr,fedramp}.yaml
  ```
  Plus: `ComplianceConfig` model added to `config/loader.py` `CartographerConfig`; `compliance` Click group added to `cli/main.py`; `compliance/` test dir under `tests/`.
- **Build/test**:
  ```bash
  pip install -e ~/Code/cartographer[dev]        # click, pydantic, pyyaml, pytest
  pip install -e ~/Code/cartographer[api]         # only if exposing compliance over HTTP later
  cd ~/WanderRepos/repos/cartographer && python3 -m pytest -q  # full suite (addopts -x -q per pyproject)
  cartographer compliance scan --help             # smoke the new command group
  ```
  LLM features use `ANTHROPIC_API_KEY` (per the user's `~/.profile`); add `anthropic` as an optional dependency group `[llm]` in `pyproject.toml`.
- **Pre-req to flag**: the installed `ledger` console-script entrypoint is broken (`ModuleNotFoundError: No module named 'cli.main'`; pyproject points at `cli.cli:cli_main`). The compliance feature reads the Ledger **registry files on disk**, so it does not depend on Ledger's CLI — but if any control ever shells out to `ledger export`, fix that entrypoint first.
- **How MEA adopts it** (first consumer, zero coupling):
  1. `cd ~/Code/MEA && cartographer init` already produces `cartographer.yaml`; add the `compliance:` block (frameworks `[cjis, iso27001, soc2, hipaa, ccpa, gdpr, fedramp, pcidss]`, `evidence_index` populated from `docs/compliance/compliance-readiness-matrix.md`, `project_tags` for CJI/EU/payment data).
  2. Optionally `cartographer compliance ingest` to parse the readiness matrix + `constraints.yaml` + `classification_registry.yaml` into seed controls under `compliance/controls/` for review.
  3. Point `stack.ledger_registry` at MEA's `.ledger/registry` so data-protection controls key off real classifications.
  4. `cartographer compliance scan --format json` → coverage report; `cartographer compliance gen-tests --out tests/compliance/` → executable controls; `cartographer compliance baseline --update` → pin posture.
  5. Add Make targets to MEA: `make compliance-scan` (`cartographer compliance scan --strict`) and `make compliance-verify` (`cartographer compliance verify`), and wire `compliance-verify` into CI alongside `make test-constraints` as a fail-closed gate.

### Key verified source anchors (Cartographer)
- `src/cartographer/compatibility/base.py:12` — `CompatibilityChecker` ABC (subclass for `FrameworkChecker`)
- `src/cartographer/compatibility/runner.py:17` — `ALL_CHECKERS` registry pattern (mirror in `compliance/runner.py`)
- `src/cartographer/models.py:123` `CheckResult`, `:132` `CompatibilityReport` (`score_pct`/`has_failures` reused unchanged)
- `src/cartographer/report/generator.py:12` `build_report`, `:74` `format_json`, `:101` `save_report` (reused unchanged)
- `src/cartographer/cli/main.py:135-162` — `check --strict --format json` + `sys.exit(1)` fail-closed gate (pattern for `scan`/`verify`)
- `src/cartographer/compatibility/ledger_checker.py:127` — `_find_registry` (reuse for Ledger composition); whole file is the worked control template
- `src/cartographer/discovery/sensitive_fields.py:95` — `load_patterns` merge mechanism (registry loader template)
- `src/cartographer/config/loader.py:60` `CartographerConfig`, `:45` `StackConfig.ledger_registry` (add `ComplianceConfig`)
- `src/cartographer/drafters/base.py` + `models.py:163` `DraftArtifact` — draft→adopt loop pattern for `add-risk`

### Key verified source anchors (MEA — first consumer / ingest sources, not coupled into the package)
- `/Users/jmcentire/Code/MEA/docs/compliance/compliance-readiness-matrix.md` — status ladder + framework matrix + evidence index (ingest source)
- `/Users/jmcentire/Code/MEA/constraints.yaml` and `/Users/jmcentire/Code/MEA/architecture/data_constraints.yaml` — constraint schema (ingest source)
- `/Users/jmcentire/Code/MEA/classification_registry.yaml`, `/Users/jmcentire/Code/MEA/trust_policy.yaml` — classification/retention facets
- `/Users/jmcentire/Code/MEA/tests/test_constraints.py` — static-assertion test style template (incl. `TestConstraintReferences` → `control_id_referenced_in_code`)
- `/Users/jmcentire/Code/MEA/services/cases/tests/compliance_verification/goodhart/goodhart_test_suite.json` — goodhart-json suite schema template
