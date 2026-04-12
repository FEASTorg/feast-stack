# Bioreactor Machine Rollout Plan (Transient)

Status: Working rollout sequence (final)
Scope: Move from current manual mixed-bus operation to safe staged automation without violating runtime architecture boundaries.

## 1) Current Baseline

Validated:

1. Shared-bus runtime with `bread0` + `ezo0` providers is operational.
2. 5-device topology is visible and controllable.
3. Manual/operator workflows are working.
4. DO percent output commissioning issue has been resolved in hardware setup.

Known caveat:

1. Intermittent low-rate BREAD read/write errors remain.
2. Conservative timing is currently the mitigation.
3. Root-cause investigation remains a separate reliability track.

## 2) Hard Boundary Policy

1. `anolis` core is general runtime only.
2. Core code cannot contain provider or machine semantics.
3. Machine behavior is expressed in machine BT/config only.
4. Core additions must be reusable across providers/machines.

## 3) Rollout Phases

### Phase A - Foundation (Core Generic)

Deliver minimal generic primitives required for maintainable machine automation:

1. `GetParameterBool`.
2. `PeriodicPulseWindow`.
3. `EmitOnChangeOrInterval` (or equivalent generic throttling/keepalive primitive).
4. `BuildArgsJson` generic args builder for `CallDevice`.
5. Generic runtime transition hooks with explicit `before_transition` / `after_transition` timing.

Gate A:

1. Unit tests pass for each primitive.
2. Boundary audit confirms no provider semantics in core.

### Phase B - Stage 1 Machine Automation (Stir + Feed)

1. Implement machine BT using `dcmt0` dual-channel atomic writes.
2. Validate feed schedule and impeller enforcement behavior.
3. Validate transition-hook handoff behavior.

Gate B:

1. Stable repeated runs in lab.
2. No channel clobbering.
3. Deterministic handoff on required transitions.

### Phase C - Stage 2 Machine Automation (pH Acid/Base)

1. Add deadband pulse dosing on `dcmt1` using `ph0` quality-gated signal.
2. Enforce one-sided dosing and lockout windows.
3. Validate disable/recovery and safety behavior.

Gate C:

1. No simultaneous acid/base drive.
2. No chatter under steady-state noise.
3. Recoverability verified under fault injection.

### Phase D - Packaging and Promotion

1. Package machine assets under `feast-stack/machines/bioreactor/`.
2. Include pinned runtime/provider config, runbook, scripts, and evidence template.
3. Re-run full validation from package entrypoint.

Gate D:

1. Clean bootstrap from package docs only.
2. No ad-hoc edits required.

## 4) Execution Order

1. Land Phase A first.
2. Run read-only BT sanity pass (no actuation).
3. Execute Phase B.
4. Execute Phase C.
5. Complete Phase D packaging.

## 5) Release Gate Checklist

Before calling automation rollout ready:

1. Architecture boundary audit passes.
2. Generic primitive tests pass in CI.
3. Stage 1 and Stage 2 hardware validation evidence is captured.
4. Operator manual takeover path is validated and documented.
5. Reliability caveats are either resolved or explicitly accepted with mitigations.
