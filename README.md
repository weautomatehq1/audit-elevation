# Audit Elevation System

> Structured-audit + self-healing layer for Claude Code, sitting on top of splittasks (`~/.claude/skills/splittasks/SKILL.md` — local infrastructure, no public repo) and feeding IFleet's M4 fingerprint pipeline. Single-developer internal infrastructure. Built 2026-05-20 → 2026-05-21 across two overnight splits.

---

## What this is

A coordinated system that turns Claude Code's ad-hoc audit chatter into **structured, queryable, self-healing work items.**

Before: `/audit-session` produced a wall of text in chat. Findings vanished after compaction. Nothing else could read them.

After: `/audit-scan` writes `<repo>/.audits/<ISO>.json` with deterministic IDs and fingerprints. Splittasks reads `.audits/index.json` and routes lanes around audit work. Codex (separate quota pool) reviews audit-fix PRs. Fingerprints repeat → DRAFT rules surface for review. Lane scheduler observes everything for future quota pacing.

**Why the name "elevation":** lifts the system from "tool that runs once" to "tool that gets categorically smarter over time."

---

## Goal

Make audit findings *first-class on-disk work items* so:

1. Parallel splittasks lanes can route around audit work safely (no file collisions)
2. Audit-fix PRs cite finding IDs and a second-opinion reviewer (Codex) can verify closure
3. Repeated bugs get fingerprinted → after 3 closures, a DRAFT rule surfaces for human approval
4. The lane scheduler can eventually pace Max-quota burn across concurrent sessions
5. IFleet's M4 fingerprint+PR-learn pipeline ingests this same data later

---

## Status

| Phase | Status | Date |
|---|---|---|
| **Phase 1 — Keystone** (commands + triager + Step 0) | ✅ shipped | 2026-05-20 overnight |
| **Phase 2 — Codex review + Tiered chain** | 🔴 BLOCKED — Codex CLI not installed; requires Sebastian setup | 2026-05-20 (design only) |
| **Phase 2.5 — Hybrid automation** (SessionStart banner + `audit autopilot` keyword) | ✅ shipped | 2026-05-20 overnight |
| **Online-eval-log v1** (Supabase-backed session-close capture) | ✅ shipped | 2026-05-21 |
| **Phase 3 — Dogfood / 2-week soak** | 🟡 soak complete (65 days elapsed as of 2026-07-25) — ⚠️ Phase 4 gate evaluation 51 days overdue (soak completed 2026-06-04; Sebastian decision needed) | started 2026-05-21 |
| **Phase 4 — Self-healing (fingerprint REGRESSION + DRAFT auto-rules)** | ✅ scaffolded | 2026-05-21 overnight |
| **Phase 5 — Lane scheduler MVP (observation-only)** | ✅ scaffolded, opt-in PM2 entry | 2026-05-21 |
| **Phase 6 — IFleet fold-in** | 📝 ADRs drafted, 5 Sebastian decisions queued | gated on Phase 3 soak |
| **Phase 7 — Proposer (M5 of IFleet elevation)** | 📝 spec drafted | future |

6 repos baselined (IFleet, factory, PhillUp, discord-claude-bot, ~/.claude, audit-elevation). As of 2026-07-25: **audit-elevation** has 0 open findings (all previous open findings closed — see `.audits/index.json`) — Phase 4 gate evaluation 51 days overdue (c3a7f912 status updated). IFleet and factory findings tracked in their own `.audits/index.json` files. The 52-finding baseline (6 CRITICAL / 37 IMPORTANT / 9 COSMETIC) was as of 2026-05-21 initial scan.

---

## Online-eval-log (v1, shipped 2026-05-21)

One structured row per session close-out, written to Supabase. Accumulates longitudinal quality data for drift detection.

### Architecture

```
/audit-scan --emit-online-eval
       │
       ├─ critic subagent → findings array
       ├─ writes <repo>/.audits/<ISO>.json
       ├─ updates <repo>/.audits/index.json
       └─ POST → Supabase public.online_eval_log
              (session_id, event, findings_total,
               findings_new, findings_carried_over,
               repo, branch, git_sha, audit_model,
               rubric_version, trigger, source, is_test)

Write paths:
  splittasks T1 close-out  →  --trigger split-t1
  Solo session end         →  ~/.claude/hooks/online-eval-end.sh (SessionEnd)
  Manual                   →  --trigger manual
```

### Table

- **Project:** iFleet (`exswghbtgtdykklcsdxq`, us-east-1)
- **Table:** `public.online_eval_log` (22 cols, schema_version=1)
- **Schema source-of-truth:** `rubric.json` (severities/categories/fingerprint) + `rubric.json#online_eval_log_schema` (column list)

### Env vars required

```bash
export SUPABASE_URL=https://exswghbtgtdykklcsdxq.supabase.co
export SUPABASE_SERVICE_KEY=<service_role_secret_from_dashboard>
```

If either is unset, `/audit-scan --emit-online-eval` skips the POST silently (one-line warning) and continues.

### How to query

```bash
python3 ~/.claude/scripts/online-eval-stats.py --days 7
python3 ~/.claude/scripts/online-eval-stats.py --days 7 --repo IFleet --json | jq '.'
```

### Manual emission

```bash
/audit-scan --emit-online-eval --session-id <id> --trigger manual
```

### Deferred to v2

- VPS-side IFleet integration (Mac-only for v1)
- Per-lane audit rows (currently one row per T1 close-out, not per lane)
- Drift statistics: CUSUM/EWMA over rolling windows
- Discord brief automation from weekly aggregator
- Orphan-reaper for T1 deaths (split aborted, no T1-done written)

---

## Data flow (one diagram, all components)

```
                  ┌──────────────────────────────────────────────┐
                  │  Sebastian invokes /audit-scan in any repo   │
                  └─────────────────────┬────────────────────────┘
                                        ▼
                  ┌──────────────────────────────────────────────┐
                  │  critic subagent  ─OR─  audit-scanner agent  │
                  │  produces structured findings (severity,     │
                  │  globs, fingerprint hash, parallel_safe)     │
                  └─────────────────────┬────────────────────────┘
                                        ▼
                  ┌──────────────────────────────────────────────┐
                  │  Writes <repo>/.audits/<ISO>.json            │
                  │  Updates <repo>/.audits/index.json (rollup)  │
                  │  Checks fingerprint against closed.json →    │
                  │  marks as REGRESSION if matches              │
                  └─────────────────────┬────────────────────────┘
                                        ▼
            ┌───────────────────────────┴───────────────────────────┐
            ▼                                                       ▼
  ┌────────────────────┐                              ┌──────────────────────┐
  │  /audit-fix        │                              │  /splittasks         │
  │  Filters by sev    │                              │  Step 0: reads       │
  │  or id.            │                              │  .audits/index.json  │
  │  audit-triager     │                              │  Surfaces open       │
  │  produces a        │                              │  findings in         │
  │  splittasks plan.  │                              │  MASTER.md           │
  │  User pastes.      │                              │  Boundaries          │
  └─────────┬──────────┘                              └──────────────────────┘
            │
            ▼
  ┌────────────────────────────────────┐
  │  Worker lanes fix the findings,    │
  │  open PRs with AUDIT-* in title    │
  └─────────┬──────────────────────────┘
            ▼
  ┌────────────────────────────────────┐
  │  Strict mode tiered review chain:  │
  │  - Feature PR → Codex only         │
  │  - Audit-fix PR → Codex + Claude   │
  │    verifier in parallel (both PASS)│
  └─────────┬──────────────────────────┘
            ▼
  ┌────────────────────────────────────┐
  │  Merge → finding closed in         │
  │  index.json + appended to          │
  │  closed.json with fingerprint      │
  │  [Phase 4 — scaffolded, not yet    │
  │   active; closed.json not written] │
  └─────────┬──────────────────────────┘
            ▼
  ┌────────────────────────────────────┐
  │  audit-fingerprint-watcher.sh      │
  │  (cron-runnable, opt-in)           │
  │  3+ closures of same fingerprint → │
  │  audit-rule-drafter.sh writes      │
  │  DRAFT to .audits/proposed-rules/  │
  │  Sebastian reviews + manually      │
  │  promotes to .claude/rules/        │
  │  [Phase 4 — scaffolded, not yet    │
  │   active]                          │
  └────────────────────────────────────┘

   Concurrent with all of the above:
   - audit-banner.sh hook prints findings count at SessionStart
   - "audit autopilot" keyword flips to self-driving mode
   - audit-broadcast skill posts Discord briefs (dry-run today)
   - lane-scheduler.mjs observes active terminals (telemetry only)
```

---

## Where the code lives

| Component | Path | Purpose |
|---|---|---|
| **Commands** | | |
| `/audit-scan` | `~/.claude/commands/audit-scan.md` | Run brutal critique, write structured findings JSON |
| `/audit-fix` | `~/.claude/commands/audit-fix.md` | Dispatch findings as splittasks plan via triager |
| `/audit-session` | `~/.claude/commands/audit-session.md` | Alias — scan + suggest fix |
| `/codex-review` | invokes skill | Per-PR cross-provider review |
| `/audit-broadcast` | invokes skill | Post brief to Discord |
| **Subagents** | | |
| `audit-triager` | `~/.claude/agents/audit-triager.md` | Group findings → splittasks plan |
| `audit-fingerprinter` | `~/.claude/agents/audit-fingerprinter.md` | Compute deterministic fingerprint |
| `audit-regression-detector` | `~/.claude/agents/audit-regression-detector.md` | NEW vs REGRESSION vs DUPLICATE |
| `critic` (existing) | `~/.claude/agents/critic.md` | Reused as scanner role |
| `verifier` (existing) | `~/.claude/agents/verifier.md` | Used in Tiered chain for audit-fix PRs |
| **Skills** | | |
| `codex-review` | `~/.claude/skills/codex-review/` | Per-PR Codex CLI wrapper |
| `audit-autopilot` | `~/.claude/skills/audit-autopilot/` | Self-driving fix dispatch via keyword |
| `audit-broadcast` | `~/.claude/skills/audit-broadcast/` | Discord brief formatting + posting |
| `splittasks` (patched) | `~/.claude/skills/splittasks/SKILL.md` | Step 0 reads .audits/, strict-mode Tiered chain |
| **Hooks** | | |
| Audit banner | `~/.claude/hooks/audit-banner.sh` | SessionStart open-findings count |
| Keyword detector (patched) | `~/.claude/hooks/keyword-detector.mjs` | Routes `audit autopilot` keyword |
| **Scripts** | | |
| Online-eval aggregator | `~/.claude/scripts/online-eval-stats.py` | Query online_eval_log, print stats + JSON |
| SessionEnd hook | `~/.claude/hooks/online-eval-end.sh` | Fires /audit-scan --emit-online-eval on session close |
| Rule drafter | `~/.claude/scripts/audit-rule-drafter.sh` | 3+ closures → DRAFT rule |
| Fingerprint watcher | `~/.claude/scripts/audit-fingerprint-watcher.sh` | Cron-runnable repo sweeper |
| Lane scheduler | `~/.claude/scripts/lane-scheduler.mjs` | Observer daemon (opt-in PM2) |
| Audit status helper | `~/.claude/scripts/audit-status.sh` | Standalone CLI: finding counts per repo |
| **Config** | | |
| rubric.json | `rubric.json` | Single source of truth for severities, categories, fingerprint algo, events, triggers |
| **Data files** | | |
| Findings (per repo) | `<repo>/.audits/<ISO>.json` | One file per scan |
| Open findings rollup | `<repo>/.audits/index.json` | Aggregate across all scans |
| Closed findings table | `<repo>/.audits/closed.json` | History for fingerprint matching |
| DRAFT rules | `<repo>/.audits/proposed-rules/` | Pending Sebastian review |
| Active lanes registry | `~/.omc/active-lanes.json` | Lane scheduler observation data |

---

## Spec docs (in this folder)

| Doc | What it covers |
|---|---|
| [plan.md](./plan.md) | The original strategy plan that drove the build. Phases, build order, deferrals, decisions. |
| [docs/proposer-spec.md](./docs/proposer-spec.md) | The nightly Proposer bot (IFleet M5). 9 sections + 8 open decisions (D1-D7 + ADR-Proposer-001). |
| [docs/lane-scheduler-spec.md](./docs/lane-scheduler-spec.md) | Lane scheduler schema (`~/.omc/active-lanes.json`), helper scripts, observer daemon, PM2 opt-in. |
| [docs/self-heal-pipeline.md](./docs/self-heal-pipeline.md) | Fingerprint → REGRESSION detection → DRAFT auto-rule generation. Lifecycle + failure modes. |
| [docs/quota-pacing-design.md](./docs/quota-pacing-design.md) | Future design for Max-quota throttling. Not implemented — design only. |
| [docs/ifleet-integration-adrs.md](./docs/ifleet-integration-adrs.md) | 5 ADRs for the eventual IFleet fold-in (Phase 6). Gated on Phase 3 soak. |

---

## How to use

### First scan in a new repo
```
cd <repo>
/audit-scan
```
Findings appear in chat and at `<repo>/.audits/<timestamp>.json`. `index.json` rollup auto-updates.

### Triage + fix
```
/audit-fix severity:CRITICAL
```
Returns a splittasks plan. Paste into terminals. Each lane closes its assigned finding(s).

### Self-driving for a session
Type `audit autopilot` mid-session. Parent Claude reads `.audits/index.json`, dispatches the triager, renders paste-box. You paste terminals.

### Check status of any repo
```
~/.claude/scripts/audit-status.sh <repo-path>
# or just `audit-status` if symlinked into PATH
```

### Manual fingerprint sweep (rule drafting)
```
~/.claude/scripts/audit-rule-drafter.sh <repo-path>
# inspect <repo>/.audits/proposed-rules/
# promote any to <repo>/.claude/rules/ manually
```

### Cron-run the watcher (optional)
See [docs/self-heal-pipeline.md](./docs/self-heal-pipeline.md) for crontab snippet. Not installed by default.

### Lane scheduler (observation only, optional)
```
pm2 start ~/.claude/scripts/lane-scheduler.pm2.json
```
Telemetry only — no throttling. See [docs/lane-scheduler-spec.md](./docs/lane-scheduler-spec.md).

---

## Open decisions (Sebastian's input needed)

From the 5 IFleet integration ADRs in [docs/ifleet-integration-adrs.md](./docs/ifleet-integration-adrs.md):

1. **Symlink vs copy** — when IFleet integrates, should `~/.claude/agents/audit-*` symlink into IFleet, or be copied?
2. **Cost cap policy** — Max-plan flat-rate, but what's the safety cap on a runaway audit-fix?
3. **`IFLEET_MANAGED` flag** — env var that flips behavior when IFleet is driving vs user-driven?
4. **Lane TTL** — how long before a lane in `active-lanes.json` is considered stale?
5. **`#ifleet-proposals` channel** — DM Sebastian only, or also broadcast to channel?

Plus 8 open decisions in the Proposer spec (D1-D7 plus ADR-Proposer-001).

---

## Safety gates honored throughout

- ✋ No IFleet source code changes (specs only)
- ✋ Auto-rules DRAFT only — never `.claude/rules/`
- ✋ Lane scheduler observation-only — no throttling
- ✋ Discord broadcast `--dry-run` only (live needs explicit go)
- ✋ PM2 / crontab documented but not installed
- ✋ No git pushes from audit baselines without explicit approval

---

## Related

> Note: paths below are local filesystem references — they are not clickable links in GitHub/VS Code.

- IFleet ROADMAP: `~/dev/ai-products/IFleet/ROADMAP.md` — 6-month elevation plan (M0-M6, M4 is the natural integration point)
- Memory entries: `~/.claude/projects/-Users-Seb/memory/` — `elevation_audit_shipped_20260521.md`, `project_status_20260521.md`
- Split session evidence: `~/.omc/splits/20260520-2224-elevation-keystone/` — keystone build T1-T5 done reports
- Split session evidence: `~/.omc/splits/20260520-2244-elevation-push/` — push build T1-T5 done reports + ADRs
