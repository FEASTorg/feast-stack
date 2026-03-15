# feast-stack

FEAST is a multi-repo stack for real-time embedded hardware automation. It connects embedded BREAD
hardware through a layered C/C++ stack to a runtime that exposes state and control over HTTP and
telemetry.

## Repos

| Repo | Role |
|---|---|
| [CRUMBS](https://github.com/FEASTorg/CRUMBS) | I2C framing transport (C, embedded-safe) |
| [bread-crumbs-contracts](https://github.com/FEASTorg/bread-crumbs-contracts) | BREAD application layer: type IDs, opcodes, payload layouts, caps (header-only C) |
| [anolis-provider-bread](https://github.com/FEASTorg/anolis-provider-bread) | ADPP hardware provider for BREAD devices over CRUMBS |
| [anolis-provider-sim](https://github.com/FEASTorg/anolis-provider-sim) | ADPP provider for simulated devices (reference implementation) |
| [anolis](https://github.com/FEASTorg/anolis) | Runtime: orchestrates providers, exposes HTTP API, drives automation |
| [anolis-protocol](https://github.com/FEASTorg/anolis-protocol) | ADPP protobuf schema (vendored as submodule in each provider and in anolis) |

## Stack

```
External consumers  (HTTP REST /v0/...)
        │
anolis  (C++17 — ProviderRegistry, DeviceRegistry, StateCache, CallRouter, HttpServer)
        │  ADPP  (protobuf + uint32_le framing, stdio)
        ├── anolis-provider-sim     (simulated devices)
        └── anolis-provider-bread   (real BREAD hardware)
                │  crumbs::Session  (C++ wrapper — retry, mutex, query_read)
                CRUMBS  (C — I2C framing, CRC-8, Linux HAL)
                │  /dev/i2cN  (linux-wire)
                BREAD hardware  (RLHT / DCMT MCUs)

Shared at compile time:
  bread-crumbs-contracts  →  type IDs, opcodes, payload layouts, caps helpers, version helpers
```

See [docs/stack.md](docs/stack.md) for layer contracts, interface rules, and design principles.

## Dev Setup

For a new Linux machine:

```sh
sudo apt update && sudo apt install -y \
  build-essential cmake ninja-build pkg-config git curl zip unzip tar

git clone https://github.com/microsoft/vcpkg ~/vcpkg
~/vcpkg/bootstrap-vcpkg.sh
echo 'export VCPKG_ROOT=$HOME/vcpkg' >> ~/.bashrc
source ~/.bashrc
```

The expected local workspace layout:

```text
repos_feast/
├── anolis/
├── anolis-provider-bread/
├── anolis-provider-sim/
├── CRUMBS/
├── bread-crumbs-contracts/
└── linux-wire/              # required for provider-bread hardware builds
```

Each repo has its own `docs/build.md` with configure, build, and test commands.

