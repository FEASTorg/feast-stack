# Bioreactor Automation Plan - Stage 2 (pH Acid/Base Extension)

Status: Planned extension (final)
Scope: Add guarded pH automation on the acid/base pump pair only after Stage 1 is stable.

## 1) Boundary and Dependency Rules

1. All Stage 1 architecture rules remain in force.
2. No provider/device/function semantics are added to `anolis` core.
3. Stage 2 starts only after Stage 1.0 foundation + Stage 1 validation are complete.

## 2) Preconditions

1. Stage 1 repeatable hardware validation passed.
2. Transition-hook handoff behavior verified.
3. pH measurement commissioned and stable.
4. Manual acid/base direction checks completed in `MANUAL` mode.
5. Operators are comfortable with `AUTO -> MANUAL` recovery.

## 3) Control Goal

Use conservative deadband pulse dosing:

1. If pH > upper band: acid pulse.
2. If pH < lower band: base pulse.
3. If inside band: no pH dosing.

This remains pulse/deadband control, not PID.

## 4) Machine Mapping (Machine Assets Only)

Bioreactor-specific mapping:

1. `dcmt1/ch1` = acid pump.
2. `dcmt1/ch2` = base pump.
3. `ezo0/ph0` = control signal (`ph.value`).

Invariants:

1. Never command acid and base simultaneously.
2. Use one atomic two-channel write for each pH dose decision.
3. Enforce lockout windows and pulse caps.

## 5) Stage 2 BT Behavior (Machine Layer)

Per control cycle:

1. Gate on pH signal quality (`ReactiveSequence` safety gating).
2. Evaluate deadband relative to target.
3. Choose exactly one branch: acid pulse, base pulse, no-dose.
4. Apply inter-dose lockout/mix wait.
5. Emit through change/keepalive policy.

## 6) Mode-Exit Handoff (Valid Transitions Only)

Required behavior:

1. `AUTO -> MANUAL`: both dosing channels off.
2. `Any -> FAULT`: both dosing channels off.
3. `MANUAL -> IDLE`: if shutdown writes are required, run in `before_transition` hook.

Do not plan against `AUTO -> IDLE`; that transition is invalid.

## 7) Stage 2 Parameters

1. `ph_auto_enable` (bool)
2. `ph_target` (double)
3. `ph_deadband` (double)
4. `acid_pwm` (int64)
5. `base_pwm` (int64)
6. `dose_pulse_s` (int64)
7. `mix_wait_s` (int64)
8. `min_interdose_s` (int64)
9. `max_pulses_per_hour` (int64)
10. `command_keepalive_s` (int64)
11. `write_failure_limit` (int64)

## 8) Validation Sequence

1. Dry-run with `ph_auto_enable=false`.
2. Enable with conservative pulse settings.
3. Verify one-sided dosing only.
4. Verify lockout and anti-chatter behavior.
5. Force signal quality failure and verify dosing halt.
6. Verify transition-hook handoff in each required path.

Acceptance criteria:

1. No simultaneous acid/base drive.
2. Stable deadband behavior without oscillation chatter.
3. Operator can disable/recover quickly.
4. Boundary audit remains clean (core stays provider-agnostic).

## 9) Out of Scope

1. PID/MPC pH control.
2. DO-driven control logic.
3. Multi-variable optimization policies.
