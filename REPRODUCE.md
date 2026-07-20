<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- SPDX-FileCopyrightText: 2026 SubLang International <https://sublang.ai> -->

# Requirements: clean-room regeneration experiment

This document states the **requirements** for running the experiment. It is
deliberately free of results, findings, or history from any prior run — the
run must be independent. The only inherited material is the decision records
in `specs/decisions/` (DR-000..004), which define protocol, not outcomes.

Read this document and the DRs before starting.

---

## 1. The experiment

**Goal.** Regenerate a Redis-compatible server in Rust, clean-room, driven by
written specifications rather than by copying source — then measure the
result against the *real* Redis test suite and improve it in a closed loop
until the pass rate plateaus or the suite is green.

**The claim under test.** That a specification, written by agents that may
read the reference implementation, is a sufficient channel to carry behavior
to implementer agents that may *never* read it. The specs are the product;
the code is downstream of the specs; the test suite is an oracle that neither
room may read.

**Clean-room boundary — this is the experiment. If it leaks, the result is
void.**

- The **spec room** may read the reference implementation source. It writes
  only specs (`specs/`).
- The **code room** may read `specs/` and its own prior code. It may
  **never** read the reference implementation source or the test suite.
- The **oracle** (the Redis `tests/` tree + `runtest`) is executed only by the
  control plane and is never mounted into a room.
- The **only** channel from oracle back into a room is an
  orchestrator-authored *report* carrying trace-derived expected/actual
  evidence. Never paste test source or reference source into a room.

Note the boundary is rooms ↔ (oracle, reference). The control plane may read
everything; that is not a leak.

**Baseline.** This repository starts as a pristine import of the upstream
Redis release, tagged `upstream/redis-8.8.0`. Do not modify the baseline
tree; the regenerated implementation is built alongside it in the code room.

---

## 2. Starting state and what you must build

At iteration 0 this repository contains only:

- the pristine upstream Redis tree (baseline + oracle source),
- `specs/decisions/` — DR-000..004,
- this document.

Everything else is yours to create, in this order:

1. `specs/meta.md` — the spec-of-specs (item syntax, package rules, citation
   rules), authored under DR-000.
2. `specs/map.md` — the package index plus the DR and IR tables.
3. `control/` — the loop harness (§7), plus `control/README.md` documenting
   the operational conventions not fixed by the DRs.
4. `control/rungs/` — the scoring manifests (§8). **Freeze these once**,
   before the first score, and never re-split them; a mid-run re-split
   destroys score comparability.
5. The room clones and their branches (§3).

Do not carry over specs, code, reports, or triage from any prior run. If a
prior run's harness scripts are reused, treat them as tooling only and still
freeze a fresh rung manifest before the first score.

---

## 3. Topology

One control repo (this one) plus **room clones** outside it at
`$REFORGE_ROOMS_DIR` (default: sibling `../reforge-rooms/`).

```
reforge-redis/                    control repo (orchestrator works here)
../reforge-rooms/spec/            clone on rooms/spec
../reforge-rooms/code/            clone on rooms/code
../reforge-rooms/oracle-exp<N>/   composed suite tree (built binary + tests)
../reforge-rooms/build-exp<N>/    isolated, tests-free build dir
```

| Branch | Contents |
| --- | --- |
| `main` | Specs + product. Spec harvests land here. No reports. |
| `rooms/spec` | Spec room's view + its committed spec work |
| `rooms/code` | Code room's Rust workspace + `room/` overlay (prompts, inbox, journal) |
| `control/reports` | Oracle reports, triage, session transcripts, manifests, staging |

Room branches must have **orphan lineage** (DR-001) — composed by the
snapshot scripts, never merged from `main`.

Tag every iteration: `exp/<N>/specs`, `exp/<N>/code`, `exp/<N>/report`.

---

## 4. The binding decision records

Read these; cite them by ID in everything you author.

| DR | File | Binds |
| --- | --- | --- |
| DR-000 | `specs/decisions/000-spec-structure-format.md` | Spec structure, format, naming |
| DR-001 | `specs/decisions/001-regen-rooms-isolation.md` | Rooms, repo layout, orphan room branches, isolation audits |
| DR-002 | `specs/decisions/002-room-content-manifests.md` | Exactly which paths are overlaid into each room |
| DR-003 | `specs/decisions/003-regen-improvement-loop.md` | Iteration protocol, session caps, concurrency, escalation, plateau rule, stop gate |
| DR-004 | `specs/decisions/004-oracle-harness-contract.md` | Execution mode, rung manifests, waivers, smoke gate |

Load-bearing parameters from DR-003, restated so they are not missed:

- **≤3 spec sessions and ≤4 code sessions per iteration.**
- **Sequential lineage within a room** (one session at a time per room);
  spec-room and code-room work of *different* iterations may pipeline.
- **Escalation = 2 parallel candidates** (tight-token-budget constraint).
- **Plateau rule:** no score movement across k iterations → declare a plateau
  and change strategy rather than iterating blindly.
- **Stop gate:** two consecutive *fruitless* escalations → **halt the loop and
  obtain human review before continuing.**

---

## 5. The iteration cycle

Each iteration `exp/<N>`:

1. **Mine.** Fan out parallel readers over the previous iteration's per-file
   oracle logs. Extract every `[err]`/`[exception]` with *verbatim*
   expected/actual evidence. Cluster by shared root cause.
2. **Plan.** Merge cross-file clusters that share a cause. Rank by expected
   yield. Split into ≤3 spec batches and ≤4 code sessions. **Every spec batch
   must have a same-iteration code route** (§9.1).
3. **Author reports** — one inbox report per session, format per §6.
4. **Snapshot the spec room**, run spec sessions sequentially, close each out
   per §8.
5. **Harvest specs to `main`**, commit, tag `exp/<N>/specs`, push.
6. **Snapshot the code room** with the *harvested* specs plus the code
   reports. Run code sessions sequentially. Tag `exp/<N>/code`, push.
7. **Score** per §8.
8. **Triage** → `triage/exp<N>.md` on `control/reports`; author `IR-<N>` on
   `main` with its `map.md` row; push everything.
9. Stage the next iteration immediately; do not idle between iterations.

---

## 6. Report format — the only channel into a room

Every numbered item **must** carry:

- **Expected** — verbatim quoted oracle evidence (assertion text, error
  replies, exception text). Never paraphrase; never invent.
- **Observed** — verbatim actual.
- **Suite context** — *mandatory*: every encoding-loop leg and every config
  mutation in force at that point in the test file. Without this, a room
  cannot reproduce the failure and will return a false "not reproducible."
- **Affected** — spec files and item IDs (spec reports), or subsystem (code
  reports).
- **Repair / Implement** — what to do, including the regression-test
  requirement.

Rules:

- Spec reports must state their same-iteration code route.
- Code reports cite **spec item IDs as the contract** and must never quote or
  describe reference-implementation source.
- Every report ends with a Definition-of-done line.
- The session prompt must instruct the room to work **ALL** items in the
  report; a prompt narrower than its report causes silently skipped items.

---

## 7. Running room sessions

Sessions are **headless** `claude -p`, launched through a wrapper, always as a
**tracked background task** — never a bare shell `&`, which dies with the
orchestrator's shell and loses the completion signal.

Wrapper requirements:

- Hardcode the room model (§10) so it cannot drift.
- Scope tools explicitly. `--dangerously-skip-permissions` is blocked by the
  auto-mode classifier; a scoped `--allowedTools` list is the sanctioned
  replacement. Code rooms need git/cargo/rustc/build-run; spec rooms need
  git and read/move tools only.
- Disallow `WebFetch`, `WebSearch`, and sub-agent/workflow tools in rooms.
- Emit `--output-format stream-json --verbose` to a per-session transcript
  file, so the transcript can be audited and archived.
- Support a `continue` mode that instructs the session to read the
  **Follow-up** section appended to its prompt file.

The harness (`control/`) must provide, per DR-001/002/004:

- `snapshot_spec_room.sh` / `snapshot_code_room.sh` — compose each room view
  from the DR-002 manifest plus a control-staged `room/` overlay.
- `run_oracle.sh` — isolated build, smoke gate, rung run, report commit (§8).
- `audit_transcripts.py` — assert every room tool call stayed in-room.
- `audit_room.sh` — blob/symlink/alternates audit plus crate-ban check.
- `audit_fingerprint.py` — detect verbatim runs from original source in specs.
- `make_rung_manifest.py` — one-time visible/holdout split.

---

## 8. Session close-out (after *every* session)

1. **Assert the model.** The transcript's model records must show only the
   mandated room model (§10). Anything else → **stop, report, void the
   session's work.**
2. **Read the `result` record** (last line of the transcript): `is_error`,
   `num_turns`, `total_cost_usd`, `duration_ms`, `duration_api_ms`.
3. **Audit isolation** with the three audit scripts. Triage every flag. A
   code-room read of the test suite or reference source is a **breach** and
   voids the session; classify anything else explicitly before accepting it.
4. **Archive the transcript** to `control/reports:sessions/exp<N>/` and append
   a row to a session manifest (model, turns, tokens, cost, result, notes).
5. **Fast-forward** the control-side room branch **from the control repo
   root** (§9.9), verify the ref moved, then push.
6. If the session was interrupted, append a **Follow-up** section to its
   prompt file naming exactly what is committed versus uncommitted, then
   relaunch in `continue` mode.

---

## 9. Operating requirements

These are mandatory operating rules for the loop and its harness. They
constrain *how* the machinery is run and how evidence is judged; they say
nothing about the product under regeneration.

**9.1 Every spec batch needs a same-iteration code route.** Spec output
changes only text; the oracle scores the binary. A spec repair with no code
session is invisible to the score and silently re-queues its cluster. At
triage, diff the iteration's spec-item IDs against the code sessions' commit
messages — every item must appear in a code commit or in the IR's explicit
defer list.

**9.2 Solo-rerun before triaging.** Concurrent per-file lanes fabricate
failures under resource contention (spurious I/O truncations and false
hangs). After the parallel pass, solo-rerun every file that dips below its
previous-map baseline and every timed-out file. **Solo results are
authoritative.** Reruns cost wall-clock, not tokens.

**9.3 Suspect cascades before positing many bugs.** A test that mutates
config or debug state and fails mid-body skips its own reset, poisoning every
later test in that file. When a cluster's failures all sit downstream of one
such failure, route the **upstream** failure only.

**9.4 Probe first; never relaunch blind.** Rooms lack the test harness and
cannot self-verify oracle divergences. When the oracle contradicts a room's
verdict, the orchestrator must reproduce it directly against the room's own
binary (wire-level probes) and relaunch with per-item pinned evidence.

**9.5 Foreground only.** A headless session that ends its turn awaiting a
background build or test is terminated, and the work is lost. Forbid
background waits in every prompt. Require **commit-early** (per item, as its
tests pass) so an interruption costs minutes, not hours.

**9.6 Expect transient drops.** Long streaming sessions drop mid-response
routinely. Retry after ~1 minute; reserve long waits for genuine usage-limit
errors. Record every attempt in the session manifest.

**9.7 Verify claims control-side.** Re-verify evidence against the current
tree at routing time, and re-run the actual oracle files against a session's
own binaries before accepting any verdict — especially "not reproducible."

**9.8 Let the spec defend itself.** Reports are hypotheses, not truth. A room
that finds the spec text contradicts its report must trust the spec and
journal the inversion rather than implement the report.

**9.9 Never `cd` into a room for control-plane git.** A compound
`cd <room> && git …` silently retargets control plumbing at the room clone,
which can leave control refs unadvanced and cause a stale build to be scored.
Run all control git from the control repo root with `git -C`/explicit paths;
keep room launches (which do need `cd`) in separate calls. If the oracle
script moves the `control/reports` ref, remove any worktree checkout of that
branch before invoking it.

**9.10 Treat the temp scratchpad as disposable.** OS cleanup can wipe it
mid-iteration. Archive per-file summaries, captures, and mining output to
`control/reports` **at creation time**; keep launch scripts regenerable;
re-check state after any gap.

**9.11 Push after every session close** — transcripts, manifest rows, room
branches, and tags — not just at iteration close.

**9.12 Keep temp files inside the room tree** (e.g. `target/tmp/`), never the
system temp dir or the harness scratchpad. Enforce via the transcript audit.

**9.13 Sweep orphaned test servers** between iterations; daemonized servers
left by test runs confound later reproductions.

---

## 10. Model requirements

- **Orchestrator / control plane:** the strongest available reasoning model,
  with parallel sub-agent orchestration for mining and report drafting.
- **All room sessions:** a single fixed model at default effort, for the
  entire run, regardless of usage state. Enforce mechanically — the launch
  wrapper hardcodes it and every close-out asserts it from the transcript.
- **A mixed-model artifact is not a valid experiment.** If any session runs on
  the wrong model, void and redo that session's work.
- Operate under a tight token budget: keep escalations at 2 candidates and do
  not add majority-vote or N-sample verification passes without approval.
  Prefer single well-scoped sessions and lean on the mechanical oracle, which
  costs wall-clock rather than tokens.

---

## 11. Scoring

Define a **rung**: a frozen manifest of test files drawn from the upstream
suite, split once into a visible list and a sealed holdout. The holdout is
**never** copied into rooms, prompts, or reports, and is evaluated rarely and
aggregate-only (DR-003/DR-004).

Two complementary runs per iteration:

1. **Canonical** (`run_oracle.sh <N>`): export the code room into a
   *tests-free* build directory (this closes the build-time channel), compose
   a suite tree, run a **smoke gate** — the server must reach readiness and
   answer a ping; if it does not, the rung is not executed and every file is
   reported blocked-at-startup — then run the suite once over the manifest.
   Commit a report on `control/reports` and tag `exp/<N>/report`. A single
   suite invocation aborts at the first exception, so treat this as a **gate,
   not the score**.
2. **Per-file map** (authoritative): run each manifest file singly, capped by
   an alarm timeout, optionally in parallel lanes with **distinct base
   ports**, then apply §9.2. Sum `[ok]`/`[err]`/`[exception]` per file; a
   timeout means a hang.

**Interpreting the score.** The denominator is only the *executed*
assertions — a file truncated by a crash hides its remaining tests. Clearing
a blocker therefore raises both ok and err counts. Rising `err` alongside
rising `ok` is progress (dead territory converted into readable failures),
not regression. Track `ok` as the headline figure and fully-green file count
as the quality figure. Record the full per-file map every iteration.

---

## 12. Definition of done

- No timed-out (hanging) files in the per-file map.
- Score plateaus across k consecutive iterations (DR-003), **or** every rung
  file is green.
- Every spec item has a code route or an explicit, dated defer.
- All session transcripts archived and pushed; every session model-asserted.
- A final IR recording the score trajectory, the plateau and escalation
  history, and the waiver list — each waiver individually human-approved,
  with the bar being "provably unrunnable without original-implementation
  internals."
