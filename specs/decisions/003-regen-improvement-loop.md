<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- SPDX-FileCopyrightText: 2026 SubLang International <https://sublang.ai> -->

# DR-003: Regeneration Improvement Loop

## Status

Proposed

## Context

- Objective: maximize the original black-box/e2e pass rate of the regenerated implementation by iterating on specs and code.
- The only sanctioned oracle leak is runtime failure output (assertions, server logs, protocol traces); test source and original source never enter rooms.
- Prompts are a leak channel and must be recorded where they act.

## Decision

### Subagent roles

| Role | Room | Inputs | Output |
| ---- | ---- | ------ | ------ |
| spec-writer / spec-fixer | spec | code view, inbox reports | spec packages (primarily `specs/user/`) |
| code-writer / code-fixer | code | spec view, inbox reports | Rust implementation + its own unit/e2e tests |
| triage | control | specs + failure logs only | per-failure verdicts |
| repro-extractor | control | runtime traces of failing visible tests (protocol capture, verbose logs) | repro: command sequence + expected replies, template-bound |
| oracle runner | control | scripts, no agent | suite report |

### Iteration protocol

1. Control refreshes room branches per [DR-002](002-room-content-manifests.md) and runs the audits of [DR-001](001-regen-rooms-isolation.md).
2. Spec-room sessions write or repair specs; control fast-forwards `rooms/spec`, tags `exp/<n>/specs`, and harvests `specs/` onto `main`.
3. Control overlays the harvested specs onto `rooms/code`; code-room sessions implement or fix; control fast-forwards `rooms/code`, tags `exp/<n>/code`.
4. Control composes an oracle worktree (code-room tree + `tests/`, `runtest*` overlaid from `upstream/redis-8.8.0`) and runs the visible suite ladder per [DR-004](004-oracle-harness-contract.md); report tagged `exp/<n>/report`.
5. Triage classifies every failure:
   - spec gap or misspecification → spec-room inbox
   - implementation bug against a clear spec → code-room inbox
   - oracle asserts implementation-specific behavior → waiver-list entry with justification (control; mechanics per [DR-004](004-oracle-harness-contract.md))
   - flaky/environment → quarantine and re-run
6. Repro-extractor distills repro blocks from runtime traces of failures triage marks under-actionable — never from test source, never for holdout tests.
7. Go to 1.

### Loop engineering

- Generation and evaluation stay strictly separated: rooms generate; only the mechanical oracle scores [[1]] [[4]].
- Fresh context per session: every room session starts clean; continuity lives in room files — specs, inbox, prompt log, `room/journal.md` (done / in progress / blocked) — and the room branch's git history [[2]] [[3]].
- Evaluation cascade, cheapest first [[4]]:

| Tier | What runs | Where | When |
| ---- | --------- | ----- | ---- |
| T0 | code room's own tests + inbox repro scripts | code room | continuously |
| T1 | previously failing visible test files | control | after each code-room fast-forward |
| T2 | current rung, visible set | control | once per iteration |
| T3 | all unlocked rungs, visible sets (regression guard) | control | once per iteration |

- Failure reports are routed in clusters grouped by spec package, so fixes target shared behavior rather than per-test patches [[5]].
- The code room must add its own regression test for each inbox report before fixing it, turning routed oracle knowledge into durable room-side assets.
- Sequential lineage by default; a failure cluster unresolved past the stuck threshold escalates to parallel fix attempts in fresh worktrees of the room clone, scored on T1, winner merged — diversity when iteration stalls [[4]].
- Concurrency, same room: sessions whose prompts declare disjoint file ownership may run concurrently in worktrees of the room clone on control-created branches; control merges the branches, and a sequential integrator session owns shared files (`map.md`, journal, cross-citations) and reviews the merge.
  Each concurrent session keeps its own prompt file and transcript; the transcript audit covers every worktree.
- Concurrency, across iterations: spec-room sessions for iteration n+1 may run while code-room sessions for iteration n are in flight — room inputs are disjoint and flow only through control (spec harvests forward, failure reports backward), and within each iteration the protocol's step order is preserved.
- Spec-first bias: when triage cannot decide between spec gap and implementation bug, it routes to the spec room, so oracle knowledge lands in specs explicitly rather than in code silently.

### Parameters

Defaults; IRs may tune them per rung without amending this protocol.

| Parameter | Default | Rationale |
| --------- | ------- | --------- |
| Holdout fraction | 10% of a rung's test files, fixed seed, frozen at rung unlock | enough signal to expose overfitting; 90% kept for the loop [[5]] |
| Reports per room per iteration | ≤ 12, in ≤ 4 clusters | batching amortizes sessions; clustering drives shared fixes [[5]] |
| Flaky policy | 3 runs; fails ≥ 2/3 → route; flaky across 2 iterations → quarantine, retest every 5 iterations | Redis suite is timing-sensitive |
| Stuck threshold | cluster unresolved after 2 routed iterations → escalate | |
| Parallel attempts on escalation | 2 candidates, T1-scored, winner merged | pass@k gains under a tight token budget |
| Plateau window | 3 iterations with < 1 pp net gain on the rung → escalate strategy; 2 fruitless escalations → stop, human review | |
| Sessions per room per iteration | ≤ 4, each fresh-context with journal handoff | bounds cost; avoids context rot [[2]] [[3]] |
| Rung promotion | visible ≥ 99% pass excluding waivers, then one holdout run with visible/holdout gap ≤ 2 pp (aggregate only) | gap bound guards against reward hacking [[5]] |
| Waiver budget | ≤ 2% of a rung's tests, human-approved | keeps the honesty valve small |

### Prompt recording

- A room subagent is launched only by pointing it at a prompt file already committed in its room at `room/prompts/<iter>-<seq>-<role>.md`.
- Follow-up messages to a running subagent are appended to that file before sending.
- Each prompt file records the agent configuration (model, tool set), so a room iteration is replayable from its branch alone.
- Control never inlines spec, code, or test content into prompts; room-local pointers only.

### Report content rules

- Reports cover visible tests only; holdout results appear in no report, inbox, or prompt.
- Reports contain: test identifier, behavioral expectation, expected/actual values, trace-derived repro command sequence, server-log excerpts.
- Reports never contain: test source lines, original source, references to original implementation structure.

### Objective and termination

- Suite ladder: core → integration (replication, persistence) → cluster/sentinel; rung manifests, exclusions, and harness contract per [DR-004](004-oracle-harness-contract.md).
- At rung unlock, the rung's test files split into `visible` and `holdout` manifests (fixed seed, frozen); T0–T3 and all routing use `visible` only.
- Holdout runs only at promotion attempts and at final termination; only the aggregate pass rate is recorded, and per-test holdout results are sealed from triage and rooms.
- Terminate when the in-loop ladder is green (minus waivers) and the holdout score meets target, or on pass-rate plateau across k iterations.
- On termination, the code-room tree is promoted to `main` per [DR-001](001-regen-rooms-isolation.md).

## Consequences

- Oracle knowledge migrates into specs only through failure behavior; the report archive quantifies exactly how much.
- The waiver list is the honesty valve: implementation-specific assertions are excluded visibly, never silently.
- Prompt files give each room a complete, self-contained audit trail; replay needs no control-room context.
- Spec-vs-memory provenance of code-room behavior is not instrumented; the language choice and crate ban ([DR-002](002-room-content-manifests.md)) are the only prior mitigations.
- Optimizing against a visible suite invites reward hacking; the holdout gap bound, clustered routing, and regression-test discipline are the structural guards [[5]].
- Repeated promotion attempts exert mild selection pressure on the holdout; rare, aggregate-only evaluation bounds it.

## References

[1]: https://www.anthropic.com/research/building-effective-agents "Building Effective Agents (Anthropic)"
[2]: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents "Effective Harnesses for Long-Running Agents (Anthropic)"
[3]: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents "Effective Context Engineering for AI Agents (Anthropic)"
[4]: https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/ "AlphaEvolve (Google DeepMind)"
[5]: https://arxiv.org/abs/2605.21384 "SpecBench: Measuring Reward Hacking in Long-Horizon Coding Agents"
