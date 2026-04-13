# Runtime Config Contract Plan

Status: Ready to execute.
Priority: Highest.
Intent: Make runtime config machine-validated and normative, with prose docs as guidance only.

## 1) Problem

Runtime config semantics are currently distributed across code and prose. This creates drift risk between:

1. Runtime loader behavior.
2. Profile YAML files (`anolis/config/...`).
3. Tools that generate or edit config (composer, scripts, manual edits).

## 2) Goals

1. Define one machine-readable runtime config contract.
2. Validate every tracked runtime profile against that contract in CI.
3. Make breaking config changes explicit and versioned.

## 3) Non-Goals

1. No provider-specific semantics in core schema beyond generic config shape.
2. No immediate redesign of runtime config layout.
3. No forced migration of every experimental local config file.

## 4) Normative Artifacts

1. Runtime config schema:
   - Suggested path: `anolis/schemas/runtime-config.schema.json`
2. Provider config schema (if split is cleaner):
   - Suggested path: `anolis/schemas/provider-config.schema.json`
3. Schema README:
   - Suggested path: `anolis/schemas/README.md`

## 5) Required Coverage

Minimum schema coverage for this wave:

1. `runtime` block:
   - name, mode defaults, startup behavior hooks if present.
2. `providers[]` block:
   - id, command, args, restart policy fields.
3. `http` block:
   - enabled, host, port bounds, thread settings if supported.
4. `polling` block:
   - interval bounds and defaults.
5. `logging` block:
   - level enum.
6. `telemetry` block:
   - enabled, sink config structure, required fields when enabled.
7. `automation` block:
   - enabled, behavior path, tick rate bounds, parameter map structure.
8. Cross-field invariants:
   - required sections when feature flags are enabled.
   - numeric ranges (`>= 0`, bounds where known).

## 6) Execution Steps

1. Contract inventory:
   - Enumerate all keys currently used by loader + active configs.
2. Schema draft:
   - Capture current behavior first (do not “improve” semantics yet).
3. Validator wiring:
   - Add local validation command (script or CMake target).
4. CI gate:
   - Validate all tracked runtime configs.
5. Migration pass:
   - Fix or annotate outlier configs.
6. Policy lock:
   - Require schema update for config-surface changes.

## 7) CI and Local Gates

Required checks:

1. Schema validates itself.
2. All tracked `anolis/config/**/*.yaml` runtime profiles pass schema validation.
3. Failure output includes file path + key path + reason.

Suggested local workflow:

1. `verify-local` should include schema validation.
2. Fast path command for config-only changes.

## 8) Versioning and Change Policy

1. Schema has explicit version field (`x-schema-version` or equivalent).
2. Breaking changes require:
   - changelog entry,
   - migration note,
   - profile updates in same PR.
3. Additive fields are allowed with defaults.

## 9) Exit Criteria

Done when:

1. Schema exists and is treated as normative.
2. CI blocks invalid configs.
3. Bioreactor profiles validate cleanly.
4. Docs point to schema as source of truth.

## 10) Risks and Mitigations

1. Risk: Schema under-specifies runtime behavior.
   - Mitigation: add fixture configs and runtime load tests.
2. Risk: Strictness blocks iteration.
   - Mitigation: allow optional experimental namespace with guardrails.
3. Risk: Duplicate constraints in code + schema drift again.
   - Mitigation: centralize validation and add contract tests in CI.

