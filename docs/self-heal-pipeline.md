# Self-heal pipeline

The self-heal pipeline turns audit findings into prevention rules without
auto-applying them. The loop is: find a defect → fix it → record the closure →
when the same defect-shape closes three times, draft a generic rule for human
review. Sebastian opts in by moving the draft to `.claude/rules/`. The pipeline
never writes to `.claude/rules/` itself.

## Why this exists

Point-fixes do not generalize. Without a feedback loop, the same class of bug
gets fixed five times in a row before anyone notices the pattern. The pipeline
makes the pattern visible: every closed finding contributes a deterministic
**fingerprint** (a hash over `category | sorted-globs | normalized-title`);
when the same fingerprint accumulates three closures in `.audits/closed.json`,
`audit-rule-drafter.sh` emits a DRAFT rule pointing at the closures so a human
can decide whether the class of mistake warrants a standing rule.

Three was chosen as the threshold because two closures is plausibly coincidence
and four is too late. The threshold is configurable via `RULE_DRAFT_THRESHOLD`.

## Components

| Component | Path | Owner | Role |
|---|---|---|---|
| `audit-fingerprinter` (subagent) | `~/.claude/agents/audit-fingerprinter.md` | T3 | Compute SHA-256 over the normalized triple. |
| `audit-regression-detector` (subagent) | `~/.claude/agents/audit-regression-detector.md` | T3 | Classify a candidate finding as NEW / REGRESSION / DUPLICATE. |
| `closed.json` (data) | `<repo>/.audits/closed.json` | `/audit-fix` (T2) | Append-only ledger of finding closures. Schema below. |
| `audit-rule-drafter.sh` | `~/.claude/scripts/audit-rule-drafter.sh` | T3 | Walk `closed.json`, draft a rule per fingerprint with count >= threshold. |
| `audit-fingerprint-watcher.sh` | `~/.claude/scripts/audit-fingerprint-watcher.sh` | T3 | Cron-friendly wrapper that invokes the drafter across a list of repos. |
| `audit-watched-repos.txt` | `~/.claude/audit-watched-repos.txt` | T3 | Newline-separated repo list. |
| `.audits/proposed-rules/<fp-short>.md` | `<repo>/.audits/proposed-rules/` | drafter | DRAFT rules. Sebastian-only promotion path. |

## Canonical schema — `<repo>/.audits/closed.json`

```json
{
  "repo": "IFleet",
  "schema_version": 1,
  "closures": [
    {
      "finding_id": "AUDIT-IFleet-a1b2c3d4",
      "fingerprint": "<64-char lowercase sha256 hex>",
      "category": "logic",
      "title": "VerifierAgent has no sandbox isolation",
      "file_globs": ["src/agents/verifier/**"],
      "closed_at": "2026-05-20T23:30:00Z",
      "closing_pr": 161,
      "closing_branch": "fix/verifier-sandbox",
      "fix_summary": "added Docker sandbox per ADR-001"
    }
  ],
  "repeat_counts": {
    "<fingerprint>": {
      "count": 3,
      "last_closed_at": "2026-05-20T23:30:00Z",
      "closures_history": ["AUDIT-...", "AUDIT-...", "AUDIT-..."]
    }
  }
}
```

Field rules:

- `schema_version` — bump only on incompatible schema change. Distinct from `rubric_version` (string semver in rubric.json, tracks the rubric document version); `schema_version` tracks the data format of closed.json itself. Field was renamed from `version` to `schema_version` in the 2026-05-24 nightly audit (AUDIT-ae-d9e0f1a2) for consistency with rubric.json conventions.
- `closures[]` — append-only; never mutate or delete an entry. This is the
  ledger that proves what was fixed when. `closed_at` is ISO-8601 UTC.
- `fingerprint` — comes from `audit-fingerprinter`. Re-deriving it from the
  finding at write time is required; do not trust an unsigned cached value
  passed in from the scanner.
- `closing_pr` — integer GitHub PR number. If the closure landed without a PR
  (rare; only for local-only repos), use `0` and explain in `fix_summary`.
- `closing_branch` — the branch that landed the fix. Useful when the PR is
  squash-merged and the branch is the only trace of the original commits.
- `repeat_counts` — convenience cache; the drafter recomputes from `closures[]`
  for safety, so this is advisory only. `/audit-fix` SHOULD update it for
  cheap lookups but the drafter will still produce correct output if it's stale.

## How `/audit-fix` appends to `closed.json` (contract for T2 / future updates)

> **FORWARD CONTRACT:** This section describes behavior `/audit-fix` MUST implement
> in Phase 4. As of 2026-07-09 (Phase 4 gated on Sebastian decision — see README),
> `closed.json` is **not yet written**. This contract is written prescriptively so
> T2/T4 implementers have a clear, auditable spec.

T3 does NOT own `/audit-fix`. This is the contract T2 (or any successor)
implements when closing a finding:

1. When a finding flips status to `verifying` and the verifier passes, on
   merge of the closing PR:
   - **Acquire a mkdir-based lock** on `<repo>/.audits/.write-lock` before
     reading or writing `closed.json`. Use `~/.claude/scripts/lane-lock.sh`
     (source-able pattern). This MUST happen before any read of `closed.json` to
     prevent concurrent lanes from losing a closure entry.
   - Read the finding from `.audits/<scan>.json` (the original scan that
     opened the finding).
   - Recompute its fingerprint via `audit-fingerprinter` (do not trust the
     cached value in `.audits/index.json`).
   - Build the closure record per the schema above.
   - Append to `closures[]` in `<repo>/.audits/closed.json` (initialize file
     with `{"repo": "<name>", "schema_version": 1, "closures": [], "repeat_counts": {}}`
     if absent).
   - Update `repeat_counts[fingerprint].count`, `last_closed_at`, and
     `closures_history`.
   - Release the lock.
   - Commit the change in the same PR that closes the finding (so the ledger
     and the fix are atomic).

2. On every scan, before opening a new finding, invoke
   `audit-regression-detector` with the candidate. If the verdict is:
   - `NEW` → open the finding normally.
   - `REGRESSION:<original_id>:<closing_pr>` → open the finding with
     `status: "regression"` (new status value); reference the original
     `finding_id` and `closing_pr` in the new finding's metadata so the
     scanner output makes the recurrence visible.
   - `DUPLICATE` → drop the candidate from the new rollup (already-open).

`/audit-fix` and `/audit-scan` are documented in their own files; T3 does not
modify them.

## How regression detection slots into `/audit-scan`

After the scanner produces a candidate finding (but before it writes to
`.audits/<ISO>.json`), it dispatches `audit-regression-detector` with:

- the repo path,
- the candidate finding (with `fingerprint` pre-computed by
  `audit-fingerprinter`).

The detector returns a single verdict line. The scanner then:

- adds `fingerprint` to the finding's persisted JSON regardless,
- adds `status: "open" | "regression" | <dropped>` based on the verdict.
  *(These are the two initial states the scanner writes. The full six-value
  lifecycle — `open → fixing → verifying → fixed → closed | regression` — is
  described in plan.md §schema. The scanner only writes the initial state;
  `/audit-fix` transitions status through the remaining lifecycle states.)*
- if `REGRESSION`, also records `regression_of: <original_id>` and
  `regression_closing_pr: <pr>` in the finding. Parse the verdict string by
  splitting on the first two `':'` characters (split limit 3):
  `parts[0]="REGRESSION"`, `parts[1]=original_id`, `parts[2]=closing_pr_number`.
  The `AUDIT-<repo>-<hash8>` ID format guarantees `original_id` contains no
  colons — this invariant must hold for all future ID formats.

This keeps regression awareness inside the data path; downstream tools
(`/audit-fix`, the drafter, the autopilot skill) can filter on
`status == "regression"` without re-running detection.

## `audit-rule-drafter.sh` contract

```
audit-rule-drafter.sh [REPO_PATH]
```

- Reads `<repo>/.audits/closed.json`.
- For every fingerprint where `count(closures with that fingerprint) >= threshold`
  (default 3, override via `RULE_DRAFT_THRESHOLD`):
  - Picks any closure (they share `file_globs`, `category`, fingerprint by
    construction).
  - Writes a DRAFT file to `<repo>/.audits/proposed-rules/<fp-short>.md` where
    `<fp-short>` is the first 12 hex chars of the fingerprint.
  - If a draft for this fingerprint already exists, the drafter SKIPS it
    (idempotent).
- Prints a one-line summary on stdout: `Drafted N proposed rules from M repeat
  fingerprints (threshold>=K; skipped X below-threshold, Y already-drafted).`
- Exit code:
  - `0` on success (including 0 drafts produced).
  - `2` if `closed.json` is missing or invalid JSON, or the repo path is bad.

The drafted file's frontmatter ALWAYS carries:

```yaml
---
paths: <comma-joined file_globs>
status: DRAFT — Sebastian to review and move to .claude/rules/<name>.md if accepted
fingerprint: <full hex>
repeated: <count>
closures: [<finding_id, ...>]
generated_at: <ISO-8601 UTC>
generator: audit-rule-drafter.sh
category: <category>
seed_title: <one sample title>
---
```

The body explains why the rule was drafted (which closures, fix summaries),
prompts the human to write the actual invariant, and shows the `mv` command
to promote the rule.

## `audit-fingerprint-watcher.sh` contract

```
audit-fingerprint-watcher.sh [WATCHED_REPOS_TXT]
```

- Iterates lines of the file (default `~/.claude/audit-watched-repos.txt`).
- Skips blank lines and `#`-comment lines. Expands `~/...`.
- For each repo: invokes the drafter; if drafter exits non-zero, marks the
  repo as skipped and continues with the next.
- Aggregates a summary: `repos_scanned`, `repos_skipped`, `total_drafts`.
- Writes a full log to `~/.omc/audit-fingerprint-runs/<ISO>.log`.

### Suggested crontab (NOT installed — Sebastian opts in)

```cron
# 3am daily — check all watched repos for new repeat-fingerprint drafts
0 3 * * * ~/.claude/scripts/audit-fingerprint-watcher.sh >/dev/null 2>&1
```

Install manually with `crontab -e`. The watcher is idempotent and side-effect
local (only `.audits/proposed-rules/<repo>/...` and the run log), so accidental
double-runs are harmless.

## DRAFT → ACCEPTED workflow (Sebastian's manual approval step)

```
.audits/proposed-rules/<fp-short>.md   ← drafter writes here
         │
         │  Sebastian reviews on wake (or on demand)
         ▼
~/.claude/rules/<descriptive-name>.md  ← only if accepted; manual mv
```

Sebastian:

1. Reads the draft. Looks at the linked closures (their fix_summary).
2. Decides: is this a class of mistake worth a standing rule, or were the
   closures coincidentally similar?
3. If accept:
   - Edit the draft's body — replace the generic suggested rule with a
     concrete invariant + verification step.
   - `mv .audits/proposed-rules/<fp-short>.md ~/.claude/rules/<name>.md`.
   - Keep the `fingerprint:` frontmatter so future scans can backlink.
4. If reject:
   - Delete the draft, or move it to `.audits/proposed-rules/rejected/` for
     posterity.

The drafter is idempotent on accepted drafts (file moved out) — so when the
fingerprint count hits 4, 5, 6 closures, the drafter does NOT re-draft because
the file is gone from `.audits/proposed-rules/`. To force a re-draft, delete
the accepted rule from `~/.claude/rules/` first (rare).

## Lifecycle diagram

```
       ┌─────────────────┐
       │   /audit-scan   │  finds candidate
       └────────┬────────┘
                │
                ▼
       ┌─────────────────┐
       │ audit-finger-   │  hash = sha256(cat|globs|title)
       │ printer         │
       └────────┬────────┘
                │
                ▼
       ┌─────────────────┐
       │ audit-regression│  consults closed.json + index.json
       │ -detector       │  → NEW | REGRESSION | DUPLICATE
       └────────┬────────┘
                │
                ▼
       ┌─────────────────┐
       │ .audits/        │  finding persisted with fingerprint + status
       │ <ISO>.json      │
       └────────┬────────┘
                │
        (/audit-fix runs;
         fix lands; PR merged)
                │
                ▼
       ┌─────────────────┐
       │ .audits/        │  closure appended (immutable ledger)
       │ closed.json     │
       └────────┬────────┘
                │
        watcher runs (cron or manual)
                │
                ▼
       ┌─────────────────┐
       │ audit-rule-     │  count >= 3? draft a rule.
       │ drafter.sh      │
       └────────┬────────┘
                │
                ▼
       ┌─────────────────┐
       │ .audits/        │  DRAFT (frontmatter + body)
       │ proposed-rules/ │
       └────────┬────────┘
                │
       (Sebastian reviews;
        accepts → mv ~/.claude/rules/<name>.md)
                │
                ▼
       ┌─────────────────┐
       │ ~/.claude/rules │  enforced by future scans + author tooling
       └─────────────────┘
```

## Failure modes & recovery

| Failure | Symptom | Recovery |
|---|---|---|
| `closed.json` corrupted (invalid JSON) | Drafter exits 2 | Restore from git; the ledger is in-repo and version-controlled. Never edit by hand without `jq empty <file>` afterward. |
| Fingerprint drifted (algorithm changed) | Same defect produces a different hash → regression detection misses | Bump `schema_version` in `closed.json` and document the algorithm change. Old closures retain their hashes; new closures use the new hash. Do NOT retroactively rewrite hashes. |
| Drafter writes to `.claude/rules/` | Catastrophe — auto-applied rules circumvent review | Scripts explicitly only write to `.audits/proposed-rules/`. If you see a draft in `.claude/rules/` without a `mv` in shell history, the drafter has been modified — `git diff` it. |
| Watcher's repo list points at a non-repo path | `[skip]` in the log; total_drafts unaffected | Fix the path in `~/.claude/audit-watched-repos.txt`. The watcher does not error on missing dirs. |
| 3+ closures coincidentally share a fingerprint with unrelated root causes | Draft proposes an irrelevant rule | Reject the draft. Consider widening the fingerprint algorithm (e.g., include a snippet of the offending code) — this is a known limitation. Document the false positive so future scans treat the fingerprint with skepticism. |
| Closure recorded with `closing_pr: 0` (local-only fix) | Regression-detector returns `REGRESSION:...:0` | Acceptable — surface in the recap. `:0` is a sentinel; the maintainer knows the original fix did not have a PR record. |
| Drafter run while `/audit-fix` is mid-flight | Race on `closed.json` write | Both processes only `append` to `closures[]`; if both write simultaneously, last writer wins and one closure may be lost. **`/audit-fix` MUST use mkdir-based locking** (macOS-compatible; `flock(1)` is not available on macOS) — see `~/.claude/scripts/lane-lock.sh` for the canonical pattern. This is a hard requirement, not optional: under ≥2 concurrent lanes a race is deterministic and will cause fingerprint count drift. Drafter is read-only on `closed.json` and needs no lock. |

## Boundaries — T3's promise

- T3 never writes to `~/.claude/rules/`.
- T3 never installs a crontab.
- T3 never modifies `/audit-fix.md` or `/audit-scan.md` (T2 owns those; T3
  documents the contract).
- T3's only mutable outputs are: the two subagent files, the two scripts, this
  doc, `~/.claude/audit-watched-repos.txt`, and the watcher's run logs under
  `~/.omc/audit-fingerprint-runs/`.

## Synthetic proof

A synthetic 3-closure scenario was executed during the T3 lane of the
`20260520-2244-elevation-push` split. The captured draft is at
`~/.omc/splits/20260520-2244-elevation-push/T3-synthetic-proof.md`.
