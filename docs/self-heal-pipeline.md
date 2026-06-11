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
| `audit-rule-drafter.sh` | `~/.claude/scripts/audit-rule-drafter.sh` | T2 | Walk `closed.json`, draft a rule per fingerprint with count >= threshold. |
| `audit-fingerprint-watcher.sh` | `~/.claude/scripts/audit-fingerprint-watcher.sh` | T2 | Cron-friendly wrapper that invokes the drafter across a list of repos. |
| `audit-watched-repos.txt` | `~/.claude/audit-watched-repos.txt` | T2 | Newline-separated repo list. |
| `audit-rejected-gate.sh` | `~/.claude/scripts/audit-rejected-gate.sh` | T3 | Gate: check/register/clear per-repo rejected fingerprints. |
| `audit-prune-stale-closures.sh` | `~/.claude/scripts/audit-prune-stale-closures.sh` | T4 | Move stale closures (globs match zero HEAD files) to `closed.archive.json`. |
| `rejected-fingerprints.json` (data) | `<repo>/.audits/rejected-fingerprints.json` | T3 | Per-repo registry of fingerprints Sebastian has rejected. |
| `closed.archive.json` (data) | `<repo>/.audits/closed.archive.json` | prune script | Archive of stale closures pruned by T4's script. |
| `.audits/proposed-rules/<fp-short>.md` | `<repo>/.audits/proposed-rules/` | drafter | DRAFT rules. Sebastian-only promotion path. |

## Canonical schema — `<repo>/.audits/closed.json`

```json
{
  "repo": "IFleet",
  "version": 1,
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

- `version` — bump only on incompatible schema change.
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

T3 does NOT own `/audit-fix`. This is the contract T2 (or any successor)
implements when closing a finding:

1. When a finding flips status to `verifying` and the verifier passes, on
   merge of the closing PR:
   - Read the finding from `.audits/<scan>.json` (the original scan that
     opened the finding).
   - Recompute its fingerprint via `audit-fingerprinter` (do not trust the
     cached value in `.audits/index.json`).
   - Build the closure record per the schema above.
   - Append to `closures[]` in `<repo>/.audits/closed.json` (initialize file
     with `{"repo": "<name>", "version": 1, "closures": [], "repeat_counts": {}}`
     if absent).
   - Update `repeat_counts[fingerprint].count`, `last_closed_at`, and
     `closures_history`.
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
- adds `status: "open" | "regression" | <dropped>` based on the verdict,
- if `REGRESSION`, also records `regression_of: <original_id>` and
  `regression_closing_pr: <pr>` in the finding.

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
  fingerprints (threshold>=K; skipped X below-threshold, Y already-drafted, Z by filter).`
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

### Flag: `--pre-draft-filter <cmd>`

Optional. Lets an external gate (e.g. T3's `audit-rejected-gate.sh`)
suppress drafts for fingerprints the maintainer has already rejected.

Semantics: for each fingerprint that passes the count-≥-threshold check
and would otherwise be drafted, the drafter pipes the full 64-char
fingerprint hex to `<cmd>` on stdin (before the existing-draft check).

- `<cmd>` exits **0** → drafter proceeds (existing-draft check, then write).
- `<cmd>` exits **1** → drafter SKIPS the fingerprint and increments `skipped_filtered`.
- Any other exit → drafter logs a `WARN` to stderr and treats it as exit 1
  (fail safe — never auto-draft when the gate is misbehaving).

The one-line stdout summary is extended with a fourth bucket (see above).

The semantic spec for the rejected-gate command itself lives in the
"Rejected fingerprints" section below. The drafter only declares the wire.

## `audit-fingerprint-watcher.sh` contract

```
audit-fingerprint-watcher.sh [REPOS_FILE]
                             [--pre-draft-filter <cmd>]
                             [--prune]
```

- Iterates lines of `REPOS_FILE` (default `~/.claude/audit-watched-repos.txt`).
- Skips blank lines and `#`-comment lines. Expands leading `~/` and `$HOME/`.
- For each repo: invokes the drafter; if the drafter exits non-zero, the repo
  is recorded as skipped and the watcher continues.
- Aggregates a stdout summary: `repos_scanned=N repos_skipped=M total_drafts=K`.
- Writes a full per-repo log to `~/.claude/audit-fingerprint-runs/<ISO>.log`
  (path corrected from `~/.omc/...` — the rebuild moved off `~/.omc/`).

### Flag: `--pre-draft-filter <cmd>`

Passthrough to the drafter. Same semantics as the drafter flag (see above).
T3's `audit-rejected-gate.sh check` plugs in here so the gate applies to
every repo the watcher visits.

### Flag: `--prune`

Opt-in. After a successful drafter run, if
`/Users/Seb/.claude/scripts/audit-prune-stale-closures.sh` exists AND is
executable, the watcher invokes it as `audit-prune-stale-closures.sh --apply <repo>`.
If the prune script is absent, the watcher writes `[no prune script — skipping]`
to the log and continues — so T4's script can be missing without breaking the watcher.

Semantic spec for `--prune` lives in the "Closure decay / archival" section below.

### Suggested crontab (NOT installed — Sebastian opts in)

```cron
# 3am daily — check all watched repos for new repeat-fingerprint drafts
0 3 * * * /Users/Seb/.claude/scripts/audit-fingerprint-watcher.sh >/dev/null 2>&1
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

## Rejected fingerprints

When Sebastian rejects a draft (moves it to `.audits/proposed-rules/rejected/` or
deletes it), the system previously forgot the decision. Three more closures of the
same fingerprint would produce the same rejected draft on the next drafter run.

The rejected-fingerprints registry closes this gap.

### Registry schema — `<repo>/.audits/rejected-fingerprints.json`

```json
{
  "repo": "<basename>",
  "version": 1,
  "rejections": [
    {
      "fingerprint": "<64-char lowercase sha256 hex>",
      "rejected_at": "<ISO-8601 UTC>",
      "rejected_by": "Sebastian",
      "reason": "<free text — why this rule shape is not worth a standing rule>",
      "expires_at": null
    }
  ]
}
```

Field rules:
- `expires_at: null` — permanent rejection (the default). Set to an ISO-8601 UTC
  date if Sebastian wants the rejection to lapse (rare).
- `version` — bump only on incompatible schema change.
- `rejections[]` — mutable; entries can be added and removed via the CLI.

### CLI: `audit-rejected-gate.sh`

Path: `~/.claude/scripts/audit-rejected-gate.sh`

```
audit-rejected-gate.sh check    --repo <path>
audit-rejected-gate.sh reject   --repo <path> --fingerprint <hex> --reason "<text>" [--expires <ISO>]
audit-rejected-gate.sh unreject --repo <path> --fingerprint <hex>
audit-rejected-gate.sh list     --repo <path>
```

**`check` (T2 drafter interface — invoked via `--pre-draft-filter`):**
- Reads one fingerprint hex from stdin (whitespace-trimmed).
- If the registry does not exist: exit 0 (no rejections — ok to draft).
- If the registry is malformed JSON: exit 0 with a stderr warning (fail-open;
  worst case is one duplicate draft, which Sebastian re-rejects).
- If the fingerprint is present with `expires_at == null` OR `expires_at > now`: exit 1 (skip).
- Otherwise: exit 0 (ok to draft).

**`reject`:**
- Initializes the registry if absent.
- Refuses to add a duplicate active rejection (warns, exits 0).
- Appends the new rejection via atomic temp+rename.
- Prints: `Rejected fingerprint <fp-short> in <repo> (reason: <reason>)`.

**`unreject`:**
- Removes the matching entry. Prints "not found" if absent, exits 0.

**`list`:**
- Prints a table with columns: `fingerprint(12)`, `rejected_at`, `expires_at`, `reason(50)`.
- Skips expired rejections.

### Design decision: per-repo registry

Per-repo (`<repo>/.audits/rejected-fingerprints.json`) was chosen over a global
`~/.claude/audit-rejected-fingerprints.json`. Rationale:
- Matches the existing pattern: `closed.json` is already per-repo.
- Rejection decisions are context-dependent. The same fingerprint shape could
  be worth a rule in one repo but coincidental in another.
- A global registry carries a higher blast radius: one bad rejection silences
  the fingerprint across every watched repo.
- Per-repo files are version-controlled alongside the code they reference,
  making audit history reproducible from the repo alone.

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
| Fingerprint drifted (algorithm changed) | Same defect produces a different hash → regression detection misses | Bump `version` in `closed.json` and document the algorithm change. Old closures retain their hashes; new closures use the new hash. Do NOT retroactively rewrite hashes. |
| Drafter writes to `.claude/rules/` | Catastrophe — auto-applied rules circumvent review | Scripts explicitly only write to `.audits/proposed-rules/`. If you see a draft in `.claude/rules/` without a `mv` in shell history, the drafter has been modified — `git diff` it. |
| Watcher's repo list points at a non-repo path | `[skip]` in the log; total_drafts unaffected | Fix the path in `~/.claude/audit-watched-repos.txt`. The watcher does not error on missing dirs. |
| 3+ closures coincidentally share a fingerprint with unrelated root causes | Draft proposes an irrelevant rule | Reject the draft. Consider widening the fingerprint algorithm (e.g., include a snippet of the offending code) — this is a known limitation. Document the false positive so future scans treat the fingerprint with skepticism. |
| Closure recorded with `closing_pr: 0` (local-only fix) | Regression-detector returns `REGRESSION:...:0` | Acceptable — surface in the recap. `:0` is a sentinel; the maintainer knows the original fix did not have a PR record. |
| Drafter run while `/audit-fix` is mid-flight | Race on `closed.json` write | Both processes only `append` to `closures[]`; if both write simultaneously, last writer wins and one closure may be lost. Solution: `/audit-fix` should use mkdir-based locking (macOS-compatible; `flock(1)` is not available on macOS) — see `~/.claude/scripts/lane-lock.sh` for the canonical pattern. Drafter is read-only on `closed.json` and is unaffected. |
| Rejected fingerprint registry corrupted (invalid JSON) | `check` logs a warning to stderr and fails open — drafter proceeds; T2 may produce a duplicate draft | Run `jq empty <repo>/.audits/rejected-fingerprints.json` to confirm corruption. Restore from git (`git checkout HEAD -- .audits/rejected-fingerprints.json`) or delete the file and re-enter any known rejections via `audit-rejected-gate.sh reject`. |
| Glob match resolves false-positive zero (e.g., git submodule path; file exists in submodule but `git ls-files` from the parent repo doesn't enumerate it) | Closure archived despite the pattern still being relevant | Restore from `closed.archive.json` by hand (copy the entry back into `closures[]` in `closed.json`); raise `--threshold-days` to give recently-active patterns a wider grace window; or add the repo/submodule path to a `.pruneexclude` list (Tier 2 enhancement). |

## Closure decay / archival

Over time `closures[]` accumulates entries whose `file_globs` no longer match any
file in the repo's HEAD. These stale closures have no active code to protect and
pollute `repeat_counts`, causing the drafter to surface fingerprints for patterns
that no longer exist in the codebase. The prune script keeps the ledger relevant
by moving stale entries to a sibling archive file.

**Script:** `~/.claude/scripts/audit-prune-stale-closures.sh`

```
audit-prune-stale-closures.sh <repo-path> [--apply] [--respect-rejected] [--threshold-days N]
```

**Default (no `--apply`):** dry-run. Prints a table of stale candidates:
`finding_id`, `fingerprint(12)`, `closed_at`, `file_globs`. Exit 0.

**`--apply`:** moves stale closures to `<repo>/.audits/closed.archive.json`.
Rewrites `closed.json` with survivors only and recomputed `repeat_counts`.

**`--threshold-days N`** (default 30): skip closures younger than N days.
Protects recently-closed findings whose associated code may have been renamed
or temporarily deleted mid-flight.

**`--respect-rejected`:** before archiving a stale closure, pipes its fingerprint
to `~/.claude/scripts/audit-rejected-gate.sh check --repo <repo>`. If the gate
exits 1 (active rejection), the closure is kept in `closed.json` rather than
archived. Rationale: Sebastian's explicit rejection of a fingerprint signals that
he considered the pattern meaningful; archiving the closure would strip the context
that informed that rejection decision, and could cause the fingerprint to
re-accumulate toward the draft threshold from a clean slate.

### Stale detection method

Uses `git ls-files | grep -E` with the glob converted to an extended regex:

- `**` → `.*`
- `*` (single-level) → `[^/]*`
- `?` → `[^/]`
- Literal `.` → `\.`

Chosen over `compgen -G` because macOS ships bash 3.2 which does not expand `**`
in `compgen -G`. The `git ls-files` approach is consistent with the repo-tracked
view of the filesystem and avoids matching untracked generated files.

Edge case: a closure with no `file_globs` is skipped — the script cannot determine
staleness without globs and treats the closure as relevant.

### repeat_counts rebuild decision

After archiving, the script **clears `repeat_counts` and recomputes it from scratch**
from the surviving `closures[]`. The doc notes that `repeat_counts` is advisory and
the drafter recomputes for safety anyway. Clearing and recomputing inline is cheaper
than a follow-up drafter run and avoids surfacing inflated counts for fingerprints
whose closures were partially archived.

### Archive file format

`<repo>/.audits/closed.archive.json`:

```json
{
  "repo": "<name>",
  "version": 1,
  "archived_at": "<ISO-8601 UTC of most recent prune>",
  "closures": [ ]
}
```

Writes are atomic (temp file + rename). The archive accumulates across runs;
`archived_at` reflects the most recent prune.

### Integration point with watcher (`--prune`)

The prune script is standalone. T2's `audit-fingerprint-watcher.sh` exposes a
`--prune` flag; when set, the watcher invokes `audit-prune-stale-closures.sh <repo> --apply`
after the drafter completes for each repo. No shared code between T4's script
and T2's watcher.

## Boundaries — T2's promise

T2 owns the resurrection lane (drafter + watcher). T2 never:
- Writes to `~/.claude/rules/`.
- Installs a crontab.
- Modifies `/audit-fix.md` or `/audit-scan.md`.

T2's mutable outputs: `audit-rule-drafter.sh`, `audit-fingerprint-watcher.sh`,
`~/.claude/audit-watched-repos.txt`, and the watcher run logs under
`~/.claude/audit-fingerprint-runs/`.

## Boundaries — T3's promise

T3 owns the rejected-fingerprints gate. T3 never:
- Writes to `~/.claude/rules/`.
- Installs a crontab.
- Modifies `/audit-fix.md`, `/audit-scan.md`, or any T2-owned script.

T3's mutable outputs: `audit-rejected-gate.sh` and per-repo
`<repo>/.audits/rejected-fingerprints.json`.

## Boundaries — T4's promise

T4 owns the closures-decay prune script. T4 never:
- Writes to `~/.claude/rules/` or `closed.json` except via the `--apply` flag.
- Installs a crontab.
- Modifies any T2 or T3-owned script.

T4's mutable outputs: `audit-prune-stale-closures.sh` and (when `--apply` is
passed) `<repo>/.audits/closed.json` and `<repo>/.audits/closed.archive.json`.

## Synthetic proof

A synthetic 3-closure end-to-end test was executed during the T1 orchestration
lane of the `20260601-1525-tier1-curation` split (Tests A, B, C all PASS).
Worker synthetic proofs are at:
- `~/.claude/splits/20260601-1525-tier1-curation/T2-synthetic-proof.md`
- `~/.claude/splits/20260601-1525-tier1-curation/T3-synthetic-proof.md`
- `~/.claude/splits/20260601-1525-tier1-curation/T4-synthetic-proof.md`

Prior proof (archived): `~/.omc/splits/20260520-2244-elevation-push/T3-synthetic-proof.md`
