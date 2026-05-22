---
Status: DRAFT
Date: 2026-05-20
Decider: Sebastian Puig (pending review)
Author: T1 (overnight elevation-push, split 20260520-2244)
Affects: IFleet Phase 4+ fold-in of the standalone audit + Codex elevation system
Extends: None
Supersedes: None
---

# IFleet integration ADRs — fold-in of the standalone elevation system

**Status:** DRAFT (5 ADRs, awaiting Sebastian review)
**Author:** T1 on split `20260520-2244-elevation-push`
**Scope:** Architectural decisions for *eventually* folding the standalone audit + Codex + lane-scheduler + Proposer system into IFleet. No IFleet code changes proposed in this document — ADRs only.

## Why these ADRs exist now

The standalone elevation system (`/audit-scan`, `/audit-fix`, `codex-review` skill, `audit-autopilot`, the self-healing pipeline T3 builds, the lane-scheduler T4 builds, the audit-broadcast + Proposer specs T5 builds) is designed to live outside IFleet for a 2-week soak (per `~/dev/ai-products/audit-elevation/plan.md` Phase 3). The *next* plan after soak is the IFleet fold-in, and Sebastian wants the architectural decisions written down **before** that plan starts so M2-M5 implementation doesn't relitigate them.

ADRs follow the IFleet style (`docs/adr/0001-0003.md`): Status, Context, Decision, Alternatives, Consequences, Reversibility, References. Each is sized for one focused fold-in PR.

---

## ADR-001 — `codex-review` skill → IFleet `src/workers/codex.ts` wiring

**Status:** DRAFT
**Affects:** M2 (plan-reviewer slot) and the existing `src/workers/codex.ts` `WorkerAdapter`

### Context

`~/.claude/skills/codex-review/SKILL.md` exists and works today as a manual `/codex-review <PR#>` invocation that shells out to `codex exec` with a structured review prompt. Separately, `~/dev/ai-products/IFleet/src/workers/codex.ts` is a 201-line `WorkerAdapter` (binary='codex', sandbox='workspace-write') that streams Codex events and never runs in production — Codex CLI isn't on the PATH yet on the VPS. Two implementations of the same provider, neither connected.

The standalone skill earns its keep on **audit-fix PRs** (cites `AUDIT-*` finding ID → Codex re-checks the exact file globs). The IFleet worker is generic — it can run *any* role (architect, editor, reviewer) using Codex as the provider. After soak, both must converge: the IFleet pipeline needs the focused per-PR review behavior, but expressed through the existing `WorkerAdapter` contract so the cross-provider invariant (PR #67/#68) still holds.

### Decision

**Fold the standalone `codex-review` skill into IFleet as a new `src/pipeline/cross-provider-reviewer.ts` that calls the existing `codex.ts` `WorkerAdapter`. The standalone skill stays as a fallback for non-IFleet repos but stops being the production review path for IFleet PRs.**

Concretely:

1. `cross-provider-reviewer.ts` is a thin coordinator: it builds the per-PR brief (same shape as `~/.claude/skills/codex-review/review-prompt.md`), calls `workerPool.spawn({ provider: 'codex', ... })`, parses the PASS/FAIL/NEEDS_REVISION verdict, and emits a `diff-reviewer:codex-verdict` trace event.
2. The audit-fix detection logic (`/AUDIT-[A-Za-z0-9_-]+/` regex on PR title/body) moves into `cross-provider-reviewer.ts`. When matched, the reviewer loads the finding from the repo's `.audits/index.json` and includes it in the Codex brief — same behavior the standalone skill ships today.
3. The standalone skill stays at `~/.claude/skills/codex-review/SKILL.md` for use against repos IFleet doesn't manage. Both implementations share the same prompt template via a symlink or copy (operational decision at fold-in time — see open question).

### Alternatives considered

1. **Keep both implementations indefinitely.** Rejected — two prompt templates will drift; one will silently regress the other. Bad.
2. **Delete the standalone skill, only use IFleet.** Rejected — Sebastian uses `/codex-review` against `~/.claude/` itself and other non-IFleet repos. The standalone path stays.
3. **Wrap the standalone skill from inside IFleet via `child_process`.** Rejected — bypasses the `WorkerAdapter` contract (no rate-limit handling, no session resume, no event streaming into the trace).

### Consequences

**Positive:** Single review prompt template across both surfaces. IFleet's pre-existing `WorkerAdapter` lifecycle (rate-limit, abort, session resume) is reused — no parallel implementation. Trace gets a `cross-provider-reviewer:codex-verdict` event for free.

**Negative:** First fold-in PR has to handle the audit-fix detection regex + `.audits/` reading, which adds ~150 LOC to `cross-provider-reviewer.ts`. Mitigated by extracting a `findingsLoader.ts` helper.

**Reversibility:** Trivial — delete `cross-provider-reviewer.ts`, re-point splittasks strict mode at the standalone skill.

### Open questions

- **Symlink vs copy** for the shared prompt template (`~/.claude/skills/codex-review/review-prompt.md` ↔ `IFleet/src/pipeline/prompts/codex-review.md`)? Symlinks break on Windows checkouts; copy needs a CI check for drift. **Sebastian decision needed.**
- Does Codex need its own `WorkerSpec.role` enum value or piggy-back on `'reviewer'`? Suggest: piggy-back, the `WorkerAdapter.provider` field disambiguates.

---

## ADR-002 — `cross-provider-reviewer.ts` placement in the IFleet pipeline

**Status:** DRAFT
**Affects:** Pipeline ordering — sits between `plan-reviewer.ts` and `diff-reviewer.ts`

### Context

IFleet's current pipeline (per `src/pipeline/`): `interview → architect → plan-reviewer (M2, 502 LOC, veto-capable) → editor → diff-reviewer (251 LOC, Haiku-gated, Claude-only by default) → pr.ts`. The cross-provider invariant lives in `diff-reviewer.ts::assertCrossProviderRule` (PR #67/#68): if editor and reviewer share a provider, throw `CrossProviderRuleViolation` unless the pool only has one provider (warn-not-block per IFleet CLAUDE.md mandatory rule #2).

The standalone elevation system adds a *second* reviewer (Codex) that runs **in parallel** with `diff-reviewer.ts` for audit-fix PRs (the "Tiered review chain" in the keystone plan, lines 116-120). Feature PRs get Codex only; audit-fix PRs get Codex + Claude `verifier` subagent sandwich. Where does this live in the IFleet pipeline?

Three placement candidates:

1. **After `diff-reviewer.ts`** — sequential second opinion. Simplest, but doubles per-round latency.
2. **In parallel with `diff-reviewer.ts`** — `Promise.all([diffReviewer, crossProviderReviewer])`. Faster, but both verdicts must reconcile (what if Claude says approve, Codex says FAIL?).
3. **Conditional, replacing `diff-reviewer.ts` when PR cites `AUDIT-*`** — only audit-fix PRs get the cross-provider path; feature PRs stay on the existing Haiku-gated diff-reviewer.

### Decision

**Option 2 (parallel), with a structured reconciliation rule. `cross-provider-reviewer.ts` runs in `Promise.all` with `diff-reviewer.ts`. Verdict merge:**

- Both PASS → merge OK
- Either FAIL → block merge, attach both verdicts to PR
- Either NEEDS_REVISION → re-queue to editor with combined feedback (max 3 retries, existing pattern)
- Codex unavailable (rate-limited / CLI missing) → fall back to `diff-reviewer.ts` only, attach a `cross-provider: unavailable` banner (matches the Docker-unreachable fallback in ADR-0002)

### Alternatives considered

1. **Sequential (Option 1).** Rejected — doubles latency on the critical path; audit-fix PRs already cost two model rounds (editor + reviewer), adding a third sequential cuts throughput in half.
2. **Conditional replacement (Option 3).** Rejected — cross-provider review is valuable on feature PRs too (catches reviewer blind spots the Haiku gate misses). Reserving it for audit-fix only wastes the standing capability.
3. **Codex as the *only* reviewer when present.** Rejected — violates ADR-0001's MARS pattern (structured pushback between roles needs both reviewers writing to the trace, not one replacing the other).

### Consequences

**Positive:** Audit-fix PRs get a true two-provider gate (Codex + Claude). Cross-provider invariant is enforced by *additive* mechanism, not by editor/reviewer assignment — `diff-reviewer.ts::assertCrossProviderRule` can relax to warn-only because the cross-provider check is now always satisfied by the parallel Codex reviewer.

**Negative:** Reconciliation logic adds complexity. Mitigated by keeping the merge rule simple (FAIL wins, no weighted voting).

**Reversibility:** Easy — collapse to sequential or remove the parallel path. The `WorkerSpec` and `WorkerPool` contracts don't change.

### Open questions

- Cost cap: should `cross-provider-reviewer.ts` inherit `planReviewer.costCapPctOfArchitect` from `config/routing.json` (10% per ADR/upgrade-02), or get its own cap? Suggest: own cap, default 15% — Codex flat-rate doesn't share the dollar-cap concern, but does compete for wall-clock time.
- Trace event name: `cross-provider-reviewer:verdict` vs `codex-reviewer:verdict`? Suggest the former — keeps the role provider-agnostic so swapping Codex for another provider later doesn't break consumers.

---

## ADR-003 — Audit findings as IFleet trace events

**Status:** DRAFT
**Affects:** `SprintManager` event bus, `.audits/index.json` ↔ trace bidirectional mapping

### Context

The standalone system writes audit findings to `<repo>/.audits/<timestamp>.json` with the schema from `plan.md` (lines 74-98) in the audit-elevation repo. Each finding has an `id` (`AUDIT-<repo>-<hash8>`), `fingerprint` (sha256 of category+globs+title), `status` (open/fixing/verifying/closed/regression), and `closing_pr`. The standalone path keeps `.audits/index.json` in sync.

IFleet's `SprintManager` already owns a canonical event trace per ADR-0001. Events have a `(taskId, seq, ts, role, kind, payload)` shape and persist to SQLite + S3-compatible blob. Audit findings are conceptually trace events too — they have a lifecycle (opened → fixing → verifying → closed/reopened), they reference PRs, and they need replay for the shadow eval and the M4 fingerprinting tables.

The question: do we keep `.audits/index.json` as the source of truth and have IFleet *read* it, or do we make IFleet's trace the source of truth and *derive* `.audits/index.json` from it?

### Decision

**Bidirectional with one canonical writer per direction. `.audits/index.json` is the source of truth for repos IFleet doesn't manage (writer = `audit-scan` subagent). For repos IFleet *does* manage, the trace is the source of truth and `.audits/index.json` is a derived view (writer = nightly rollup, same pattern as `learnings.md` in ADR-0001).**

Three new `TraceEvent.kind` values:

- `audit.finding.opened` — payload: full finding JSON (id, severity, fingerprint, file_globs, etc.). Emitted by the audit-scanner role.
- `audit.finding.closed` — payload: `{ findingId, closingPr, verifierEvidence }`. Emitted by `diff-reviewer.ts` / `cross-provider-reviewer.ts` when a PR merges that cites the finding.
- `audit.finding.regressed` — payload: `{ findingId, originalClosePr, regressionFingerprint, newOccurrenceTaskId, regression_of, regression_closing_pr }`. Emitted by the M4 regression detector (T3 builds the standalone version tonight; M4 wires the IFleet version). Note: `regression_of` (original finding id) and `regression_closing_pr` (PR number of the original fix) align this event with the scan-level regression fields in self-heal-pipeline.md so the bidirectional sync described in this ADR does not lose data.

`.audits/index.json` for IFleet-managed repos is regenerated nightly from `audit.finding.*` events, same cron that derives `learnings.md`. Stale entries (finding marked open in `.audits/` but `audit.finding.closed` in trace) are reconciled — trace wins.

### Alternatives considered

1. **Keep `.audits/index.json` canonical everywhere.** Rejected — violates ADR-0001 single-trace invariant. Anything that mutates audit state outside the trace breaks replay and shadow-eval.
2. **Drop `.audits/index.json` entirely, only trace.** Rejected — non-IFleet repos (Factory, the standalone `~/.claude/`) need the file-based ledger because they have no SprintManager. Cross-repo audits need to work without IFleet.
3. **Event-sourced `.audits/` with append-only files instead of `index.json` rollup.** Rejected — adds operational complexity (rollup queries) without the trace-replay benefit; for IFleet-managed repos the SQLite trace already gives us that.

### Consequences

**Positive:** Replay works — the trace contains every audit lifecycle event, the M4 fingerprint table can join on `audit.finding.*` directly. The M5 Proposer reads from one place (the trace) for IFleet-managed repos, simplifying its query layer. Non-IFleet repos keep working unchanged.

**Negative:** Two writers (audit-scanner for non-IFleet, IFleet pipeline for managed repos) creates a "which writer applies" decision at each invocation. Mitigated by a repo-level config flag (`IFLEET_MANAGED=true` in the repo's `.audits/.config.json`).

**Reversibility:** Hard — once IFleet-managed repos stop having `.audits/index.json` as a primary, restoring would require a backfill from the trace. Estimated 1-day cost per repo. **Worth doing right the first time.**

### Open questions

- Event payload size: a full finding JSON can be 1-5 KB. With 17 repos × 20 findings/week × 4 weeks = ~6 MB of audit events/month. Acceptable for SQLite; flag for S3 blob compression at M3 boundary.
- Does the nightly `.audits/` regeneration need to run *before* the morning Discord brief, or can it be best-effort? Suggest: before — the brief should reflect canonical state.

---

## ADR-004 — Lane scheduler ↔ IFleet daemon coordination

**Status:** DRAFT
**Affects:** Future relationship between `~/.claude/scripts/lane-scheduler.mjs` (T4 ships tonight) and `IFleet/src/orchestrator/daemon.ts`

### Context

T4 ships an observation-only lane scheduler tonight that reads `~/.omc/active-lanes.json` and emits telemetry. Each Claude Code terminal registers itself on start (kind, owns_globs, started_at). The scheduler does NOT throttle or block — Phase 4+ adds enforcement once 2 weeks of telemetry inform the policy.

IFleet has its own daemon (`src/orchestrator/daemon.ts`) that manages sprint queue depth, worker pool pressure (`pressure.ts`), and capability gating (`capabilities.ts`). It already throttles — it just doesn't know about Claude Code terminals that aren't IFleet sprints (Sebastian's REPL session, audit-scan invocations, splittasks lanes).

Two terminals running heavy Opus prompts simultaneously hit the single-seat Max-plan policy (IFleet CLAUDE.md mandatory rule #1). The lane scheduler's whole point is making that pacing decision system-wide, not per-product. Should IFleet's daemon *become* the lane scheduler, or should they coordinate via shared state?

### Decision

**Coordinate via shared state file (`~/.omc/active-lanes.json`). IFleet's daemon registers each sprint as a lane on start, deregisters on done. The lane scheduler is the *system-wide* arbiter; IFleet's daemon is the *IFleet-internal* throttle. The shared file is the integration surface — no IPC, no socket, no daemon-to-daemon protocol.**

Concretely:

1. `IFleet/src/orchestrator/daemon.ts` gains a `LaneRegistrar` interface that appends an entry to `~/.omc/active-lanes.json` on sprint-start and removes it on sprint-end. Schema matches T4's spec (`{id, kind, started_at, owns_globs}`).
2. Before spawning a new sprint, IFleet's daemon checks `~/.omc/active-lanes.json` for the system-wide lane count. If above the soft cap (Phase 4 sets the number), the sprint queues instead of spawning.
3. The lane scheduler stays observation-only in Phase 4; *IFleet's* daemon enforces the cap because it's the one that owns sprint lifecycle. Other Claude Code terminals (audit-scan, splittasks) keep being unregulated — they're transient and bursty by nature.
4. If the lane scheduler later grows enforcement (Phase 5+, blocking new terminal spawns), IFleet's daemon defers to it — both read the same cap from `~/.omc/lane-config.json`.

### Alternatives considered

1. **Merge IFleet daemon into the lane scheduler.** Rejected — IFleet's daemon has product-specific concerns (sprint queue, worker pool, GitHub webhook ingestion) that don't belong in a general-purpose scheduler. Conflating them is a categorical violation of "deep modules" (per `~/.claude/skills/references/deep-modules.md`).
2. **Daemon-to-daemon IPC (unix socket, HTTP).** Rejected — adds an always-on dependency between two services that should be loosely coupled. Shared file is simpler and survives either daemon restarting.
3. **Lane scheduler reads IFleet's SQLite store directly.** Rejected — couples the scheduler to IFleet's schema. The shared JSON file is provider-agnostic.

### Consequences

**Positive:** IFleet stays the IFleet-throttle. Lane scheduler stays the observation-and-eventually-enforcement layer. Either can ship independently; either can be replaced without touching the other.

**Negative:** Shared-file race conditions on registration (two terminals appending simultaneously). Mitigated by mkdir-based locking (as used in lane-lock.sh — macOS has no flock(1)) per T4's lane-scheduler-spec.md (T1 to verify in Phase C).

**Reversibility:** Easy — remove the `LaneRegistrar` calls in `daemon.ts`, IFleet reverts to per-product throttling.

### Open questions

- Should the lane scheduler get a vote on *which* sprint IFleet runs next (prioritize audit-fix sprints over feature sprints when capacity is tight)? Suggest: NO in Phase 4 — too much coupling. Reconsider for Phase 5.
- Lane TTL: if IFleet crashes mid-sprint, the entry stays in `active-lanes.json` forever. Suggest: each entry has a `heartbeat_ts` field; scheduler reaps entries older than 30 minutes.

---

## ADR-005 — Proposer (M5) consuming audit findings

**Status:** DRAFT
**Affects:** M5 Proposer architecture (currently spec-only, T5 drafts the spec tonight)

### Context

The M5 Proposer is the nightly bot that reads `ROADMAP.md` + `SPRINT.md` and DMs Sebastian a splittasks-shaped plan for the morning. The standalone elevation system adds a third input source: `.audits/index.json` (open CRITICAL/IMPORTANT findings across all repos). The Proposer should weigh roadmap progress against audit debt — if 12 CRITICAL findings are open, the morning plan should not propose new features.

Three input scopes to nail down:

1. **What does the Proposer read?** `ROADMAP.md` (high-level), `SPRINT.md` (active commits), `.audits/index.json` (open findings, per-repo). Maybe `learnings.md` (derived nightly per ADR-0001) for "don't propose this pattern again" guidance.
2. **What does the Proposer NOT read?** Per `NON_GOALS.md`, anything out of explicit scope. Per `SECURITY.md`, protected paths get flagged not auto-proposed.
3. **Output format:** splittasks paste-box (Sebastian copies into terminals) vs auto-dispatched splittasks session (Proposer creates the session dir, Sebastian only confirms). Standalone audit-autopilot keyword today does the former. The Proposer should do the same — DM in Discord, paste-box format, Sebastian-in-the-loop until 2-week soak validates the workflow.

### Decision

**Proposer reads (in priority order): `.audits/index.json` open CRITICAL findings → `SPRINT.md` active items → `ROADMAP.md` next-up → `learnings.md` derived guidance. Output is a splittasks paste-box DM'd to `#ifleet-proposals` channel (NOT auto-dispatched). Sebastian confirms by replying ✅ in Discord; the existing `discord-mcp` add_reaction handshake (per global CLAUDE.md Discord brief protocol) triggers the actual splittasks render.**

Decision rules baked into the Proposer's morning prompt:

1. If ≥3 CRITICAL findings open on any repo → morning plan prioritizes audit-fix over feature work for that repo. Plan title becomes "Audit triage: <repo>".
2. If active `SPRINT.md` has unfinished items → propose finishing those before opening new fronts.
3. Never propose work that touches `SECURITY.md` protected paths — surface as "needs Sebastian decision" instead of "auto-dispatch".
4. Never propose work explicitly listed in `NON_GOALS.md`.
5. Cap proposal at 5 lanes (matches splittasks default; respects single-seat Max-plan policy).

### Alternatives considered

1. **Proposer auto-dispatches without confirmation.** Rejected — too much faith in the planner heuristics during soak. Sebastian-in-the-loop until Proposer accuracy proven.
2. **Proposer reads only `.audits/`, leaves roadmap to humans.** Rejected — undersells the Proposer; the value is weighing audit debt against feature progress.
3. **Proposer writes a markdown file under `~/.omc/proposals/` instead of DMing Discord.** Rejected — Sebastian works from Discord (per Discord brief protocol). File-based proposals get missed.

### Consequences

**Positive:** Morning becomes "read DM, react ✅, paste paste-box" — zero typing, full review. Audit debt becomes a first-class scheduling input instead of a separate `/audit-fix` invocation Sebastian has to remember.

**Negative:** Proposer prompt is now longer (4 inputs + 5 decision rules). Token cost per nightly run ~3-5k input tokens; acceptable on Max plan.

**Reversibility:** Trivial — disable the nightly cron, Proposer goes dormant. Standalone `/audit-fix` and `/splittasks` keep working unchanged.

### Open questions

- Channel name: `#ifleet-proposals` (per `ifleet_elevation_plan.md`) or `#ifleet` (existing brief channel, `1504120127791042631`)? Suggest: new `#ifleet-proposals` so morning proposals don't get lost in operational chatter. **Sebastian decision needed.**
- Cron timing: **03:00 local** — resolved by proposer-spec.md D7 (canonical). 5h buffer before 08:00 start covers Discord MCP retries.
- Budget gate (per `ifleet_elevation_plan.md` M5 line): does the Proposer estimate cost before DMing, and refuse if over a daily cap? Per `feedback_no_budget_caps_claude_max.md`, NO — Max plan is flat-rate, the scarce resource is lanes not dollars. The 5-lane cap above subsumes the budget gate.

---

## Summary: Sebastian decisions needed before any fold-in PR

| # | Decision | ADR | Default if no answer |
|---|---|---|---|
| 1 | Symlink vs copy for shared Codex review prompt | 001 | Copy + CI drift check |
| 2 | Cross-provider reviewer own cost cap value | 002 | 15% of architect cost |
| 3 | `.audits/.config.json` `IFLEET_MANAGED` flag schema | 003 | Boolean, default false |
| 4 | Lane TTL / heartbeat interval | 004 | 30 min idle reap |
| 5 | `#ifleet-proposals` channel creation | 005 | Reuse `#ifleet` (1504120127791042631) |

## Plain-language recap

These five draft ADRs describe how the standalone elevation system (the audit scanner, Codex reviewer, lane scheduler, and Proposer that get built this week and next) would eventually fold into IFleet so they aren't permanently separate. The big calls: (1) the standalone Codex review skill stays usable for non-IFleet repos but IFleet itself gets a thin in-pipeline wrapper that calls the existing `WorkerAdapter` so we don't end up with two prompt templates drifting; (2) audit findings become real trace events inside IFleet for repos IFleet manages, and a derived `.audits/index.json` file outside IFleet for everything else; (3) IFleet's daemon and the new lane scheduler coordinate by sharing a JSON file in `~/.omc/`, not by merging into one mega-daemon; (4) the M5 morning Proposer always prioritizes open CRITICAL audit findings over new feature work, and DMs you a paste-box in Discord rather than auto-dispatching, until the soak proves it can be trusted. None of this gets built tonight — these ADRs are the architectural skeleton so when the fold-in plan starts (after the 2-week soak), we already agree on the shape. Five small decisions need your input before any code lands; the table above lists them with safe defaults.

---

🗣️ In plain terms:
Tonight's overnight build is happening as five parallel terminals; T1 (this) writes the documents that say how everything we ship tonight will *eventually* connect into IFleet (the autonomous fleet repo) months from now. The five ADRs are draft-only — no IFleet code changes — and each ends with the specific decisions you need to make before the actual fold-in PRs are written. The morning brief will surface these decisions explicitly so you can answer them over coffee instead of digging through this file.
