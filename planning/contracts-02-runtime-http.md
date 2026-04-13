# Runtime HTTP Contract Plan

Status: Ready to execute.
Priority: High.
Intent: Make the `/v0` HTTP surface explicit, testable, and stable enough for UI/composer/tooling integration.

## 1) Problem

The HTTP surface is primarily documented in prose and inferred from runtime behavior. This creates drift risk for:

1. Operator UI.
2. Validation scripts.
3. External tooling/integration.

## 2) Goals

1. Create a normative OpenAPI spec for active `/v0` endpoints.
2. Add CI lint + conformance checks.
3. Ensure examples and docs match actual runtime responses.

## 3) Non-Goals

1. No API redesign in this wave.
2. No version bump from `/v0`.
3. No auth model expansion in this workstream.

## 4) Normative Artifacts

1. OpenAPI document:
   - Suggested path: `anolis/docs/http/openapi.v0.yaml`
2. HTTP contract README:
   - Suggested path: `anolis/docs/http/README.md`
3. Captured golden examples:
   - Suggested path: `anolis/docs/http/examples/`

## 5) Required Endpoint Coverage

Minimum set to specify now:

1. `GET /v0/runtime/status`
2. `GET /v0/providers/health`
3. `GET /v0/devices`
4. `GET /v0/devices/{provider_id}/{device_id}/capabilities`
5. `GET /v0/state`
6. `GET /v0/state/{provider_id}/{device_id}`
7. `POST /v0/call`
8. `GET /v0/mode`
9. `POST /v0/mode`
10. `GET /v0/parameters`
11. `POST /v0/parameters`
12. Automation status/tree endpoints currently shipped.

## 6) Execution Steps

1. Surface inventory from route registration and handlers.
2. OpenAPI draft mirroring current behavior (status codes, payload shape, enums).
3. Add lint gate (spectral or equivalent).
4. Add contract checks:
   - schema validation for example payloads,
   - smoke conformance against a running runtime profile.
5. Update docs to point to OpenAPI as source of truth.

## 7) CI and Local Gates

Required checks:

1. OpenAPI lint passes.
2. OpenAPI validates structurally.
3. Example JSON payloads validate against schema components.
4. At least one runtime smoke job verifies core endpoints return contract-compatible payloads.

Suggested local workflow:

1. Fast command for lint + example validation.
2. Optional full runtime conformance check for changed handlers.

## 8) Backward Compatibility Policy (v0)

1. Additive fields are allowed.
2. Existing fields cannot change meaning without explicit migration notes.
3. Removed or renamed fields require explicit deprecation window (even in v0).

## 9) Exit Criteria

Done when:

1. OpenAPI exists and is linked from runtime docs.
2. CI blocks drift between handlers/examples/spec.
3. Operator UI and validation scripts consume the same contract assumptions.

## 10) Risks and Mitigations

1. Risk: Spec drifts from implementation.
   - Mitigation: handler-level conformance tests and generated examples.
2. Risk: Over-specifying unstable internals.
   - Mitigation: only include fields with user-facing intent; keep internal fields optional where needed.
3. Risk: Event/SSE payloads are underdefined.
   - Mitigation: mark SSE as provisional subsection with explicit TODO and validation target.

