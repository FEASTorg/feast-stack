# UI and Composer Contract Alignment Plan

Status: Ready after runtime config + HTTP contracts land.
Priority: High (immediately after contract foundations).
Intent: Remove drift between operator UI, system composer output, and runtime accepted surfaces.

## 1) Problem

UI and composer currently depend on inferred runtime behavior. This causes recurring friction:

1. Type mismatches (bool/int/string handling).
2. Feature drift between generated configs and runtime loader expectations.
3. Endpoint payload assumptions in UI that are not contract-checked.

## 2) Goals

1. Bind composer output to runtime config schema.
2. Bind UI HTTP client behavior to OpenAPI contract.
3. Add fixture-based CI tests to block contract drift.

## 3) Non-Goals

1. No immediate repo split for UI/composer.
2. No full UI rewrite.
3. No machine-specific workflow logic in generic UI core.

## 4) Alignment Model

1. Runtime config is normative through schema.
2. HTTP payloads are normative through OpenAPI.
3. UI and composer must pass compatibility checks against both artifacts.

## 5) Composer Work Items

1. Explicit mapping contract:
   - source: composer project file(s)
   - target: runtime YAML (schema-valid)
2. Fixture suite:
   - canonical input fixtures
   - expected output YAML fixtures
3. Determinism:
   - same input must generate byte-stable (or semantically stable) output.
4. CI gate:
   - generated output validates against runtime config schema.

## 6) Operator UI Work Items

1. Typed parameter rendering driven by declared runtime value types:
   - bool -> checkbox/select,
   - numeric -> numeric input,
   - string -> text input.
2. HTTP client:
   - request/response handling aligned with OpenAPI response shapes.
3. View stability:
   - preserve input focus/edit session on background refresh.
4. Error UX:
   - map API errors to actionable operator messages.

## 7) Cross-Tool Contract Tests

1. UI API fixture tests:
   - validate parsing/rendering for representative `/v0` responses.
2. Composer integration tests:
   - generate config -> schema validation -> runtime load smoke.
3. Regression fixtures:
   - include current bioreactor profile as pinned baseline.

## 8) CI and Local Gates

Required CI:

1. Composer output fixtures pass.
2. Generated runtime configs pass schema validation.
3. UI contract fixtures pass against current OpenAPI models.

Suggested local commands:

1. Config contract check.
2. HTTP contract check.
3. UI/composer fixture check.

## 9) Exit Criteria

Done when:

1. Composer can only emit schema-valid runtime configs.
2. UI typed controls are contract-driven and stable.
3. Drift is caught by CI before merge.

## 10) Risks and Mitigations

1. Risk: Legacy UI paths bypass typed handling.
   - Mitigation: remove legacy fallback paths once fixtures are green.
2. Risk: Composer supports knobs runtime no longer accepts.
   - Mitigation: schema-backed generation and failure-on-unknown fields.
3. Risk: Runtime changes outpace fixtures.
   - Mitigation: require fixture updates in same PR as surface changes.

