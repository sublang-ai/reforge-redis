<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- SPDX-FileCopyrightText: 2026 SubLang International <https://sublang.ai> -->

# DR-001: Regeneration Rooms and Isolation

## Status

Proposed

## Context

- Experiment: derive behavior specs from Redis core code without seeing test code; regenerate an implementation from specs without seeing original code; judge only by the original black-box/e2e suites; loop until best pass rate.
- Leak channels: shared git object stores (any branch/worktree exposes all objects via `git cat-file`), agent conversation context, harness-authored prompts, network access, model training priors.
- Priors over Redis cannot be removed; Rust is chosen partly to reduce verbatim structural recall, and residual behavioral priors are an accepted limitation.

## Decision

### Rooms

| Room | Role | Sees | Never sees |
| ---- | ---- | ---- | ---------- |
| Control (this repo) | harness, oracle runs, triage, audits | everything | — |
| Spec room | write/repair specs | code view without test code | test code |
| Code room | implement from specs, write own tests | spec view | original code, test code |

### Repository layout

- This repo is the experiment's long-term home.
- `main` holds the product: specs and, once promoted, the regenerated implementation with its own tests.
Room process artifacts (`room/`) never land on `main`.
- One lifecycle branch per room: `rooms/spec` and `rooms/code`, holding that room's full history — input snapshots, prompt files, inbox reports, and the room's own outputs — so any iteration is replayable by checking out its commit and re-running the recorded prompt ([DR-003](003-regen-improvement-loop.md)).
- The original import stays reachable via ref `upstream/redis-8.8.0`; code views and oracle overlays derive from it, independent of `main`'s content.
- Promotion: when [DR-003](003-regen-improvement-loop.md) termination criteria are met, control snapshots the code-room tree (minus `room/`) onto `main`, recording the source tag; original code is then retired from `main`'s tree.

### Isolation by construction

- Everything reachable from a room branch is content permitted for that room ([DR-002](002-room-content-manifests.md)); control only ever appends permitted snapshots.
- Each room is a separate clone that checks out only its branch's latest snapshot, so the room's object store never contains disallowed objects; deepening within the room's own branch is safe by the ancestry rule.
- Clone shape is part of the guarantee [[1]]: `file://` URL only (a path-local clone hardlinks or copies the control object store), with `--single-branch --depth 1 --no-tags`; `--local`, `--shared`, `--reference`, and alternates are forbidden.
- Worktrees and extra branches are permitted inside the control repo and inside a room's own clone; never across rooms.
- Room work flows back by control fetching from the room clone and fast-forwarding the room branch, tagged per iteration; flow into control is unrestricted.
- Room agents are fresh-context subagents (or headless sessions) with cwd inside their room; prompts reference room-local paths only ([DR-003](003-regen-improvement-loop.md)).
- Code room runs with network disabled to prevent fetching Redis sources.

### Audits (mechanical, every iteration)

- Blob audit: forbidden blob SHAs (test blobs for the spec room; all original code blobs for the code room) intersected with the room's `git cat-file --batch-all-objects` must be empty; `--batch-all-objects` also surfaces alternate-store objects [[2]], and the audit asserts `.git/objects/info/alternates` is absent.
- Spec-room forbidden test blobs include the unstripped originals of `src/` files whose `REDIS_TEST` regions are stripped per [DR-002](002-room-content-manifests.md); stripping yields new blob SHAs, so the audit list must enumerate the originals explicitly.
- Symlink audit: room trees must contain no symlink entries (mode 120000); a symlink can point outside the room and defeats the blob and transcript audits.
- Transcript audit: room subagent tool calls must not touch paths outside the room, by absolute path, `../` traversal, or home-directory reference; a violation voids the iteration.
- Fingerprint audit: harvested specs flagged on verbatim runs copied from original source.

### Implementation language

- The code room implements in Rust.
- Rationale, as assessed 2026-07: strong model fluency (less implementation noise unrelated to spec quality); no known public Rust Redis clone near oracle depth; Lua 5.1 available via bindings; no GC, keeping memory accounting and eviction observable.
- Redis-related crates are banned ([DR-002](002-room-content-manifests.md)).
- The module-API suite (`runtest-moduleapi`, C ABI) is out of experiment scope.

## Consequences

- Every room input and output is a tagged commit in this repo; the experiment is replayable and auditable.
- `main` stays a clean product history (specs, then regenerated code); process history lives on room branches.
- Replay fidelity covers inputs exactly; agent outputs may vary across model versions, so prompt files record the agent configuration ([DR-003](003-regen-improvement-loop.md)).
- Residual behavioral priors are accepted, not measured; results claim oracle conformance, not prior-free derivation.
- Control bears snapshot/audit scripting overhead.
- Oracle composition can use a worktree of this repo, since control has no read restrictions.

## References

[1]: https://git-scm.com/docs/git-clone "git-clone Documentation"
[2]: https://git-scm.com/docs/git-cat-file "git-cat-file Documentation"
