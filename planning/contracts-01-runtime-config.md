# Runtime Config Contract Plan

Status: Ready to execute.
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

1. Unknown keys: warned and ignored (root and known map sections).
2. Deprecated but accepted:
   - `automation.behavior_tree_path` alias for `automation.behavior_tree`.
   - flat telemetry keys under `telemetry.*` (`influx_url`, `influx_org`, `influx_bucket`, `influx_token`, `batch_size`, `flush_interval_ms`) with deprecation warnings.
3. Rejected legacy key:
   - `runtime.mode` is a hard error.
4. Semantic checks are enforced in code:
   - ranges, required sections, provider uniqueness, restart-policy consistency, automation invariants, hook structure.

## 4) Contract Model (Two-Layer Validation)

We will use two required layers, not schema-only:

1. Static schema layer:
   - JSON Schema validates structural/type/range shape and most cross-field rules.
   - Schema runs in compatibility mode for this wave: unknown keys remain allowed so behavior matches runtime's warn-and-ignore loader semantics.
2. Runtime semantic layer:
   - `anolis-runtime --check-config` validates loader behavior and semantic/runtime-coupled checks.

Reason: runtime currently includes compatibility and semantic behavior that cannot be fully or safely encoded as static schema alone without divergence risk.

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

## 11) CI and Local Gates

Required CI gates:

1. Schema file is syntactically valid.
2. Runtime config fixture suite passes.
3. All tracked runtime YAML files pass schema and `--check-config`.
4. Failure output includes exact file and failing field/rule.

Required local workflow:

1. Single command to run runtime config contract checks.
2. `verify-local` includes runtime config contract check by default.

## 12) Acceptance Criteria

Done when all are true:

1. Runtime config schema exists and is used in CI.
2. All tracked runtime YAML profiles validate cleanly.
3. Contract tests cover compatibility aliases and reject hard-invalid keys.
4. Docs clearly state:
   - runtime schema + loader checks are authoritative,
   - provider-local configs are out of this contract scope.

## 13) Risks and Mitigations

1. Risk: Schema and loader diverge over time.
   - Mitigation: dual-layer gate (schema + `--check-config`) and fixture requirements.
2. Risk: Overly broad file discovery causes false failures.
   - Mitigation: strict runtime-file target patterns only.
3. Risk: Future cleanup of deprecated aliases breaks existing configs abruptly.
   - Mitigation: explicit deprecation window with fixture and changelog requirements.
