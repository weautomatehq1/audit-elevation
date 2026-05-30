# Lane Scheduler — Specification

**Status:** observation-only MVP (Phase 4 will add quota pacing). See `quota-pacing-design.md` for the future design.

**Owner lane:** T4 of split `20260520-2244-elevation-push`.

**Scope:** track active Claude Code lanes (terminals) in a shared registry so a daemon and human helpers can answer "what is currently running, how long has it been running, and is anything stale?" The daemon **does not throttle, block, or rate-limit** anything. It only writes telemetry.

---

## 1. The shared registry — `~/.omc/active-lanes.json`

Canonical schema (version `1`):

```json
{
  "version": 1,
  "lanes": [
    {
      "id": "T2-push-20260520-2244",
      "kind": "audit-scan | audit-fix | dogfood | self-heal | scheduler | broadcast | feature | other",
      "model": "claude-opus-4-8",
      "started_at": "2026-05-21T02:50:32Z",
      "last_heartbeat": "2026-05-21T02:51:01Z",
      "owns_globs": ["src/api/**", ".audits/**"],
      "session_dir": "/Users/Seb/.omc/splits/<id>/",
      "terminal_label": "T2 — dogfood-audit",
      "estimated_done_at": null
    }
  ],
  "completed_today": [
    {
      "id": "T2-push-20260520-2244",
      "kind": "dogfood",
      "model": "claude-opus-4-8",
      "started_at": "2026-05-21T02:50:32Z",
      "ended_at": "2026-05-21T03:42:00Z",
      "duration_seconds": 3088,
      "reason": "completed"
    }
  ]
}
```

### Schema rules

- `version` is a top-level integer. Currently `1`. Bump only on breaking changes.
- `lanes` is an array. Order is **not significant**; helpers identify lanes by `id`.
- `id` must be unique among active lanes. Convention: `T<N>-<purpose>-<YYYYMMDD-HHMM>`.
- `kind` is a free-form lowercase tag. Recommended values: `audit-scan`, `audit-fix`, `dogfood`, `self-heal`, `scheduler`, `broadcast`, `feature`, `other`. Other lanes from the elevation split (`20260520-2244-elevation-push`) extended this with task-specific kinds like `orchestrator+adr-author` and `proposer-spec-and-broadcast`. **Treat unknown kinds as informational, never reject them.**
- `model` SHOULD be a Claude model ID (e.g. `claude-opus-4-8`, `claude-sonnet-4-6`, `claude-haiku-4-5`). Helpers default to `claude-opus-4-8` if omitted.
- `started_at`, `last_heartbeat`, `estimated_done_at` are ISO-8601 UTC with the `Z` suffix.
- `owns_globs` is an array. Empty array is fine. Globs are advisory hints for humans diagnosing overlap.
- `session_dir`, `terminal_label`, `estimated_done_at` are nullable.

### Backward compatibility

Helpers MUST tolerate minimal entries that only have `id`, `kind`, `started_at`, `owns_globs` (the format T1/T2/T3/T5 used in the elevation split before T4's spec landed). Specifically:

- Missing `model` → display `?` (lane-list.sh) or `unknown` (scheduler snapshots).
- Missing `last_heartbeat` → treat `started_at` as the heartbeat for staleness checks.
- Missing `version` at top level → assume `1`.

Never rewrite or "normalize" foreign entries on heartbeat / unregister. Touch only the row you own.

---

## 2. Helper scripts (`~/.claude/scripts/lane-*.sh`)

All five are atomic via mkdir-based locking (`lane-lock.sh`). macOS has no `flock(1)`, so we use a lock directory at `~/.omc/active-lanes.lock`. Default timeout 15 s; stale locks older than 75 s are force-released.

| Script | Purpose | Atomic |
|---|---|---|
| `lane-register.sh <id> <kind> [globs] [model] [session_dir] [label] [eta]` | Append a lane entry (or replace existing entry with same `id`). Creates the file with the empty skeleton if missing. | yes |
| `lane-heartbeat.sh <id>` | Update `last_heartbeat` to now-UTC. No-op if `id` isn't present. | yes |
| `lane-unregister.sh <id> [reason]` | Remove from `lanes`, append a summary to `completed_today` with computed `duration_seconds`. `reason` defaults to `completed`. | yes |
| `lane-list.sh` | Pretty-print active lanes. Color codes: green `OK`, yellow `STALE` (no heartbeat for >10 min, configurable via `STALE_AFTER`) or `NO-HB` (no `last_heartbeat` field at all). Strips ANSI color codes when `NO_COLOR` env var is set or stdout is not a TTY; status text (`OK`/`STALE`/`NO-HB`) is always emitted regardless of color mode. | read-only |
| `lane-prune-stale.sh` | Move lanes whose `last_heartbeat` is >60 min old (`STALE_AFTER_SEC`, default 3600) into `completed_today` with `reason: "stale"` and `terminated: "stale"`. | yes |

`lane-lock.sh` is sourced by the mutating helpers and is not standalone.

### Calling conventions

- All scripts read `LANES_FILE` env var (default `~/.omc/active-lanes.json`).
- All atomic helpers write via tmp-file + rename.
- Heartbeats are cheap — call from a `while true; do sleep 60; done &` background loop inside long-running lanes if you want to look alive in lane-list.

---

## 3. Observer daemon — `~/.claude/scripts/lane-scheduler.mjs`

Node ESM script, follows the `keyword-detector.mjs` style (top-level imports, no shebang `node` flags needed because PM2 invokes node directly).

Behavior:

1. Every 30 s (configurable via `LANE_TICK_MS`):
2. Read `~/.omc/active-lanes.json` (returns empty skeleton if file missing or unparseable).
3. Compute a snapshot:
   - `active_count`, `stale_count`, `stale_ids`
   - `total_runtime_seconds` (sum of `now - started_at` across all active lanes)
   - `by_kind` distribution, `by_model` distribution
   - `completed_today_count`
4. Append the snapshot as one JSON line to `~/.omc/lane-scheduler/log/<YYYY-MM-DD>.jsonl`.
5. Print to stdout: `[lane-scheduler] active: N | stale: M | by_kind: {dogfood:1, ...}`.

`--once` flag runs a single snapshot and exits — used by tests and the T4 observation sampling.

### What the scheduler does NOT do

- **No** rate limiting.
- **No** lane termination.
- **No** rewriting of `active-lanes.json` (read-only daemon).
- **No** PM2 lifecycle management — Sebastian opts in by running:

  ```bash
  pm2 start /Users/Seb/.claude/scripts/lane-scheduler.pm2.json
  pm2 save
  ```

  T4 does NOT execute this on his behalf.

The PM2 ecosystem file lives at `~/.claude/scripts/lane-scheduler.pm2.json`. Log file: `~/.omc/lane-scheduler/pm2.log`. Memory cap: 100 MB.

---

## 4. Operational guidance

- **Every long-running Claude Code session should register itself** at the start and unregister on clean exit. The 5 elevation-push lanes (T1–T5) all did this; T4 ate its own dog food via `lane-register.sh`.
- If a session crashes without unregistering, the next `lane-prune-stale.sh` invocation (run manually or by a future cron) will sweep it into `completed_today` with `reason: stale`.
- The log file `~/.omc/lane-scheduler/log/<date>.jsonl` is append-only JSONL. Easy to grep, `jq -s`, or feed into a dashboard.
- The scheduler tolerates a wide variety of partial-schema entries. Treat its snapshot output as the source of truth for "what was running at time T".

---

## 5. Future work (Phase 4+) — NOT in this MVP

- Token-bucket quota pacing (see `quota-pacing-design.md`).
- IFleet integration: emit lane events to the IFleet event bus.
- Discord brief on lane completion (via T5's audit-broadcast skill).
- Web dashboard at `~/.omc/lane-scheduler/dashboard/` (not built tonight).
