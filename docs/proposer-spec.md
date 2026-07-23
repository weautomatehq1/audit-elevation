# Proposer — IFleet M5 spec

> **Status:** DRAFT — design only. No IFleet source code lands until Sebastian approves (Phase 4 gate pending — see README, 49 days overdue as of 2026-07-23).
> **Author:** T5 (overnight split `20260520-2244-elevation-push`, 2026-05-20).
> **Predecessors:** ROADMAP.md M5, ADR-0001 (single-shared-trace), SECURITY.md, NON_GOALS.md.
> **Companion:** `~/.claude/skills/audit-broadcast/` — the channel the Proposer reuses for its morning DM/post.

The Proposer is the nightly autonomous bot that reads what we know (roadmap, sprint, audits, learnings) and proposes the next morning's splittasks plan via Discord. Sebastian approves with ✅ before any worker runs.

---

## 1. Purpose

Turn "Sebastian wakes up and decides what to work on" into "Sebastian wakes up to a ready splittasks plan and either approves or redirects." The Proposer never executes — it only proposes. Approval is one Discord reaction; rejection is one word.

This is the M5 milestone on the IFleet ROADMAP. It is the first goal-driven feature in the fleet and the highest-noise risk, so noise discipline is the design's first concern.

## 2. Inputs

Read-only. The Proposer never writes to source.

| Source | Path | Used for |
|---|---|---|
| Roadmap (top-level direction) | `~/dev/ai-products/IFleet/ROADMAP.md` | Identify the current month's milestone + KPIs |
| Sprint (this week) | `~/dev/ai-products/IFleet/SPRINT.md` | Identify in-flight tasks vs. open slots |
| Non-goals filter | `~/dev/ai-products/IFleet/NON_GOALS.md` | Drop any proposal whose summary matches |
| Security gates | `~/dev/ai-products/IFleet/SECURITY.md` | Flag any proposal touching protected paths — require explicit Sebastian approval flag |
| Audit findings (watch-list) | `<repo>/.audits/index.json` for each repo in `~/.omc/proposer-watchlist.json` | Surface CRITICAL/IMPORTANT findings as candidate tasks; regressions (status == "regression") receive +30 score bonus (loaded in same pass — not a separate file read) |
| Audit closures | `<repo>/.audits/closed.json` | Suppress already-resolved findings |
| Lane patterns | `~/.omc/lane-scheduler/log/*.jsonl` (last 7 days) | Detect overload — if yesterday hit lane ceiling, propose smaller plan |
| Learnings | `~/dev/ai-products/IFleet/learnings.md` | Inform ranking ties (avoid recently-flagged anti-patterns) |
| Yesterday's proposal | `~/.omc/proposer/state/last-proposal.json` | Suppress duplicates; track Sebastian's accept/reject pattern |

**Watch-list file** (`~/.omc/proposer-watchlist.json`) is the single source of truth for which repos the Proposer scans. Sebastian edits this by hand. Default contents:

```json
{
  "repos": [
    "~/dev/ai-products/IFleet",
    "~/dev/coordination/factory",
    "~/dev/products/PhillUp",
    "~/dev/coordination/discord-claude-bot"
  ]
}
```

## 3. Ranking algorithm

**Pre-filter (before scoring):** drop any finding with `status` not in `["open", "regression"]`. DUPLICATE findings are excluded from `index.json` by the regression-detector; if any appear (scanner bug), log a warning and skip them. COSMETIC findings with `status: "closed"` or `"fixed"` are also skipped — only open work items enter scoring.

Each candidate task gets a numeric score. Higher score → higher position in the morning plan.

| Class | Priority base | Notes |
|---|---|---|
| CRITICAL audit findings | **100** | One per finding. Always above roadmap work. |
| REGRESSION (any severity) | **+30 bonus** added to its underlying-finding base | Stacks: a CRITICAL regression scores 130. |
| IMPORTANT findings affecting roadmap items | **80** | Affected = the finding's globs overlap a roadmap item's owned glob set. |
| Roadmap items unblocked + no findings on their globs | **60** | "Unblocked" = all predecessor IDs in ROADMAP.md marked `done`. |
| IMPORTANT findings NOT on the roadmap | **50** | Tech debt; eligible for proposal but loses ties to roadmap work. |
| COSMETIC findings | **20** | Batched into one "cosmetic sweep" task — never one per finding. |
| Already-proposed-yesterday (rejected by Sebastian) | **−40 penalty** | Suppress nagging. Drops items below the cut-line. |

After scoring:

1. Apply the **NON_GOALS filter**: drop any candidate whose summary text matches a non-goal heading or bullet (case-insensitive substring match — keep it dumb; if Sebastian wants stricter matching, that's an open decision).
2. Apply the **SECURITY filter**: any candidate touching a path under `SECURITY.md`'s protected-paths list gets a `requires_security_approval: true` flag AND drops in rank to the bottom of its class. The morning DM then lists it under a "🔒 requires explicit approval" section, never auto-mixed with normal work.
3. Apply the **lane-budget filter**: based on T4's lane scheduler observations from yesterday — if Sebastian ran ≥5 lanes for ≥4h, cap morning plan at 3 lanes; if ≤2 lanes for ≤2h, allow up to 6 lanes. Default cap: 5 lanes.
4. Top N (by score, post-filters) become the proposal. N = lane-budget cap.

Tie-breaking (same score): older finding wins, then alphabetical by repo, then by id.

## 4. Output

A single Discord message (DM to Sebastian or post to a project channel — see Open Decisions §8). The message contains:

1. **Greeting line.** `[Sebas]` prefix per CLAUDE.md routing convention. Body opens with `Proposer plan — <UTC-date>`.
2. **Headline counts.** "N CRITICAL · M IMPORTANT · K regressions · L roadmap items unblocked".
3. **The plan**, as a ready-to-paste splittasks block (markdown table with terminal name, role, model, focus). Format identical to what Sebastian pastes by hand today. Each row also lists the source finding id(s) so the worker can `/audit-fix AUDIT-<id>` directly.
4. **🔒 Security-flagged items** (separate section, only present if non-empty).
5. **Why this order** (one short paragraph — the actual KPI the plan moves).
6. **Reject path.** "Reply with ❌ to reject, or send a redirected prompt. Otherwise react ✅ to greenlight the morning."
7. **Plain-language recap** (mandatory per CLAUDE.md). 3–6 sentences.

The plan is also written to `~/.omc/proposer/state/proposed-<UTC-date>.md` for audit trail.

## 5. Schedule

- **Cron / launchd:** 03:00 local time. Configured via `~/.omc/proposer/launchd/com.weautomatehq.proposer.plist` (host TBD per §8). Not installed by this spec — Sebastian installs after approval.
- **Manual:** `/proposer-run` slash command. Runs the same pipeline ad-hoc, marks the output `MANUAL` in the state file. Use cases: Sebastian wants a fresh plan after a long session, Esme wants to see what the Proposer would say.
- **Frequency cap:** at most one DM per UTC date unless Sebastian explicitly requests another via `/proposer-run --force`.

## 6. Failure modes

| Failure | Detection | Response |
|---|---|---|
| **No findings, no roadmap work unblocked** | Final candidate list empty after scoring | Send a single-line DM: `Proposer: nothing to propose — repos green, roadmap clear. Take the morning slow.` Mark state `EMPTY`. |
| **Discord MCP unavailable** | `mcp__discord-mcp__send_message` throws or times out (>10s) | Write proposal to `~/.omc/proposer/state/proposed-<date>-UNDELIVERED.md`. Retry every 30 minutes for 2 hours. After that, log to `~/.omc/proposer/log/errors.jsonl` and stop. |
| **Sebastian already running a session** | Detected via T4's `~/.omc/active-lanes.json` showing ≥1 entry started within last 6h | DM is sent with a prefix banner: `⚠️ You appear to still be working — see lanes [list]. The plan below assumes a fresh morning; reply ❌ if you'd rather continue current work.` |
| **All candidates filtered out (NON_GOALS / SECURITY)** | Scoring produced ≥1 candidate, post-filters produced 0 | Send a "filtered to zero" message listing what was dropped and why, so Sebastian can see what the Proposer wanted to suggest. |
| **Stale input** (roadmap or sprint older than 14 days) | mtime check at start | Send a "Roadmap is stale — refresh before I run again" DM. Skip proposal. |
| **Budget gate closed** (see §7) — this is the intentional default, not an error | Default: gate is closed | Generate the proposal but do NOT send. Write to state file with `WITHHELD_BY_BUDGET_GATE`. |

## 7. Gates

### Budget gate

The Proposer is mute until Sebastian explicitly enables it. Mechanism: a flag file `~/.omc/proposer/ENABLED`.

- **If absent (default on install):** compute proposal and write to state, but DO NOT send to Discord. This is the INTENTIONAL default, not an error condition.
- **If present:** compute proposal and send to Discord.

The flag is lifted only on IFleet M5 launch, after Sebastian inspects two weeks of `WITHHELD` proposals and confirms the noise is acceptable.

Rationale: M5's KPI explicitly tolerates "≥1 approved+merged proposal/week, **0 noise complaints**". A Proposer that DMs noise on day one fails the KPI by definition.

### SECURITY.md respect

Any candidate touching a path matched by `SECURITY.md`'s protected-paths list is segregated into the `🔒 requires explicit approval` section. The morning message format requires Sebastian to react with a separate ✅ on that section before any worker can act. Workers consuming the plan MUST check the `requires_security_approval` flag in the source state JSON and refuse if missing.

### Plain-language recap

Mandatory in the Discord DM (per CLAUDE.md "Plain-language recap" rule). The Proposer's prompt template emits the recap automatically; the output post-processor verifies the `🗣️ In plain terms:` block exists before send. If missing, abort send and log.

## 8. Open decisions for Sebastian

| # | Decision | Options | T5's recommendation |
|---|---|---|---|
| **D1** | Run on Sebastian's Mac via launchd, or on Hostinger VPS where the IFleet daemon lives? | (a) Mac launchd — closest to repos, no SSH friction. (b) VPS daemon — runs even when Mac sleeps. (c) Both, with Mac as primary and VPS as 24h fallback. | **(a) Mac launchd for v1.** The Proposer reads from `~/dev/...` directly; rsyncing repos to VPS is its own headache. Move to VPS only if morning misses become a pattern. |
| **D2** | Extend IFleet's existing daemon process, or run as standalone? | (a) Extend `IFleet/src/orchestrator/` — reuses approval-gate.ts plumbing. (b) Standalone `~/.omc/proposer/` script — zero IFleet coupling for v1. | **(b) Standalone for v1.** Per ADR-0001's "single shared trace" rule, anything writing into the IFleet trace should be inside IFleet. The Proposer is read-only and pre-trace, so it lives outside until M5+. |
| **D3** | DM Sebastian only, or also post to `#ifleet` / `#ifleet-proposals`? | (a) DM only. (b) DM + summary line in #ifleet. (c) Dedicated `#ifleet-proposals` channel (per ROADMAP M5 line). | **(c) — match ROADMAP.** ROADMAP M5 explicitly lists `#ifleet-proposals`. Create it on enable; DM Sebastian a pointer, post the full plan in the new channel so Esme can also see it. |
| **D4** | Allow Esme to ✅ a proposal? | (a) Sebastian only. (b) Either of them. | **(a) Sebastian only for v1.** Esme can react with comments; only Sebastian's ✅ greenlights. Revisit after first month. |
| **D5** | What does the worker do when picking up an approved plan? | (a) Sebastian copies into a fresh splittasks invocation by hand. (b) Slash command `/proposer-accept` auto-builds the splittasks dir from the state file. | **(b) for v1.5.** Start with (a) — fewer moving parts; once the format settles, automate. |
| **D6** | Reject feedback loop | Should ❌ reactions become learnings? | **Yes — store reject reasons in `~/.omc/proposer/state/rejections.jsonl` and feed into next-night's `−40` penalty calculation.** This is the first behavioral-fingerprinting touch point ahead of M4. |
| **D7** | Cron time | 03:00 local? Earlier? | **03:00 local.** Sebastian usually starts at 08:00; 5h buffer covers Discord MCP retries. |

These decisions block any IFleet PR. Sebastian answers on wake; T5 (or whoever's next) re-spins the spec with answers baked in.

---

## ADR-Proposer-001: Single-source-of-truth for ranking input

**Status:** DRAFT — Sebastian to approve
**Date:** 2026-05-20
**Author:** T5 (overnight split `20260520-2244-elevation-push`)
**Decider:** Sebastian Puig (pending review)
**Affects:** M5 Proposer architecture — how inputs are read at proposal time

**Context:** The Proposer reads from at least eight sources (ROADMAP, SPRINT, NON_GOALS, SECURITY, multiple `.audits/`, lane logs, learnings, yesterday's state). Any stale input produces a wrong proposal. If the proposal is wrong, Sebastian loses trust in the Proposer fast, and the M5 KPI ("0 noise complaints") fails. We need an explicit decision about how the Proposer determines which inputs are authoritative on a given morning.

**Decision (Sebastian to pick A / B / C):**

| Option | What it means | Trade-offs |
|---|---|---|
| **A — Trust live files** | Read every input file at run time. No caching. If a file's mtime is older than 14 days, refuse to propose. | + Simple. + Always fresh. − Slow (8+ file reads + JSON parses). − Brittle if any input was edited mid-night. |
| **B — Nightly snapshot** | Before scoring, copy every input into `~/.omc/proposer/state/snapshot-<date>/`. Score from the snapshot. Snapshots persist for 14 days for audit. | + Reproducible — Sebastian can re-run the same scoring later. + Snapshot becomes the single source of truth for the morning. − One more directory to clean up. − Snapshot drift if Sebastian edits roadmap between snapshot and run. |
| **C — Index-only** | Maintain a single `~/.omc/proposer/state/index.json` that the Proposer updates incrementally as files change (file-watcher). At run time, read only the index. | + Fastest. + Decoupled from where source files live. − Complex (needs a watcher process, which is what T4's lane scheduler already is — coupling risk). − If watcher dies overnight, Proposer wakes blind. |

**T5's recommendation:** **Option B — Nightly snapshot.** It is the cheapest reproducibility win and aligns with the trace-everything ADR-0001 philosophy (snapshots are an audit trail). Option A wastes the next morning if Sebastian wants to see "why did you propose X" — there's no record. Option C is M6-level engineering for an M5 feature.

**Consequences:**

- ✅ Sebastian can re-run a past morning's scoring (debug + behavior calibration).
- ✅ Snapshots become input to behavioral-fingerprinting (M4) for free — every reject decision is grounded in a frozen snapshot.
- ✅ Watcher decoupling — Proposer and lane-scheduler stay independent processes.
- ⚠️ Disk: ~14 × 5MB = 70MB max per repo on the watch-list. Cheap, but worth noting.
- ⚠️ Snapshot drift: if Sebastian edits ROADMAP.md between 03:00 (snapshot) and 08:00 (review), the proposal reflects the pre-edit version. Mitigation: the morning DM includes a `Snapshot: <timestamp>` line so Sebastian sees what frame the Proposer worked from.
- ⛔ If Sebastian picks A or C, this ADR's downstream assumptions (M4 fingerprint inputs, debug re-run) need re-spec.

**Decision blocked on:** Sebastian's pick (A / B / C) before any code lands.

---

🗣️ In plain terms:
The Proposer is the bot that DMs you each morning with a ready-to-paste plan for the day, based on what audits found overnight and what's next on the IFleet roadmap. This document is design only — no code is written yet. The eight open decisions at the bottom (D1–D7 plus ADR-001) are the things only you can answer: where it runs, how it talks to Discord, whether Esme can approve, and most importantly the "snapshot vs. live read" question that controls whether you can debug a proposal later. The default ships with the budget gate **closed** so it can't spam you until you flip the flag yourself.
