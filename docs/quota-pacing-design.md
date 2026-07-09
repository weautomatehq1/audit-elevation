# Quota Pacing — Design Doc (Phase 4+)

**Status:** DESIGN ONLY — no IFleet code shipped with the 2026-05-20 overnight elevation push (`20260520-2244-elevation-push`). T4 shipped the observation-only lane scheduler (`lane-scheduler-spec.md`) on 2026-05-21. This doc is what to build on top once we have telemetry to calibrate against. Phase 3 soak completed 2026-06-04; Phase 4 implementation is gated on Sebastian's go/no-go decision (see README — 35 days overdue as of 2026-07-09).

**Author:** T4 (lane-scheduler MVP), 2026-05-20.

---

## 1. The problem

Sebastian is on **a single-seat Claude Max plan** (flat-rate, not API-billed — see `~/.claude/CLAUDE.md` "No Budget Caps — Claude Max plan"). The flat rate has no per-token charge, but it does have **session-window rate limits** that throttle concurrent Opus calls.

Tonight's elevation push runs **5 concurrent Opus-4.7 terminals** (T1–T5). With deeper splits (`splittasks` can spawn up to 15 lanes), we will routinely hit the per-window cap. When we do, lanes 429 — the lane either retries, waits, or aborts mid-task. Worst case: a 5-lane split degenerates into "Lane 1 lands, lanes 2–5 all crash on Anthropic-side throttling and lose their per-lane plan."

The scarce resource is **lanes (concurrent Opus capacity)**, not dollars.

---

## 2. Today's mitigation (folklore)

- **Manual 30-second stagger.** Sebastian pastes terminal commands one at a time, ~30 s apart. The hope is that lane N reaches its first tool call after lane N-1 has finished its opening burst.
- **All-Opus discipline.** No mixing Sonnet/Haiku into the same split unless explicit, because routing decisions become opaque under throttling.
- **No quota check.** No code anywhere inspects the response headers Anthropic returns on 429 (`anthropic-ratelimit-*`). We discover throttling only when a lane visibly stalls.

This works for 3–4 lanes most of the time. It does not scale to 10+. It also gives Sebastian no signal about which lane is most expensive in token-burn terms — he just sees them all running.

---

## 3. Proposed v1 (Phase 4+, NOT tonight)

A **token-bucket scheduler** layered on top of the existing observation daemon.

### 3.1 Token bucket

- The scheduler maintains an in-memory virtual bucket sized to "approximately one Opus session-window of effective concurrent work."
- Each lane has a **weight** representing its expected burst rate:

  | Model | Weight |
  |---|---|
  | claude-opus-4-8 | 3 |
  | claude-sonnet-4-6 | 1 |
  | claude-haiku-4-5 | 0.3 |
  | unknown | 2 (conservative) |

- Total weight across active lanes is the "concurrent cost."
- Configurable bucket capacity (initial guess: 9 — i.e. ~3 Opus lanes comfortably).
- **Implementation note:** Model IDs in this table must match exactly what the Claude CLI/API returns. Use prefix matching (e.g. `model.startsWith('claude-opus')`) rather than exact match to handle minor version suffixes (e.g. `claude-opus-4-8` vs `claude-opus-4`) gracefully. Unknown suffixes should fall through to the `unknown` weight (2), not error.

### 3.2 Lane lifecycle changes

A lane that opts into pacing calls a new helper **before** starting work:

```bash
~/.claude/scripts/lane-request-start.sh <id> <kind> <model>
# stdout: "go"        → proceed immediately
# stdout: "wait 12"   → sleep 12s and retry
```

The scheduler holds the bucket. On `request-start`:

- If `current_weight + lane_weight ≤ capacity` → emit `go`, reserve weight.
- Otherwise → emit `wait <N>` with N = estimated seconds until enough lanes finish (use median lane duration from `completed_today`).

`lane-unregister.sh` already exists and would naturally return reserved weight to the bucket.

### 3.3 Reactive 429 handling

When any lane reports a 429 to the scheduler (new helper: `lane-report-429.sh <id> [retry-after-seconds]`), the scheduler:

1. Doubles the wait interval offered to all `request-start` calls for the next 5 minutes.
2. Marks the offending lane with a "back-off" flag in `active-lanes.json`.
3. Logs the event to `~/.omc/lane-scheduler/log/<date>.jsonl` with `event: "429"`.

This is purely additive observability — the lane decides whether to actually back off.

---

## 4. Risks and mitigations

- **False starves.** If lane weights are mis-calibrated (e.g. Opus weight=3 when it should be 2), big lanes may never be admitted under multi-lane load. Mitigation: an **emergency override flag** `LANE_BYPASS_SCHEDULER=1` env var that makes `request-start` always return `go`. Sebastian sets it when he knows what he's doing.
- **Stale reservations.** A lane that gets `go` and then crashes without `unregister` will hold weight until the next `lane-prune-stale.sh` sweep. Mitigation: `lane-prune-stale.sh` is already idempotent; we run it in the same daemon tick if `--prune` flag is set.
- **Coordination overhead.** Each `request-start` call costs a file lock + jq parse. Mitigation: cap at ~1 lock/sec/lane (lanes don't ask 50 times in a row).
- **Misreporting drift.** A lane that lies about its model (claims sonnet, runs opus) gets under-counted. Mitigation: post-hoc audit — completed_today rows include `model`, easy to grep for outliers.

---

## 5. Decision points for Sebastian

Before any of this gets built, Sebastian needs to answer:

1. **What weights?** Opus=3, Sonnet=1, Haiku=0.3 is a starting guess. With a week of telemetry from the observation MVP, we can compute actual burst rates per model and replace these guesses.
2. **What capacity?** 9 (3 Opus comfortably) is conservative. We could go to 12 if telemetry shows 5 Opus lanes routinely surviving without 429s.
3. **What's the refill rate?** Token-bucket implies tokens regenerate over time. For our usage, "bucket fills on `unregister`" is simpler than time-based refill. Start with that; revisit if needed.
4. **Hard cap or soft warning?** Two modes:
   - **Hard cap:** scheduler refuses `request-start` if capacity exceeded. Lanes block until weight frees up.
   - **Soft warning:** scheduler always returns `go`, but logs a "would have throttled" event. Sebastian sees the data; nothing actually slows down.

   Sebastian's call. **Default recommendation: soft warning for the first 2 weeks**, hard cap only after we've validated weights are right.

5. **Per-lane priority?** A `--priority high` flag lets critical lanes (e.g. self-healing reactions) skip the queue. Useful for the morning Proposer running while another lane is mid-feature. Worth adding from day one.

---

## 6. Out of scope for v1

- Cross-machine coordination (Sebastian only runs Claude Code on one laptop).
- Multi-user (single-seat Max plan).
- Auto-tuning weights from telemetry. v1 is hand-calibrated; auto-tuning is v2.
- Integration with the eventual Sentry alarm (lane 429 → Sentry breadcrumb) — possible Phase 5.

---

## 7. Path from MVP to v1

1. **Week 1–2 of MVP:** observation only, gather `<date>.jsonl` snapshots. Look at peak `total_runtime_seconds`, peak `active_count`, kind distribution.
2. **Calibrate weights:** grep `completed_today` for Opus rows, compute median `duration_seconds` and concurrent overlap. Refine the table in §3.1.
3. **Ship soft-warning mode.** Add `lane-request-start.sh` returning `go` always, logging "would-throttle" events.
4. **Validate.** Sebastian compares "would-throttle" log against actual 429s observed during the same window.
5. **Flip to hard cap** once §4 confirms the model is right.

Tonight gets us to step 1.
