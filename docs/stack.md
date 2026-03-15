# FEAST Stack — Architecture Reference

## Layers

### CRUMBS

**Job:** I2C transport. Encode/decode variable-length framed messages (4–31 bytes), CRC-8,
SET_REPLY query pattern, role-based dispatch. Zero knowledge of application payloads.

Key headers: `crumbs.h`, `crumbs_message_helpers.h`, `crumbs_i2c.h`, `crumbs_linux.h`,
`crumbs_version.h`.

Invariants:
- CRUMBS has no knowledge of BREAD, RLHT, or DCMT. Protocol semantics belong entirely in the layer
  above.
- HAL abstraction via `crumbs_i2c_write_fn` / `crumbs_i2c_read_fn` fn-pointers — no Linux
  dependency in the core C library.
- Version encoded as `major*10000 + minor*100 + patch` for single-integer `>=` comparison.

---

### bread-crumbs-contracts

**Job:** BREAD application layer on top of CRUMBS. Type IDs, per-device opcodes, payload byte
layouts, capability schema and flag assignments, version helpers. Header-only C.

Key headers: `bread_ops.h`, `bread_caps.h`, `bread_version_helpers.h`, `rlht_ops.h`, `dcmt_ops.h`.

Invariants:
- Capabilities are **additive only** — new features add bits; existing bit assignments never change
  meaning. No type-ID splits for capability variations.
- `bread_check_crumbs_compat` requires `CRUMBS_VERSION >= 1100` (v0.11.0).
- The `rlht_get_state` / `dcmt_get_state` C helpers use `crumbs_device_t` fn-pointers suited for
  embedded controllers. **C++ providers must not use these helpers** — they bypass `crumbs::Session`
  and cannot be unit-tested without hardware. Use `session.query_read()` and parse frame payload
  bytes directly from the struct field docs.
- Headers include `crumbs.h`, so CRUMBS headers must be on the include path of any consumer. CMake
  consumers must add CRUMBS includes manually until targets export this dependency transitively.

---

### anolis-provider-bread

**Job:** ADPP hardware provider for BREAD devices. Owns the CRUMBS session, runs discovery and
compatibility checking, builds ADPP inventory, serves all ADPP RPCs over stdio+uint32_le framing.

Key module boundaries:
- `crumbs::Session` — C++ wrapper over the C CRUMBS API. `Transport` ABC enables `ScriptedTransport`
  for unit tests without hardware. All device adapter code uses `session.query_read()` and
  `session.send()` exclusively.
- `devices/rlht/`, `devices/dcmt/` — one adapter per device family. Each owns ADPP signal and
  function metadata, read translation (frame payload → `SignalValue`), and call dispatch (ADPP
  `CallFunctionRequest` → CRUMBS send).
- `core/health.cpp` — provider and device health from `RuntimeState`. Surfaces degraded state
  (ready but some expected devices missing) and unreachable entries for missing expected IDs.

**DCMT parse rule:** `frame.payload[0]` is the mode byte. Open-loop (`0x10`): `value1/value2`
alias `target1/target2`. Closed-loop (`0x11`): `value1/value2` are independent measured values.
Parse must branch on mode before reading velocity and position fields.

**Capability gating:** `build_rlht_capabilities(flags)` / `build_dcmt_capabilities(flags)` gate
which `FunctionSpec` and `SignalSpec` entries appear in `DescribeDevice`. This happens once at
inventory-build time. Adapters do not re-check flags at call time — the inventory is the gate.

---

### anolis-provider-sim

**Job:** ADPP provider for simulated devices. Reference implementation of the full provider
contract. Not hardware-dependent — used for development, testing, and system-level validation under
anolis.

The per-device module pattern (per-type `State` struct + `init()` + `get_capabilities()` +
`read_signals()` + `call_function()`) is the pattern provider-bread device adapters follow.

---

### anolis

**Job:** Orchestrate providers, maintain `DeviceRegistry`, poll state via `StateCache`, route
control via `CallRouter`, expose HTTP REST `/v0/...`, emit telemetry, run optional automation.

Invariants:
- All control goes through `CallRouter.execute_call()` — HTTP, BT automation, and tests use the
  same path. No provider bypass.
- `DeviceRegistry` returns by-value (`get_device_copy`, `get_all_devices`) — no pointer leaks
  across provider restarts.
- `ProviderSupervisor` handles crash detection and exponential-backoff restart. Providers are
  isolated processes with no shared memory.
- Automation (`BTRuntime`) uses only `{state_cache, call_router}` — no new protocol paths.
- Mode state machine starts in IDLE. FAULT recovery requires an explicit transition through MANUAL
  — no shortcut FAULT → AUTO.
- HTTP API is localhost-only by default (v0). No authentication. Must be addressed before any
  networked deployment.

---

## Interfaces

### ADPP (Anolis Device Provider Protocol)

Protobuf schema + uint32_le length-prefixed framing over stdio. Schema lives in `anolis-protocol`,
vendored as a submodule in each repo that uses it. Both sides must have the submodule at the same
commit.

Key coupling: `SignalSpec.poll_hint_hz > 0` in a `DescribeDevice` response causes `StateCache` to
automatically poll that signal. Provider-bread sets `poll_hint_hz = 1.0` on all signals.

`provider_id` (anolis config `id:` key) is the routing identifier anolis uses everywhere.
`provider_name` in `HelloResponse` is the provider's self-reported label — anolis does not use it
for routing.

### crumbs::Session

C++ boundary wrapping the C CRUMBS library for C++ providers. Provides `open()`, `scan()`,
`send()`, `query_read()` with configurable retry, per-session mutex, and timeout propagation.
`ScriptedTransport` implements the `Transport` ABC for deterministic unit tests.

### bread-crumbs-contracts headers

Included at compile time by `anolis-provider-bread` and any C++ consumer of BREAD wire contracts.
Because the headers include `crumbs.h`, CRUMBS headers must be on the include path of all
consumers. Until these have proper exported CMake targets, consumers add CRUMBS includes manually.

---

## Version / Compatibility Chain

```
CRUMBS_VERSION (encoded major*10000 + minor*100 + patch)
  → bread_check_crumbs_compat()       requires >= 1100
BREAD module version (e.g. RLHT v1.0)
  → bread_check_module_compat()       major must match; minor >= required minimum
probe_device()
  → ProbeStatus: Supported | IncompatibleCrumbsVersion | IncompatibleModuleMajor/Minor
build_inventory_from_probes()
  → excludes incompatible devices with diagnostics; continues with compatible set
ADPP DescribeDevice
  → anolis DeviceRegistry (immutable post-discovery)
StateCache / CallRouter
```

When `GET_CAPS` fails during discovery, `make_baseline_capability_profile()` is used — all
currently supported capability flags are set. This is the caps-unavailable fallback policy.

---

## Dependency Rules

```
anolis-provider-bread
  → anolis-protocol   (ADPP proto, git submodule)
  → CRUMBS            (sibling repo, source-ingested; Linux HAL gated by ENABLE_HARDWARE)
  → bread-crumbs-contracts  (sibling repo, headers only)
  → linux-wire        (sibling repo, via CRUMBS Linux HAL, hardware builds only)
  → vcpkg: protobuf, yaml-cpp, gtest

anolis-provider-sim
  → anolis-protocol   (same submodule pattern)
  → vcpkg: protobuf, yaml-cpp, gtest, BehaviorTree.CPP (optional), cpp-httplib

anolis
  → anolis-protocol   (submodule)
  → vcpkg: protobuf, yaml-cpp, gtest, cpp-httplib, nlohmann-json, BehaviorTree.CPP (opt), openssl

CRUMBS
  → linux-wire        (sibling repo, Linux HAL only)

bread-crumbs-contracts
  → CRUMBS headers    (not a full build dep; included for inline helpers)
```

Dependency direction is strictly downward — no lower layer has knowledge of any layer above it.
No circular dependencies. No layer reaching past its immediate lower neighbor.

---

## Design Principles

| Principle | Rule |
|---|---|
| Transport / protocol separation | CRUMBS = transport only. BREAD semantics start at bread-crumbs-contracts. |
| Additive capabilities | Cap flags only gain bits. No type-ID splits for capability variations. |
| Capability gating at inventory time | Adapters do not re-check caps on every call. The DescribeDevice inventory is the gate. |
| C++ providers use `crumbs::Session` | Never call CRUMBS or contracts C helpers directly from adapter code. |
| Baseline fallback on caps query failure | Use `make_baseline_capability_profile()`. Document the risk for production use. |
| Provider isolation | Crash in a provider does not crash anolis. ProviderSupervisor restarts with backoff. |
| Single control path | All calls go through `CallRouter`. Automation is not a bypass. |
| Safe by default | Provider-bread: exclude incompatible hardware, continue with rest. anolis: start in IDLE. |
| No early abstraction | Generic multi-family CRUMBS provider is speculative. Extract only when a second real consumer exists. |
