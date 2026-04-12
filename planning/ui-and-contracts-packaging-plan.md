# UI and Contracts Packaging Plan (Transient)

Status: Working draft (finalized for current phase)
Purpose: Reduce drift across runtime, machine configs, operator UI, and composer before any repo-boundary split.

## 1) Problem Statement

Four coupled drift risks remain:

1. Tool packaging boundary is unresolved (`operator-ui` and `system-composer` currently in `anolis/tools/`).
2. Runtime config contract is prose-heavy and vulnerable to drift.
3. HTTP API contract is prose-heavy and vulnerable to drift.
4. Automation machine-contract boundary is not yet formalized end-to-end.

## 2) Current Position

1. Keep `operator-ui` and `system-composer` in `anolis` for now.
2. Prioritize contract hardening before any repo split.
3. Treat machine BT/config as first-class contract assets, not ad-hoc files.

## 3) Contract Hardening Scope

### 3.1 Runtime config schema (normative)

1. Add machine-readable runtime schema.
2. Encode required sections, types, ranges, enums, and invariants.
3. Enforce schema validation in CI and local verification.
4. Keep prose docs explanatory; schema becomes normative.

### 3.2 Composer-to-runtime alignment

1. Define explicit transform contract (`system.json` -> runtime YAML).
2. Add fixture tests for generated runtime configs.
3. Block CI on mapping drift.

### 3.3 HTTP API contract (OpenAPI)

1. Add OpenAPI spec for active `/v0` endpoints.
2. Lint spec in CI.
3. Keep examples aligned with real runtime responses.

### 3.4 Automation/machine boundary contract

1. Formalize ownership:
   - Core runtime: generic primitives only.
   - Machine profile: provider/device IDs, function names, channel mapping.
2. Add a boundary check in CI/local verification to detect provider-specific strings in core automation code.
3. Add runtime schema entries for transition hooks (`before_transition` / `after_transition`) as generic structures.
4. Version machine profile assets as part of machine package promotion.

## 4) Packaging Decision Gates

Do not split tools out of `anolis` until all pass:

1. Runtime schema is CI-enforced.
2. Composer-to-runtime transform tests are stable.
3. OpenAPI lint gate is green.
4. Automation boundary checks are in place.
5. At least one full machine profile runbook executes cleanly from package entrypoint.

## 5) Execution Order

1. Runtime schema + validation.
2. Composer transform alignment tests.
3. OpenAPI baseline + lint.
4. Automation boundary checks + transition-hook schema coverage.
5. Re-evaluate packaging split decision.

## 6) Out of Scope

1. Immediate multi-repo tool split.
2. Broader auth redesign.
3. API versioning policy expansion beyond current v0 baseline.
