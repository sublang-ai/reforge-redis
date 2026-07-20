<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- SPDX-FileCopyrightText: 2026 SubLang International <https://sublang.ai> -->

# DR-002: Room Content Manifests

## Status

Proposed

## Context

- Room branches ([DR-001](001-regen-rooms-isolation.md)) must contain only permitted content.
- Test code is not confined to `tests/`: unit tests are embedded in 45 `src/` files inside `REDIS_TEST`-gated regions, plus the harness `src/testhelp.h`.
- Root `modules/` bundles separate module products (redisbloom, redisearch, redisjson, redistimeseries, vector-sets) with their own code and tests.

## Decision

### Common

- `modules/` is out of experiment scope and removed from both room views,
  except `modules/vector-sets/` sources, which the core server binary links
  (`hnsw.o`, `vset.o`, `vset_config.o`); vector-sets' own test and example
  files (`test.py`, `fastjson_test.c`, `examples/`) are removed.
- `specs/decisions/` and `specs/iterations/` are control-plane records and removed from both room views; dangling links in room-view `map.md` are accepted residue.
- Room branches accumulate the room's own outputs (specs or Rust code, `room/` files); the manifests below govern only what control overlays from outside the room ([DR-001](001-regen-rooms-isolation.md)).
- This DR is the authoritative manifest; the snapshot scripts must match it.

### Spec room (`rooms/spec`)

Overlaid by control: the `upstream/redis-8.8.0` tree minus the table below, plus current `specs/` from `main` (minus control-plane records), plus `room/` updates.

Removed from the upstream tree:

| Removed | Reason |
| ------- | ------ |
| `tests/` | oracle suites, helpers, fixtures |
| `runtest`, `runtest-cluster`, `runtest-sentinel`, `runtest-moduleapi` | suite launchers |
| `src/testhelp.h` | embedded unit-test harness |
| `REDIS_TEST`-gated regions and `testhelp.h` includes in `src/` | embedded unit tests and harness references; stripped by script |
| `.github/`, `codecov.yml` | CI/coverage configs enumerate suites |
| `utils/gen-test-certs.sh`, `utils/req-res-log-validator.py`, `utils/req-res-validator/`, `utils/speed-regression.tcl` | test-suite tooling |
| `deps/hiredis/test.c`, `deps/hiredis/test.sh`, `deps/hiredis/.github/`, `deps/lua/test/`, `deps/tre/tests/` | bundled-library test content; hiredis's suite asserts live-server reply behavior |

- Retained residue, accepted: build files and docs may name test paths; names carry no test logic.
  The room-view build produces all product binaries, then exits non-zero on the test-module auxiliary target; accepted, no build-file surgery.
- `src/memtest.c` is retained: `--test-memory` is product behavior, not suite code.
- `deps/jemalloc/test/` is retained: allocator self-tests carry no server behavior, and jemalloc's configure requires the directory to generate its build.

### Code room (`rooms/code`): overlay allowlist

Control may overlay onto the room's latest tree only:

| Included | Content |
| -------- | ------- |
| `specs/` minus `decisions/`, `iterations/` | behavior specs, `meta.md`, `map.md`, licensing |
| `room/` | operational contract, report inbox, prompt log, journal ([DR-003](003-regen-improvement-loop.md)) |

- `deps/` is withheld (Rust implementation); scripting semantics (Lua 5.1, `cjson`, etc.) must therefore be fully specified.
- Crate policy: crates implementing RESP, Redis clients/servers, or Redis data structures are banned; general-purpose crates (async runtime, Lua binding, allocator, parsers) are allowed; control audits `Cargo.lock` every iteration.

## Consequences

- `REDIS_TEST` stripping is content surgery: performed by a deterministic script; its diff is auditable.
- Spec completeness burden rises where views withhold material (scripting, formats); gaps surface as oracle failures and route back per [DR-003](003-regen-improvement-loop.md).
- Manifest changes are visible as edits to this DR plus snapshot-script diffs.
