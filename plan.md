# Plan — splittasks × audit-session × Codex reviewer (elevation)

## Context

**Problem stated by Seb:** splittasks runs 5 parallel terminals great, but every `/audit-session` produces a flood of findings that take so long to fix sequentially that project velocity stalls. He wanted audits to "auto-spawn splittask sessions" so feature work continues.

**What I found after research that reshapes the problem:**

1. **splittasks already has a coordination protocol.** Every MASTER.md contains a "File-overlap matrix" + "Boundaries — do NOT touch" sections. (Reference: `~/.omc/splits/20260520-1700-task-queue-parallel/MASTER.md`.) The issue isn't missing structure — the structure exists per-session but doesn't persist or get read by other sessions.
2. **`/audit-session` (`~/.claude/commands/audit-session.md`) writes NOTHING to disk.** It's a 20-line chat prompt. Findings live in transcript only. This is the keystone gap — without structured findings, no other automation is possible (splittasks can't read them, verifier can't close them, learnings can't fingerprint them).
3. **IFleet already scaffolds Codex.** `~/dev/ai-products/IFleet/src/workers/codex.ts` exists as `WorkerAdapter` (binary='codex'), plus `src/pipeline/reviewer.ts`, `diff-reviewer.ts`, `plan-reviewer.ts`. But Codex CLI is not on PATH and the worker is design-intent, not deployed.
4. **The actual scarce resource is Max-plan quota collisions, not $.** Per `~/.claude/projects/-Users-Seb/memory/feedback_no_budget_caps_claude_max.md` + `IFleet/CLAUDE.md` mandatory rule #1. Codex on flat-rate subscription = separate quota pool = directly addresses this constraint.
5. **The 30+ split sessions in `~/.omc/splits/` show audit and feature work are ALWAYS sequential, never concurrent.** Manual trust-based scheduling. That's the real bottleneck.

**Decisions confirmed with Seb:**
- Codex home: standalone skill now, fold into IFleet later
- Codex role first: per-PR reviewer
- Cost posture: Codex CLI flat-rate subscription
- Sequence: my call → **keystone first, then Codex reviewer**
- Automation posture: **Hybrid** — Conservative default, `audit autopilot` keyword flips to self-driving for that session
- Per-PR review chain: **Tiered** — feature PRs get Codex only (fast). Audit-fix PRs get full Codex + Claude `verifier` subagent sandwich (high-stakes, finding closure must be verified)
- Self-healing (fingerprint → REGRESSION detection → auto-rule generation): **deferred** until after the 2-week soak with Phase 1+2 in production
- Background audit watcher (PM2 daemon): **deferred** — manual scans only in Phase 1

**Subagents are central, not optional.** First draft underused them. Corrected mental model:
- **Subagents** = within-session delegation (audit scanning, triage, review) — share Max quota + share Claude blindspots
- **Codex (external CLI)** = cross-provider second opinion + separate quota pool
- **`.audits/` ledger** = cross-session memory
All three solve different problems; none substitutes for the others. Existing OMC agents (`code-reviewer`, `critic`, `verifier`, `security-reviewer`) get reused — no need to invent.

**Intended outcome:** audit findings become first-class on-disk work items with IDs. splittasks reads them. A new standalone `/codex-review` skill provides a separate-quota review gate on every PR a splittasks lane opens. After 7–14 days of using these in production, fold the working pieces into IFleet (M2 plan-reviewer slot).

---

## Why keystone before Codex (sequence rationale)

| If we ship Codex reviewer first | If we ship keystone first |
|---|---|
| Codex reviews feature PRs that splittasks opens. Useful. | Audits stop vanishing into chat. Findings get IDs. |
| Audit findings still rot in chat. | splittasks can now consume audit findings as work items. |
| When Codex reviews an "audit fix" PR, it has no machine-readable finding to verify against — just commit message. | When Codex reviews an audit-fix PR, the PR cites `AUDIT-<repo>-<hash8>`. Codex re-checks the exact glob. Closes the loop. |
| Solves *quality* of review. Doesn't solve *bottleneck*. | Solves *bottleneck* (findings are dispatchable). Codex adds quality on top. |

Order matters because Codex reviewer earns its keep on audit-fix PRs *only if* findings have IDs. Otherwise it's just "another reviewer."

---

## Recommended approach — three phases

### Phase 1 — Keystone: structured audit findings + subagent chain (~3–4 hours)

Restructure `/audit-session` into two slash commands. Each slash command is a **thin wrapper that invokes a dedicated subagent**. Existing `/audit-session` stays as an alias for backward compat.

**Files to create:**
- `~/.claude/commands/audit-scan.md` — thin wrapper. Invokes the `audit-scanner` subagent (or reuses existing `critic` agent for the first cut). The subagent:
  1. Produces findings in chat as today (for review)
  2. **Also** writes `<repo-root>/.audits/<ISO-timestamp>.json` with the schema below
  3. Updates `<repo-root>/.audits/index.json` (open findings rollup)
  4. Prints finding IDs at end of chat output
- `~/.claude/commands/audit-fix.md` — thin wrapper. Invokes the `audit-triager` subagent (NEW) which:
  1. Reads `.audits/index.json`
  2. Filters by arg (`id|severity|all`)
  3. Groups overlapping file_globs, decides parallel-safe vs sequential
  4. Returns a splittasks-shaped plan that parent renders
- `~/.claude/agents/audit-triager.md` — new subagent definition. Read-only tools + plan generation. Returns JSON plan.

**Reuse don't invent:** for the scanner role, evaluate the existing `critic` agent first (Opus, multi-perspective, all-tools-except-write/edit). If it works, no new agent file needed — just instruct it to also write `.audits/*.json`. If it doesn't fit, create `~/.claude/agents/audit-scanner.md`.

**Files to modify:**
- `~/.claude/commands/audit-session.md` — replace body with: "Run `/audit-scan` then summarize results. Suggest `/audit-fix` if critical findings exist."
- `~/.claude/skills/splittasks/SKILL.md` — add Step 0: read `<repo>/.audits/index.json` if present, surface open critical findings in MASTER.md under a new section "Open audit findings (visible to all lanes)."

**Findings JSON schema (load-bearing — design once, reuse forever):**
```json
{
  "audit_id": "audit-20260520T2300Z",
  "repo": "IFleet",
  "scanned_at": "2026-05-20T23:00:00Z",
  "scanner": "claude-opus-4-7",
  "findings": [
    {
      "id": "AUDIT-IFleet-a1b2c3d4",
      "severity": "CRITICAL|IMPORTANT|COSMETIC",
      "category": "logic|assumption|overconfidence|risk|redo|cosmetic",
      "title": "one-line summary",
      "detail": "2-3 sentence description",
      "file_globs": ["src/pipeline/**", "src/workers/codex.ts"],
      "fix_sketch": "what closing this looks like",
      "parallel_safe": true,
      "fingerprint": "sha256 of category+globs+normalized title",
      "status": "open|fixing|verifying|fixed|closed|regression",
      "opened_at": "2026-05-20T23:00:00Z",
      "closed_at": null,
      "closing_pr": null
    }
  ]
}
```

The `fingerprint` is deterministic — same bug surfacing as a regression gets same fingerprint → enables repeat-finding detection in Phase 3.

**Status lifecycle:**

| Status | Who writes it | Meaning |
|---|---|---|
| `open` | scanner | Finding freshly detected; not yet being worked |
| `fixing` | `/audit-fix` | Lane assigned; PR in flight |
| `verifying` | `/audit-fix` | PR open; awaiting verifier review |
| `fixed` | `/audit-fix` | PR merged; fix is live in the branch |
| `closed` | `/audit-fix` | Recorded in `closed.json` ledger post-merge; fingerprint count updated |
| `regression` | scanner | Finding re-detected after a previous `closed` entry for its fingerprint |

`fixed` → `closed` is a two-step: the `/audit-fix` marks `fixed` at merge time, then appends to `closed.json` and flips to `closed`. Scanners only ever write `open` or `regression`.

> ⚠️ **Critical implementation requirement for `/audit-fix` (Phase 1):** Before any read or write of `closed.json`, `/audit-fix` MUST acquire a mkdir-based lock on `<repo>/.audits/.write-lock`. macOS has no `flock(1)` — use `~/.claude/scripts/lane-lock.sh` (the canonical pattern). Under ≥2 concurrent audit-fix lanes, a race on `closed.json` is deterministic and will cause fingerprint count drift. This is documented in full in `docs/self-heal-pipeline.md` §"How `/audit-fix` appends to `closed.json`" — that document is the contract; this note is the cross-reference so Phase 1 implementers don't miss it. (AUDIT-ae-locking-gap)

**`.gitignore` decision:** `.audits/` ignored by default. Per-repo opt-in by removing the ignore line + adding `.audits/.gitkeep`. Public repos like IFleet: keep ignored (don't leak open issues to the world). Private repos: commit and let teammates see.

### Phase 2 — Codex per-PR reviewer + tiered review chain (standalone skill, ~1 day)

**Blocked prerequisites (Sebastian owns — BLOCKING):**

| Prerequisite | Status | Blocks |
|---|---|---|
| Install Codex CLI (`codex --version` must resolve) | ⛔ NOT MET — Codex CLI not on PATH | All Phase 2 work |
| Confirm flat-rate Codex subscription active | ⛔ NOT MET — unconfirmed | All Phase 2 work |
| Decide auth (Codex CLI config) | ⬜ Pending | Phase 2 start |

**Files to create:**
- `~/.claude/skills/codex-review/SKILL.md` — new skill triggered by `/codex-review <PR#>` and auto-invoked by splittasks strict mode. Wraps `codex exec` with a focused review prompt.
- `~/.claude/skills/codex-review/review-prompt.md` — the per-PR prompt template. Inputs: PR diff, PR title/body, optional `AUDIT-*` finding ID if cited.

**Files to modify:**
- `~/.claude/skills/splittasks/SKILL.md` — strict mode section gets a new **Tiered review chain** subsection:

  | PR type | Review chain | Why |
  |---|---|---|
  | Feature PR (no `AUDIT-*` cited) | Codex only — `/codex-review <PR#>` | Fast (~30–60s), one provider, sufficient for greenfield work |
  | Audit-fix PR (cites `AUDIT-*`) | **Codex + Claude `verifier` subagent in parallel** | Finding closure is high-stakes. Two providers, two angles. Both must PASS to merge. ~60–90s. |

  T1 reads the PR title/body. If it matches `/AUDIT-[A-Za-z0-9_-]+/i` (case-insensitive — repo names in IDs preserve filesystem casing, e.g. `IFleet` vs `factory`), runs the sandwich. Otherwise just Codex. The Claude side uses the existing `verifier` agent from the OMC catalog — no new agent needed.

**Reviewer prompt shape (compact, same for both providers):**
```
You are reviewing PR <NUM> in <repo>. Goal: catch what a Claude-only review missed.

Diff:
<pasted unified diff>

[If audit-fix PR:]
This PR claims to close <AUDIT-id>. Original finding:
<finding JSON>

Verify:
1. Does the diff actually close the finding's intent (not just rename)?
2. Are the listed file_globs all touched?
3. Any new issues introduced?

Output ONE of: PASS | FAIL | NEEDS_REVISION
With 1-paragraph evidence. No preamble.
```

### Phase 2.5 — Hybrid automation layer (~3–4 hours)

Default behavior stays Conservative (you type the commands). One keyword flips a session into Aggressive.

**Files to create:**
- `~/.claude/hooks/audit-banner.sh` — new SessionStart hook. Reads `<cwd>/.audits/index.json` if present. Prints `🩺 Audit: 12 open findings (2 CRITICAL). Type /audit-fix critical or 'audit autopilot' to dispatch.` Nothing more. Suggest-only.
- `~/.claude/skills/audit-autopilot/SKILL.md` — trigger word `audit autopilot`. When invoked: parent Claude reads `.audits/index.json`, invokes `audit-triager` subagent, generates the splittasks plan, renders the paste box. User pastes terminals, that's it.

**Files to modify:**
- `~/.claude/settings.json` — register the new SessionStart hook entry (additive, doesn't disturb existing hooks).
- `~/.claude/hooks/keyword-detector.mjs` — add `"audit autopilot"` to the trigger table mapping to the new skill.

**What's deliberately NOT in Phase 2.5:**
- Stop hook that auto-runs `/audit-scan` at session end — defer to post-soak (risk: hides token cost)
- PR-open hook that auto-fires codex-review — defer to post-soak (risk: noisy on draft PRs)
- Background PM2 watcher — explicitly deferred per Seb decision

### Phase 3 — Use it for 7–14 days, THEN decide what to fold into IFleet

Hard rule: **no IFleet PRs in Phase 3.** Just use the system. Track in a single doc what worked, what didn't:
- Did Codex actually catch things Claude missed? (Count.)
- Did `.audits/index.json` get out of date? (Stale findings?)
- Did splittasks pick up audit findings in MASTER.md correctly?
- Did the audit-fix PRs cite finding IDs as designed?

**After 7–14 days, IFleet integration becomes the next plan, slotted into elevation M2.** Concrete path: `src/workers/codex.ts` becomes the codex executor, `src/pipeline/diff-reviewer.ts` or a new `cross-provider-reviewer.ts` calls it, audit findings flow through the trace event bus as `audit.finding.opened` / `audit.finding.closed` events. Out of scope for *this* plan.

---

## Critical files to be modified or created

| Path | Action | Phase |
|---|---|---|
| `~/.claude/commands/audit-scan.md` | create (thin wrapper, invokes scanner subagent) | 1 |
| `~/.claude/commands/audit-fix.md` | create (thin wrapper, invokes triager subagent) | 1 |
| `~/.claude/commands/audit-session.md` | modify (becomes alias for scan + fix) | 1 |
| `~/.claude/agents/audit-triager.md` | create (NEW subagent) | 1 |
| `~/.claude/agents/audit-scanner.md` | **only create if reusing `critic` agent doesn't fit** — evaluate first | 1 |
| `~/.claude/skills/splittasks/SKILL.md` | modify (Step 0 reads `.audits/index.json` + Tiered review chain in strict mode) | 1 + 2 |
| `~/.claude/skills/codex-review/SKILL.md` | create | 2 |
| `~/.claude/skills/codex-review/review-prompt.md` | create | 2 |
| `~/.claude/hooks/audit-banner.sh` | create (SessionStart, suggest-only) | 2.5 |
| `~/.claude/skills/audit-autopilot/SKILL.md` | create (trigger: `audit autopilot`) | 2.5 |
| `~/.claude/settings.json` | modify (register audit-banner hook) | 2.5 |
| `~/.claude/hooks/keyword-detector.mjs` | modify (add `audit autopilot` trigger) | 2.5 |
| Per-repo `.audits/.gitignore` policy | document in each repo's CLAUDE.md when first audit runs | 1 (operational) |

**Not touched in this plan:**
- IFleet source (`~/dev/ai-products/IFleet/src/**`) — touched in a later plan
- Stop hooks for auto-`/audit-scan` at session end — explicitly deferred to post-soak (risk: hides token cost)
- PR-open hooks for auto-fire codex-review — deferred to post-soak (risk: noisy on draft PRs)
- Lane scheduler daemon (Max-quota pacing) — deferred (Phase 4+)
- Background audit watcher (PM2 hourly diff-scoped) — deferred per Seb decision
- Self-healing: fingerprint REGRESSION + auto-rule generation — deferred until 2-week soak proves which fingerprints actually repeat

---

## Existing utilities to reuse

| Need | Existing implementation | Reuse how |
|---|---|---|
| Split session directory + handoff format | `~/.claude/skills/splittasks/SKILL.md` Step 3 (session id, MASTER.md, T*.md, BEGIN block, file-overlap matrix) | `/audit-fix` generates the same shape, just seeded with findings |
| Boundary declaration in handoff files | splittasks "Boundaries — do NOT touch" section | Each audit-fix lane lists the file_globs from its findings |
| Severity vocabulary | current audit-session.md (`CRITICAL / IMPORTANT / COSMETIC`) | Used verbatim in findings schema |
| Strict-mode per-PR reviewer pattern | splittasks SKILL.md strict mode (lines 174–200) | Audit-fix PRs invoke Codex + Claude `verifier` sandwich; feature PRs invoke Codex only |
| Codex worker scaffolding | `~/dev/ai-products/IFleet/src/workers/codex.ts` | Inspiration for the standalone skill's CLI invocation pattern — NOT imported (skill stays standalone in Phase 2) |
| **Audit scanner role** | Existing `critic` agent (Opus, all tools except Write/Edit) | First-choice scanner. Only create `audit-scanner.md` if `critic` doesn't fit the role. |
| **Audit-fix PR verifier role** | Existing `verifier` agent (OMC catalog) | Used as the Claude side of the Codex+Claude sandwich on audit-fix PRs |
| **Diff-review role** | Existing `code-reviewer` agent (already used by splittasks strict mode) | Stays the default for non-audit PRs |
| Plain-language recap rule | `~/.claude/CLAUDE.md` (Plain-language recap section) | Every audit-fix split's MASTER.md ends with recap |
| Session-start hook scaffolding | `~/.claude/hooks/session-start.mjs` + `remind-audit-session.sh` | Phase 2.5 adds a sibling `audit-banner.sh` that prints open-findings count from `.audits/index.json` |
| Keyword detector hook | `~/.claude/hooks/keyword-detector.mjs` | Phase 2.5 extends with `audit autopilot` → invokes audit-autopilot skill |

---

## Verification

### Phase 1 verification (after writing the three command files)

1. **Smoke `/audit-scan` in a throwaway repo:**
   ```
   cd ~/tmp/audit-test-repo
   git init && echo "function foo(){eval(x)}" > bad.js && git add . && git commit -m init
   /audit-scan
   ```
   Expected: `.audits/<timestamp>.json` exists, contains at least one finding for `bad.js`, severity assigned, `.audits/index.json` updated, finding IDs printed in chat.

2. **Smoke `/audit-fix` referencing one finding ID:**
   ```
   /audit-fix AUDIT-audit-test-repo-XXXXXXXX
   ```
   Expected: a splittasks session dir is created under `~/.omc/splits/<id>/`, MASTER.md includes the finding ID + file globs, each `T*.md` cites `AUDIT-*` in "Done when."

3. **Smoke splittasks Step 0:**
   ```
   /splittasks "add a feature in this repo"
   ```
   Expected: MASTER.md includes a new "Open audit findings (visible to all lanes)" section listing the open finding from step 1, even though the feature work is unrelated.

### Phase 2 verification (after writing the codex-review skill)

1. **Confirm Codex CLI installed and authed:**
   ```
   codex --version
   codex exec "say hi"  # one-shot smoke
   ```

2. **Run `/codex-review` on a real open PR in IFleet:**
   ```
   /codex-review 161
   ```
   Expected: outputs PASS/FAIL/NEEDS_REVISION + 1-paragraph evidence. Latency < 60s.

3. **Tiered review chain — feature PR:**
   - Open a feature PR (no `AUDIT-*` in title/body) from a splittasks lane in strict mode
   - T1 should invoke `/codex-review <PR#>` only (no Claude verifier)
   - Verify total review time < 90s, single PASS/FAIL

4. **Tiered review chain — audit-fix PR (the sandwich):**
   - Pick a real CRITICAL finding in a test repo
   - Run `/audit-fix <id>` in strict mode
   - When one lane opens its PR (title cites `AUDIT-*`), T1 invokes Codex AND Claude `verifier` agent **in parallel**
   - Verify both return PASS before merge; if either FAIL, finding stays open
   - After merge: `.audits/index.json` flips that finding to `closed` with `closing_pr` set

### Phase 2.5 verification (hybrid automation)

1. **SessionStart banner appears:**
   - `cd` into a repo with a non-empty `.audits/index.json`
   - Start a new Claude Code session
   - Expected: banner prints `🩺 Audit: N open findings (X CRITICAL)`. Cwd without `.audits/` → no banner.

2. **`audit autopilot` keyword triggers self-driving mode:**
   - In a session, type `audit autopilot`
   - Expected: parent Claude reads `.audits/index.json`, invokes `audit-triager`, generates a splittasks plan automatically, renders the paste box
   - Verify no other commands had to be typed

### Operational verification (Phase 3, 7–14 days)

Track in a single `~/.omc/audit-elevation-log.md`:
- Total findings opened, by severity
- Median time-to-close per severity
- Codex catches Claude missed (manually flagged)
- Stale findings (`.audits/index.json` says open but actually fixed elsewhere)
- Splittasks sessions that referenced open findings vs. ones that ignored them

**Success criteria to justify Phase 4 (IFleet fold-in):**
- ≥5 audit findings closed end-to-end with Codex gate
- ≥1 Codex catch that Claude missed
- Zero "phantom open" findings after 14 days
- Splittasks MASTER.md surfaced findings correctly in every session

If any criterion fails: don't fold into IFleet. Iterate on the standalone skills first.

---

## Out of scope (explicit)

- Lane scheduler daemon (Max-plan quota pacing) — Phase 4+
- Continuous post-merge audit hook — Phase 4+
- Stop hooks that auto-trigger `/audit-scan` at session end — post-soak decision (risk: hides token cost from user)
- PR-open hooks that auto-fire codex-review on every PR — post-soak decision (risk: noisy on draft PRs)
- Background PM2 audit watcher (hourly diff-scoped) — explicitly deferred per Seb decision
- **Self-healing (the loop that prevents repeat issues)** — fingerprint REGRESSION detection + auto-write `.claude/rules/audit-learnings.md` after 3 closures of same fingerprint. Deferred until the 2-week soak proves which fingerprints actually repeat. Schema field `fingerprint` is shipped in Phase 1 so data accumulates from day one.
- Auto-rule generation with no human gate — decided posture (when self-healing ships), but the *feature itself* is deferred
- Fingerprint-driven auto-rule-generation in IFleet — Phase 4+ (lands with IFleet M4)
- Proposer (nightly Discord-broadcast plan from `.audits/index.json` + ROADMAP.md) — IFleet M5
- IFleet `src/workers/codex.ts` deployment — separate plan after Phase 3 success
- Cross-provider audit (Claude + Codex simultaneous scan) — defer to Phase 4+, requires keystone first

---

## Risks & mitigations

| Risk | Mitigation |
|---|---|
| `.audits/` accidentally committed to public IFleet repo | Default `.gitignore` entry; per-repo opt-in must be explicit |
| Stale findings — fixed in a side commit, `.audits/index.json` still shows open | Phase 4 post-merge hook closes the loop; until then, manual reconciliation acceptable |
| Codex CLI subscription changes pricing / availability | `codex-review` skill abstracts the CLI call to one function — swap provider without touching splittasks |
| Codex flat-rate has rate limits that conflict with high-volume PR runs | Detect in Phase 3 logs; if hit, add a queue in the codex-review skill before Phase 4 |
| Findings schema needs to change after Phase 3 use | `audit_id` is versioned per file; migration script can rewrite old findings. Don't over-engineer schema versioning yet |
| splittasks MASTER.md becomes too long with audit findings | Cap at top 10 critical opens; link to `.audits/index.json` for the rest |

---

## Plain-language recap

The core fix is making audit findings *real objects on disk* instead of free text that disappears into chat. Once that exists, every other improvement (parallel fixing, Codex reviewing, IFleet automating, self-healing) becomes plumbing instead of redesign.

Three things are central — and they each solve a different problem. **Subagents** handle delegation within a session (audit scanning, triage, review) so the parent stays focused. **Codex** (a separate CLI from a different AI provider on its own flat-rate subscription) gives you a second opinion that doesn't share Claude's quota or blindspots. **The `.audits/` ledger** is what makes everything remember between sessions — audit findings persist with IDs, splittasks reads them, PRs cite them, and over time the system can learn which patterns repeat.

The build order is keystone first (about 3–4 hours), then Codex reviewer with tiered chain (about a day), then a thin hybrid automation layer that's conservative by default and flips to self-driving on the keyword `audit autopilot` (about 3–4 hours). Then you use it for 1–2 weeks before deciding which pieces deserve to be built into IFleet permanently. Self-healing (the loop that auto-writes rules so Claude stops repeating the same bug) is explicitly deferred — but the fingerprint data starts accumulating from day one, so when you ship it later, it can mine real history.

Total Phase 1+2+2.5 effort: ~2 working days. Then 2 weeks of just using it.
