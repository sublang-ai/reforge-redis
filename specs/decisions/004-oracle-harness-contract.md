<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- SPDX-FileCopyrightText: 2026 SubLang International <https://sublang.ai> -->

# DR-004: Oracle Harness Contract

## Status

Proposed

## Context

- [DR-003](003-regen-improvement-loop.md) runs the suite ladder in an oracle worktree but leaves the harness mechanics undecided.
- The Tcl harness is not purely wire-level: it spawns `src/redis-server <config>`, regexp-matches server log lines for lifecycle detection, and asserts on `DEBUG`, `OBJECT`, `INFO`, and digest introspection.
- Plain `./runtest` globs `unit/moduleapi` and `unit/cluster` into its default run list, so launcher names alone do not define rungs.
- Some implementation-specific assertions sit inline within otherwise-valid test files; file-level waivers would discard valid coverage.

## Decision

This DR is the authoritative harness contract; the oracle runner scripts must match it.

### Execution mode

- Oracle runs use the spawning harness, never external-server mode (`--host`): integration, sentinel, and cluster rungs require process lifecycle control.
- Control builds the code-room tree and places the produced binaries at `src/redis-server`, `src/redis-sentinel`, `src/redis-cli`, `src/redis-benchmark`, and `src/redis-check-aof` in the oracle worktree; `redis-check-rdb` is unexercised by the suite and out of contract.

### Harness-load-bearing surface

Everything below is in-scope product behavior: failures route to rooms as ordinary [DR-003](003-regen-improvement-loop.md) reports; none of it is waiver material.

| Surface | Harness use |
| ------- | ----------- |
| `src/redis-server <config-file> [--option value ...]` | spawn, restart, config overrides |
| `src/redis-sentinel <config-file>`; `src/redis-check-aof <options> <aof-or-manifest>` | sentinel instance spawn; AOF validation/repair assertions |
| `src/redis-cli` invocations, including `-c` and `--cluster` manager operations | CLI behavior tests, cluster create/add-node/check/reshard, output-format assertions |
| `src/redis-benchmark <workload options>` | benchmark behavior tests (exit status, output) and background load generation in other tests |
| Stdout log lines: `PID: <pid>`, `Server initialized`, `Ready to accept`, `Failed listening on port` | readiness, restart, and port-conflict detection |
| `CONFIG GET`/`CONFIG SET`, `INFO` fields, `DEBUG` subcommands exercised by the suite | assertions and wait conditions |
| `OBJECT ENCODING` names and config-driven conversion thresholds | encoding assertions; client-observable behavior |
| `DEBUG DIGEST`, `DEBUG DIGEST-VALUE` | reload/replica consistency checks; self-consistency within one build suffices, parity with original digests is not required |

- The suite runs stock: `--ignore-encoding` and `--ignore-digest` stay off.

### Rung manifests

- A rung is defined by an explicit test-set manifest below; launcher defaults never define scope.
- `./runtest` realizes manifests as `--single`/`--skipunit` file lists; `tests/instances.tcl` launchers (`./runtest-cluster`, `./runtest-sentinel`) support only `--single` substring patterns, so their manifests are `--single` lists with each pattern matching exactly one test file.
- Visible/holdout splits ([DR-003](003-regen-improvement-loop.md)) are realized the same way; holdout files are never named in skipfiles, manifests handed to rooms, reports, or prompts.

| Rung | Test set | Launcher |
| ---- | -------- | -------- |
| R1 core | `tests/unit/*.tcl`, `tests/unit/type/*.tcl` | `./runtest` |
| R2 integration | `tests/integration/*.tcl` | `./runtest` |
| R3 cluster/sentinel | `tests/unit/cluster/*.tcl`; `tests/cluster/`; `tests/sentinel/` | `./runtest`; `./runtest-cluster`; `./runtest-sentinel` |
| Excluded | `tests/unit/moduleapi/` ([DR-001](001-regen-rooms-isolation.md)); `tests/vectorset/` (module scope, [DR-002](002-room-content-manifests.md)) | — |

### Waivers

- On `./runtest` rungs, the waiver list is a skipfile committed in control (one test name or `/regexp` per line), passed via `--skipfile`; granularity is the individual test block, never the file.
- `tests/instances.tcl` launchers have no skipfile support; cluster/sentinel waivers are file-level omissions from the `--single` manifest, recorded in the waiver list with the same justification requirement.
- The [DR-003](003-regen-improvement-loop.md) waiver budget counts skipped test blocks against the rung's total; a file-level waiver counts every test block in the file.

## Consequences

- Startup, logging, config, and introspection surface becomes explicit spec workload; early iterations may fail wholesale on readiness detection until specced — expected, and routed as ordinary reports.
- Test-block-level skipfiles preserve valid coverage inside partially waived files on `./runtest` rungs; cluster/sentinel waivers pay file granularity, and the per-block budget accounting prices that in.
- Digest self-consistency keeps reload/replica checks meaningful without mandating the original hash internals.
- Rung scope changes are edits to this DR, auditable alongside oracle-runner script diffs.
