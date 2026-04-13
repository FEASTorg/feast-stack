# Machine Profile Contract and Packaging Plan

Status: Partially defined, pending implementation decisions.
Priority: Medium (after runtime config + HTTP contracts).
Intent: Package each machine setup as a clear, versioned, runnable unit without leaking machine specifics into core runtime code.

## 1) Problem

Machine assets currently span config, provider maps, behavior trees, and runbooks across multiple locations. Packaging boundaries are implicit.

## 2) Desired End State

A machine profile package should contain:

1. Runtime profile YAML(s).
2. Provider profile YAML(s).
3. Behavior tree(s), if automation-enabled.
4. Optional telemetry/export profile.
5. Validation/runbook commands.
6. Profile manifest with version and compatibility metadata.

## 3) Candidate Structure (Proposed)

1. `anolis/config/machines/<machine_id>/`
2. Required manifest file (`machine-profile.yaml` or similar).
3. Stable entrypoint profile names:
   - `runtime.manual.yaml`
   - `runtime.telemetry.yaml`
   - `runtime.automation.yaml`
   - `runtime.full.yaml`

## 4) Key Contract Rules

1. Core runtime remains provider-agnostic.
2. Provider/device/channel mapping belongs only in machine package config.
3. Behavior trees reference machine-specific IDs through profile parameters, not core code constants.
4. Validation scripts must run from machine package entrypoints.

## 5) Decision Points (Open)

1. Package location:
   - Keep under `anolis/config/` or promote to separate machine repo later.
2. Manifest format:
   - YAML vs JSON; required metadata fields.
3. Compatibility model:
   - pin exact runtime/provider versions vs semver ranges.
4. Artifact ownership:
   - which repo owns long-term machine runbooks and evidence templates.

## 6) Minimal First Slice

1. Formalize current bioreactor profile as first machine package baseline.
2. Add manifest with:
   - machine ID,
   - profile file list,
   - required providers,
   - supported runtime version range.
3. Add a package validator script (presence + schema checks).

## 7) Exit Criteria

Done for this wave when:

1. Bioreactor package passes validator.
2. Package can be launched and validated via one documented command path.
3. Package schema + manifest fields are stable enough for second machine reuse.

