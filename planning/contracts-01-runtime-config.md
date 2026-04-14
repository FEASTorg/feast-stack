# Runtime Config Contract Plan

Status: In progress (hardening/close-out required before moving to contracts-02).
Priority: Highest.
Intent: Define and enforce one authoritative runtime-config contract for Anolis.

## 1) Problem

Runtime config behavior currently lives in multiple places:

1. Loader and validator code (`core/runtime/config.cpp`, `config.hpp`).
2. Prose docs (`docs/configuration.md`, `docs/configuration-schema.md`).
3. Runtime profile YAML files (`config/**/anolis-runtime*.yaml`, composer outputs).

This creates drift and repeated regressions in config authoring, validation, and tooling.

## 2) Scope and Boundaries

In scope for this contract:

1. Runtime YAML consumed by `anolis-runtime --config` / `--check-config`.
2. Runtime top-level sections: `runtime`, `http`, `providers`, `polling`, `telemetry`, `logging`, `automation`.
3. Compatibility behavior currently implemented by runtime loader (warnings, deprecated aliases, strict failures).

Out of scope for this document:

1. Provider-local YAML schemas (`provider-bread*.yaml`, `provider-ezo*.yaml`, etc.).
2. Telemetry exporter YAML (`telemetry-export*.yaml`).
3. HTTP API contract details (handled by `contracts-02-runtime-http.md`).

## 3) Authoritative Behavior Snapshot (Current Runtime)

This plan is based on current runtime behavior and must preserve it first.

1. Unknown keys: warned and ignored (root and most nested map sections).
2. Deprecated but accepted:
   - `automation.behavior_tree_path` alias for `automation.behavior_tree`.
   - flat telemetry keys under `telemetry.*` (`influx_url`, `influx_org`, `influx_bucket`, `influx_token`, `batch_size`, `flush_interval_ms`) with deprecation warnings.
3. Rejected legacy key:
   - `runtime.mode` is a hard error.
4. Semantic checks are enforced in code:
   - ranges, required sections, provider uniqueness, restart-policy consistency, automation invariants, hook structure.

Notes from implementation audit:

1. Unknown keys are warned at root and many nested sections, but warning coverage is not uniform (`polling` and `logging` currently do not emit unknown-key warnings).
2. `function_id == 0` is treated as "not specified" in runtime call resolution; config entries using only `function_id: 0` are runtime-invalid unless `function_name` is also supplied.
3. `--check-config` validates load-time semantics only; runtime-initialization-time behaviors (for example duplicate automation parameter name redefinition warnings from `ParameterManager`) are outside the current contract gate.

## 4) Contract Model (Two-Layer Validation)

We will use two required layers, not schema-only:

1. Static schema layer:
   - JSON Schema validates structural/type/range shape and most cross-field rules.
   - Schema runs in compatibility mode for this wave: unknown keys remain allowed so behavior matches runtime's warn-and-ignore loader semantics.
2. Runtime semantic layer:
   - `anolis-runtime --check-config` validates loader behavior and semantic/runtime-coupled checks.

Reason: runtime currently includes compatibility and semantic behavior that cannot be fully or safely encoded as static schema alone without divergence risk.

Critical caveat (must be addressed in close-out):

1. Current schema layer uses Python YAML parsing (`yaml.safe_load`), while runtime uses `yaml-cpp`.
2. These parsers are not behavior-identical for important edge cases (duplicate keys and YAML scalar resolution), so parser-alignment work is required for truly authoritative two-layer validation.

## 5) Normative Artifacts

1. Runtime schema:
   - `anolis/schemas/runtime-config.schema.json`
2. Schema README:
   - `anolis/schemas/README.md`
3. Runtime-config fixture set:
   - `anolis/tests/contracts/runtime-config/valid/*.yaml`
   - `anolis/tests/contracts/runtime-config/invalid/*.yaml`
4. Contract validation script:
   - `anolis/tools/contracts/validate-runtime-configs.py`

## 6) Validation Target Set

Only runtime YAML files are validated by this contract gate:

1. `anolis/config/anolis-runtime*.yaml`
2. `anolis/config/**/anolis-runtime*.yaml`
3. `anolis/systems/**/anolis-runtime.yaml`

Explicitly excluded:

1. `anolis/config/**/provider-*.yaml`
2. `anolis/config/**/telemetry-export*.yaml`
3. Any non-runtime YAML under tools/tests unless explicitly listed as fixtures.

## 7) Required Runtime Surface Coverage

The schema and fixtures must cover all currently supported runtime keys.

1. `runtime`:
   - `name`, `shutdown_timeout_ms`, `startup_timeout_ms`
   - reject `mode`
2. `http`:
   - `enabled`, `bind`, `port`, `cors_allowed_origins` (string or array), `cors_allow_credentials`, `thread_pool_size`
3. `providers[]`:
   - `id`, `command`, `args`, `timeout_ms`, `hello_timeout_ms`, `ready_timeout_ms`
   - `restart_policy.enabled`, `max_attempts`, `backoff_ms`, `timeout_ms`, `success_reset_ms`
4. `polling`:
   - `interval_ms`
5. `logging`:
   - `level` enum
6. `telemetry`:
   - `enabled`
   - nested `influxdb.url|org|bucket|token|batch_size|flush_interval_ms|max_retry_buffer_size|queue_size`
   - deprecated flat key compatibility documented and tested
7. `automation`:
   - `enabled`, `behavior_tree`, `tick_rate_hz`, `manual_gating_policy`
   - `parameters[]`: `name`, `type`, `default`, `min`, `max`, `allowed_values`
   - `mode_transition_hooks.before_transition|after_transition`
   - hook fields: `from`, `to`, `fail_on_error`, `calls[]`
   - call fields: `device_handle`, `function_id|function_name`, `args` scalar values

## 8) Cross-Field and Semantic Rules

Required rules in contract tests (schema where possible, runtime check always):

1. `providers` must be present and non-empty.
2. Provider IDs must be unique.
3. If restart policy enabled:
   - `max_attempts >= 1`
   - `backoff_ms` non-empty
   - `len(backoff_ms) == max_attempts`
   - each `backoff_ms[i] >= 0`
   - `timeout_ms >= 1000`
   - `success_reset_ms >= 0`
4. HTTP:
   - `1 <= port <= 65535`
   - `thread_pool_size >= 1`
   - `cors_allowed_origins` non-empty
   - wildcard origin forbidden when `cors_allow_credentials=true`
5. Polling:
   - `interval_ms >= 100`
6. Runtime:
   - `500 <= shutdown_timeout_ms <= 30000`
   - `5000 <= startup_timeout_ms <= 300000`
7. Automation enabled:
   - `behavior_tree` required
   - `1 <= tick_rate_hz <= 1000`
   - parameter type/default consistency
   - duplicate parameter names are currently compatibility behavior (accepted at load, later definitions ignored by parameter manager); do not introduce a breaking uniqueness check in this wave
   - numeric constraints valid only for numeric parameter types
   - string `allowed_values` only for string parameter type
   - hooks have valid `from`/`to` values and non-empty `calls`
   - each call has `device_handle` and one of `function_id` or `function_name`
   - hook arg scalar parsing follows current loader behavior (bool/int/float literal detection, else string); fixture set must include edge cases for quoted numeric-like strings

## 9) Compatibility and Change Policy

Policy for config surface evolution:

1. Compatibility-first for this wave:
   - keep deprecated aliases accepted with warnings.
2. Contract change types:
   - additive (safe): new optional fields with defaults.
   - constrained additive: new field with required sibling condition.
   - breaking: removal/rename/meaning change of existing field.
3. Breaking changes require in one PR:
   - schema update,
   - runtime loader update,
   - fixture updates,
   - migration notes in docs/changelog.
4. No config `schema_version` field added to runtime YAML in this wave.
   - Schema versioning remains artifact-level (schema file version metadata, changelog, tags).
5. Unknown-key behavior remains compatibility-preserving in this wave.
   - Typos and unknown keys are still warned by runtime and not hard-failed by the schema gate.

## 10) Implementation Plan

1. Baseline inventory lock:
   - capture current loader behavior and keys from `config.cpp` and tests.
2. Author schema:
   - encode current behavior, no redesign.
3. Build fixture pack:
   - valid/invalid YAML set covering all rules and compatibility aliases.
4. Add validator script:
   - discover runtime files from target set
   - run schema validation
   - run `anolis-runtime --check-config` for each
   - fail fast with clear message if `anolis-runtime` binary is not available
   - aggregate readable errors (`file`, `path`, `reason`)
5. Wire local check:
   - integrate into `tools/verify-local.sh`.
6. Wire CI check:
   - add dedicated job/lane for runtime config contract validation.
7. Align docs:
   - keep prose docs, but point to schema + loader tests as source of truth.

## 10A) Critical Audit Findings (Hard-Nosed)

1. Parser divergence is a real contract risk.
   - Empirical local check: `yaml.safe_load` accepts duplicate keys and applies last-key-wins behavior.
   - Empirical local check: runtime `yaml-cpp` accepts duplicate keys without hard failure and does not match `PyYAML` behavior for duplicate resolution.
   - Empirical local check: unquoted YAML 1.1 boolean-like tokens (for example `on`) are resolved differently by `PyYAML` vs runtime behavior.
2. Duplicate keys are currently silently accepted in the contract path.
   - This weakens contract safety because "same-looking file" can validate differently depending on parser.
3. Schema draft/keyword expectations are mixed.
   - Current schema uses draft-07 validator but includes `deprecated` annotations.
   - `deprecated` is a metadata keyword introduced in later drafts (2019-09); with a draft-07 validator, it is non-enforced metadata at best.
4. Fixture coverage is good for baseline semantics but still thin on parser and selector edge cases.
   - No fixture currently locks duplicate-key behavior.
   - No fixture currently locks `mode_transition_hooks.calls[].function_id: 0` runtime-invalid semantics.
5. Contract script currently does not meta-validate the schema against its declared draft before instance validation.

## 10B) Blocking Close-Out Work (Must Finish Before contracts-02)

1. Parser-alignment hardening:
   - Replace schema-layer YAML loading with a YAML 1.2-capable loader that can reject duplicate keys deterministically (or add an explicit pre-parse duplicate-key gate).
   - Add tests that prove duplicate-key configs are rejected by contract validation.
2. Add explicit schema meta-validation:
   - Run meta-schema check for `runtime-config.schema.json` in the validator script before validating instances.
3. Add edge fixtures:
   - `invalid/runtime`: `mode_transition_hooks` call with `function_id: 0` and no `function_name` (must fail runtime check).
   - `invalid/schema` (or dedicated parse-invalid bucket): duplicate-key config fixture once duplicate-key rejection gate is added.
   - Parser-resolution fixture(s) for ambiguous YAML scalars to prevent unintentional parser drift.
4. Tighten behavior documentation:
   - Document that `--check-config` is load-time contract validation, not full runtime-init validation.
   - Document exact unknown-key warning behavior (or standardize implementation to warn across all known map sections).
5. Decide and lock schema draft strategy:
   - Option A: keep draft-07 and treat `deprecated` as documentation-only annotation.
   - Option B: migrate validator/schema to newer draft where metadata vocabulary is explicit.
   - Record decision in `schemas/README.md` and this plan before close-out.

## 10C) Decision Locks (Set Before Implementation)

These decisions are now locked for contracts-01 close-out unless explicitly changed in this document:

1. Test architecture:
   - Keep Python as orchestration layer (file discovery, schema checks, reporting).
   - Keep runtime/C++ (`anolis-runtime --check-config`) as semantic authority.
   - Do not rely on Python YAML semantics as authoritative.
2. Parser-alignment strategy for this wave:
   - Implement deterministic duplicate-key rejection in contract tooling.
   - Add parser-sensitive fixtures so parser drift is observable in CI.
   - Do not attempt a full runtime parser rewrite in this wave.
3. Schema draft strategy for this wave:
   - Keep draft-07 for compatibility/stability.
   - Treat `deprecated` as non-enforcing annotation and document this explicitly.
   - Revisit draft migration in a later scoped follow-up, not inside contracts-01 close-out.

## 10D) Execution Slices (Ordered Close-Out)

Slice 1: Parser guardrail.
1. Add duplicate-key detection gate to contract validation path.
2. Add failing fixture(s) proving duplicate-key rejection.
3. Ensure clear failure output with file + key path context.

Slice 2: Runtime semantic edge coverage.
1. Add `invalid/runtime` fixture for `function_id: 0` without `function_name`.
2. Add parser-sensitive fixtures for ambiguous scalar literals.
3. Confirm fixtures pass/fail exactly as intended via runtime binary checks.

Slice 3: Schema gate hardening.
1. Add schema meta-validation (against declared draft/meta-schema) before instance checks.
2. Keep instance validation behavior unchanged otherwise.
3. Document the draft-07 + `deprecated` annotation semantics in schema docs.

Slice 4: Documentation normalization.
1. Align baseline docs with audited behavior (unknown-key warning coverage and parser authority note).
2. Clarify `--check-config` scope as load-time semantics (not full runtime-init behavior).
3. Confirm local + CI workflow docs match actual gate order and behavior.

Slice 5: Final contract close-out verification.
1. Run full contract script against tracked runtime configs + fixtures.
2. Run local verification script to ensure integrated flow is stable.
3. Confirm CI lane is green with new guardrails.

## 11) CI and Local Gates

Required CI gates:

1. Schema file is syntactically valid.
2. Runtime config fixture suite passes.
3. All tracked runtime YAML files pass schema and `--check-config`.
4. Failure output includes exact file and failing field/rule.
5. Contract gate rejects duplicate-key YAML (after parser-alignment slice lands).
6. Schema meta-validation gate runs before instance checks.

Required local workflow:

1. Single command to run runtime config contract checks.
2. `verify-local` includes runtime config contract check by default.
3. Local contract check behavior is parser-consistent with CI (same loader/settings).

## 12) Acceptance Criteria

Done when all are true:

1. Runtime config schema exists and is used in CI.
2. All tracked runtime YAML profiles validate cleanly.
3. Contract tests cover compatibility aliases and reject hard-invalid keys.
4. Docs clearly state:
   - runtime schema + loader checks are authoritative,
   - provider-local configs are out of this contract scope.
5. Parser-alignment close-out complete:
   - duplicate keys are rejected deterministically in contract validation.
   - parser-sensitive fixtures are present and enforced in CI.
6. Schema meta-validation is enforced.
7. Decision locks in section 10C are reflected in implementation and docs.

## 12A) No-Go Criteria (Do Not Start contracts-02 If Any Hold)

1. Duplicate keys can still pass contract validation.
2. Parser-sensitive fixtures are absent or flaky across environments.
3. Schema meta-validation is missing.
4. Baseline docs still state behavior that differs from audited runtime implementation.

## 13) Risks and Mitigations

1. Risk: Schema and loader diverge over time.
   - Mitigation: dual-layer gate (schema + `--check-config`) and fixture requirements.
2. Risk: Overly broad file discovery causes false failures.
   - Mitigation: strict runtime-file target patterns only.
3. Risk: Future cleanup of deprecated aliases breaks existing configs abruptly.
   - Mitigation: explicit deprecation window with fixture and changelog requirements.
4. Risk: Parser drift (schema-layer vs runtime-layer) creates false pass/fail outcomes.
   - Mitigation: parser-alignment slice + duplicate-key rejection + parser edge fixtures.

## 14) External Hard Facts (for Decisions)

1. YAML requires unique mapping keys.
   - YAML 1.2/1.2.2 spec materials state mapping keys are unique and duplicates are an error/non-portable behavior.
   - Reference: https://yaml.org/spec/1.2.0/
2. JSON object duplicate-name behavior is not reliably interoperable.
   - RFC guidance states object names SHOULD be unique and duplicate-name behavior is implementation-dependent.
   - Reference: https://www.rfc-editor.org/rfc/rfc8259
3. JSON Schema `deprecated` is a metadata/annotation concept introduced in later drafts (2019-09 metadata vocabulary), not a validation assertion in draft-07.
   - Reference: https://json-schema.org/draft/2020-12

## 15) Drift-Elimination Control Matrix

Each drift class must have a primary control and at least one backstop:

1. Shape/type drift:
   - Primary: JSON Schema validation.
   - Backstop: runtime `--check-config` on all tracked runtime YAML files.
2. Semantic drift:
   - Primary: runtime `--check-config`.
   - Backstop: `invalid/runtime` fixture coverage for each known semantic rule family.
3. Parser drift:
   - Primary: duplicate-key rejection + parser-sensitive fixtures.
   - Backstop: runtime parser authority documented and enforced in validation flow.
4. Documentation drift:
   - Primary: baseline docs updated as part of close-out slices.
   - Backstop: local/CI workflow docs tied to the same contract command path.
5. Scope drift (wrong files accidentally validated):
   - Primary: strict runtime-file discovery patterns.
   - Backstop: explicit exclusions and fixture directory contracts.

## 16) Immediate Remaining Items (Execution-Ready Checklist)

1. Implement duplicate-key rejection in contract tooling and add failing fixture.
2. Add `invalid/runtime` fixture for hook call selector edge (`function_id: 0`, missing `function_name`).
3. Add schema meta-validation in validator script.
4. Update docs:
   - `schemas/README.md`
   - `docs/contracts/runtime-config-baseline.md`
   - `docs/local-verification.md` (if execution flow text drifts again)
5. Re-run contract gate locally and in CI; close only when all no-go criteria are clear.
