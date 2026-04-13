# Packaging Boundary Decision Record

Status: Deferred decision.
Priority: Later (after contract hardening).
Intent: Decide whether `operator-ui` and `system-composer` stay in `anolis` or split into one or more repos.

## 1) Why This Is Deferred

A repo split before contract stabilization will likely move drift and coordination overhead into more places.

## 2) Decision Timing

Revisit only after:

1. Runtime config schema is CI-enforced.
2. OpenAPI contract is CI-enforced.
3. UI/composer fixture alignment is stable.
4. At least one machine package is fully validated end-to-end.

## 3) Options to Evaluate

1. Keep both tools in `anolis`.
2. Split `operator-ui` into its own repo, keep composer in `anolis`.
3. Split `system-composer` into its own repo, keep UI in `anolis`.
4. Merge UI + composer into one dedicated app repo.
5. Split both into separate repos.

## 4) Evaluation Criteria

1. Contract stability and release cadence.
2. Cross-repo coordination burden.
3. CI/runtime integration complexity.
4. Contributor workflow clarity.
5. Versioning and support burden.

## 5) Evidence Required Before Decision

1. At least two release cycles with stable config + HTTP contracts.
2. Measured churn rate in UI/composer vs runtime.
3. Clear ownership model per component.

## 6) Decision Output Template

When ready, record:

1. Chosen boundary model.
2. Migration plan (if split).
3. Versioning and compatibility policy.
4. CI/release changes required.

