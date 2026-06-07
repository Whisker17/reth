# op-reth Auto-Bump Workflow Enhancement Design

> Target file: `.github/workflows/update-op-reth.yml`
> Reference: [Tempo's update-reth.yml](https://github.com/tempoxyz/tempo/blob/ef6926c/.github/workflows/update-reth.yml)

## Background

The current workflow ran a dry run against `op-reth/v2.2.2` and reported `validation_status=failed` despite cargo check passing. Root cause: 6 informational "manual structural conflicts" (e.g. "local deleted, upstream modified; took upstream") block validation even though the workflow already auto-resolved them.

Additionally, the Claude Code retry loop was never triggered because:
1. No `<<<<<<<` markers existed (all merge conflicts were structural, not textual)
2. `cargo check` passed

Comparing with Tempo's workflow: they auto-resolve conflicts via ours/theirs strategy, run 3 independent AI fix loops (clippy, test compilation, CI feedback), and manual conflicts don't block validation.

This design enhances the workflow in 3 independent phases, each shippable on its own.

---

## Phase 1: Manual Conflicts — Warning, Not Blocker

### Problem

`has_manual_conflicts` is set when **any** of these are logged to `manual-conflicts.txt`:

| Conflict type | Currently | Should be |
|---|---|---|
| `upstream deleted, local modified` | Blocker | **Info** — file preserved locally |
| `local deleted, upstream modified; took upstream` | Blocker | **Info** — took upstream version |
| `binary conflict: local and upstream both modified` | Blocker | **Blocker** — needs human decision |
| `merge-file error, exit N` | Blocker | **Blocker** — needs investigation |

The first two categories are already auto-resolved. They should warn, not block.

> **Note**: "upstream deleted, local modified" means the file may be obsolete upstream. Keeping the local version is a safe default, but reviewers must verify it's not dead code. This is why we label it "review recommended" rather than silently passing.

### Design

#### 1a. Split manual conflicts into two files

Replace single `/tmp/manual-conflicts.txt` with:
- `/tmp/manual-conflicts-info.txt` — resolved cases (took upstream, kept local)
- `/tmp/manual-conflicts-unresolved.txt` — truly unresolved (binary conflict, merge-file error)

Each conflict type routes to the appropriate file at its origin point (lines ~278-318).

#### 1b. Replace `has_manual_conflicts` variable

Introduce two new variables:
- `has_unresolved_manual_conflicts` — from `/tmp/manual-conflicts-unresolved.txt`
- `has_manual_info` — from `/tmp/manual-conflicts-info.txt`

Only `has_unresolved_manual_conflicts` participates in `validation_status` determination:

```bash
# Before
if [ "$rev_extraction_failed" = "true" ] || [ "$has_unresolved_conflicts" = "true" ] || [ "$has_manual_conflicts" = "true" ]; then
  validation_status="failed"

# After
if [ "$rev_extraction_failed" = "true" ] || [ "$has_unresolved_conflicts" = "true" ] || [ "$has_unresolved_manual_conflicts" = "true" ]; then
  validation_status="failed"
```

#### 1c. Update conflict counting

```bash
# Before
manual_conflicts="$(wc -l < /tmp/manual-conflicts.txt | tr -d ' ')"
total_conflicts=$((marker_conflicts + manual_conflicts))

# After
manual_info_conflicts="$(wc -l < /tmp/manual-conflicts-info.txt | tr -d ' ')"
manual_unresolved_conflicts="$(wc -l < /tmp/manual-conflicts-unresolved.txt | tr -d ' ')"
total_blocking_conflicts=$((marker_conflicts + manual_unresolved_conflicts))
```

#### 1d. Update `write_conflict_report()` function (line ~172)

Split the single "Manual Conflicts" section into two. When a conflict report is generated for blocking conflicts, include info conflicts in that report as additional review context:

```bash
# Before
if [ -s /tmp/manual-conflicts.txt ]; then
  echo "## Manual Conflicts"
  echo '```'
  cat /tmp/manual-conflicts.txt
  echo '```'
fi

# After
if [ -s /tmp/manual-conflicts-unresolved.txt ]; then
  echo "## Manual Conflicts (requires resolution)"
  echo '```'
  cat /tmp/manual-conflicts-unresolved.txt
  echo '```'
fi
if [ -s /tmp/manual-conflicts-info.txt ]; then
  echo "## Auto-Resolved Structural Changes (review recommended)"
  echo '```'
  cat /tmp/manual-conflicts-info.txt
  echo '```'
fi
```

The `write_conflict_report` function is called (and generates `docs/op-reth-auto-bump-conflicts.md`) only when marker or unresolved manual conflicts exist. Info-only conflicts do NOT trigger conflict report generation; they still appear in the PR body and artifacts.

#### 1e. Update `build_artifacts()` function (line ~186)

Both files are always copied so artifacts always contain the full picture:

```bash
# Before
cp /tmp/manual-conflicts.txt "$ARTIFACT_DIR/manual-conflicts.txt" 2>/dev/null || true

# After
cp /tmp/manual-conflicts-info.txt "$ARTIFACT_DIR/manual-conflicts-info.txt" 2>/dev/null || true
cp /tmp/manual-conflicts-unresolved.txt "$ARTIFACT_DIR/manual-conflicts-unresolved.txt" 2>/dev/null || true
```

#### 1f. Update PR body manual conflicts section (line ~660)

**Both sections always appear in PR body** when their respective files are non-empty. This ensures auto-resolved changes are always visible to reviewers:

```bash
# Before
if [ -s /tmp/manual-conflicts.txt ]; then
  echo "### Manual Conflicts"
  echo '```'
  cat /tmp/manual-conflicts.txt
  echo '```'
fi

# After
if [ -s /tmp/manual-conflicts-unresolved.txt ]; then
  echo "### Manual Conflicts (requires resolution)"
  echo '```'
  cat /tmp/manual-conflicts-unresolved.txt
  echo '```'
fi
if [ -s /tmp/manual-conflicts-info.txt ]; then
  echo "### Auto-Resolved Structural Changes (review recommended)"
  echo '```'
  cat /tmp/manual-conflicts-info.txt
  echo '```'
fi
```

#### 1g. Update warnings section (line ~615)

```bash
# Before
if [ "$has_manual_conflicts" = "true" ]; then
  warnings="${warnings}\n- Manual structural conflicts require human resolution."
fi

# After
if [ "$has_unresolved_manual_conflicts" = "true" ]; then
  warnings="${warnings}\n- Unresolved manual structural conflicts require human resolution."
fi
if [ "$has_manual_info" = "true" ]; then
  warnings="${warnings}\n- Auto-resolved structural conflicts present; review recommended."
fi
```

#### 1h. Update failure_summary (line ~552)

```bash
# Before
if [ "$has_manual_conflicts" = "true" ]; then
  failure_summary="${failure_summary}- manual structural conflicts remain\n"
fi

# After
if [ "$has_unresolved_manual_conflicts" = "true" ]; then
  failure_summary="${failure_summary}- unresolved manual structural conflicts remain\n"
fi
```

#### 1i. Update runbook (docs/runbooks/op-reth-auto-bump.md)

The following sections reference the old `manual-conflicts.txt` and must be updated:
- Artifact table (line ~739): replace `manual-conflicts.txt` with two rows for `manual-conflicts-info.txt` and `manual-conflicts-unresolved.txt`
- "Manual Structural Conflicts" troubleshooting (line ~973): update file reference and describe the info vs unresolved distinction

### Implementation Status

The following changes are already applied (uncommitted):
- [x] 1a. Split files at initialization (line ~91)
- [x] 1b. Replace variables at initialization (line ~100)
- [x] Conflict type routing (lines ~278-318)
- [x] 1c. Conflict counting variables (line ~323)
- [x] 1b. `validation_status` determination (line ~535)
- [x] All `has_manual_conflicts` → `has_unresolved_manual_conflicts` replacements in condition checks
- [x] 1h. failure_summary (line ~552)
- [x] 1g. warnings section (line ~615)

Still remaining:
- [ ] 1d. `write_conflict_report()` function (line ~172)
- [ ] 1e. `build_artifacts()` function (line ~186)
- [ ] 1f. PR body manual conflicts section (line ~660)
- [ ] 1i. Runbook update

---

## Phase 2: Restructure AI Fix Loops

### Problem

Current single loop (lines 394-474):

```
for attempt in 1..MAX_ATTEMPTS:
  if markers exist → Claude fix markers
  elif cargo check fails → Claude fix compilation
  elif cargo check passes → break (done)
```

Issues:
1. Test compilation failures (`cargo test --no-run`) are never caught by the AI loop
2. Marker resolution and compilation fix share a single attempt counter — a marker fix counts against compilation fix budget
3. Claude prompts are passive — they don't instruct Claude to run the validation command itself
4. Single commit at the end makes review harder

### Design

#### 2a. Split into 3 sequential phases

Each phase is an independent loop with its own attempt counter. They share a global deadline but each calculates its remaining budget dynamically.

```
Phase A: Conflict Marker Resolution
  while markers exist && attempts < 5 && time < global_deadline:
    Claude: resolve markers, then run `grep -IRl '<<<<<<<'` to verify
  final recheck → set result
  commit if changes made

Phase B: Compilation Fix
  while cargo check fails && attempts < 8 && time < global_deadline:
    Claude: fix compilation, then run `cargo check --workspace` to verify
  final recheck → set result
  commit if changes made

Phase C: Test Compilation Fix
  while cargo test --no-run fails && attempts < 5 && time < global_deadline:
    Claude: fix test compilation, then run `cargo test --workspace --no-run` to verify
  final recheck → set result
  commit if changes made
```

#### 2b. Per-phase configuration

| Phase | Max Attempts | Validation Command | Commit Message |
|---|---|---|---|
| A: Markers | 5 | `grep -IRl '<<<<<<<'` | `fix: resolve merge conflict markers from op-reth sync` |
| B: Compilation | 8 | `cargo check --workspace` | `fix: resolve compilation errors from op-reth sync` |
| C: Test Build | 5 | `cargo test --workspace --no-run` | `fix: resolve test compilation errors from op-reth sync` |

Total AI deadline stays at 45 min (`AI_DEADLINE_SECONDS=2700`), shared across all phases. Each phase checks `SECONDS > AI_DEADLINE_SECONDS` before each attempt.

#### 2c. Improved Claude prompts

Key improvements over current prompts:

1. **Tell Claude to verify its own fix** — instruct it to run the validation command after making changes
2. **Richer context** — include old tag, new tag, reth rev information
3. **Guard rails** — "Do NOT suppress warnings with `#[allow(...)]`" (matching Tempo)
4. **Phase C regression guard** — tell Claude to also run `cargo check` before `cargo test --no-run` to avoid regressions

Example Phase B prompt:

```
This Rust workspace synced op-reth/ from upstream ${OLD_TAG} to ${NEW_TAG}.
The paradigmxyz/reth rev was ${CURRENT_RETH_REV} -> ${UPSTREAM_RETH_REV}.

cargo check is failing:

${check_output}

Fix compilation errors, then run `cargo check --workspace` to verify your fix.

Rules:
- Fix files in op-reth/ and mantle-reth/ first.
- reth core crates come from paradigmxyz/reth git deps; do not modify their source.
- Do not modify Cargo.toml git URLs, revs, or branch references.
- Do NOT suppress warnings with #[allow(...)] attributes.
- Focus on compilation errors, not warnings.
```

#### 2d. Per-phase commits with explicit paths

Each phase commits independently after its loop completes (if changes were made).

**Important**: Do NOT use `git add -A` — it may stage artifacts, reports, or other unintended files. Use explicit paths:

```bash
AI_COMMIT_PATHS="op-reth/ mantle-reth/ patches/ Cargo.toml Cargo.lock .op-reth-base-tag"

# After each phase:
if ! git diff --quiet HEAD -- $AI_COMMIT_PATHS 2>/dev/null; then
  git add $AI_COMMIT_PATHS
  if [ "$phase_result" = "resolved" ]; then
    git commit -m "fix: resolve <phase> errors from op-reth sync"
  else
    git commit -m "wip: partially resolve <phase> errors from op-reth sync"
  fi
fi
```

This also fixes the opposite bug in the current workflow: follow-up commit detection only checks `op-reth/ mantle-reth/ patches/` but Claude may fix `Cargo.toml`/`Cargo.lock`, causing those changes to be silently dropped.

#### 2e. Track phase results with post-loop final recheck

Add output variables for PR body reporting:

```bash
markers_phase_result="skipped"       # skipped | resolved | partial | failed
compilation_phase_result="skipped"
test_build_phase_result="skipped"
```

**Each phase must run a final validation after the loop exits** to get the true result. Otherwise the last Claude attempt might have fixed things but the loop counter ran out before the next check:

```bash
# After loop exits:
remaining="$(grep -IRl '<<<<<<<' op-reth/ mantle-reth/ patches/ Cargo.toml Cargo.lock 2>/dev/null || true)"
if [ -z "$remaining" ]; then
  markers_phase_result="resolved"
elif [ "$markers_phase_result" != "skipped" ]; then
  markers_phase_result="partial"
fi
```

Same pattern for Phase B (`cargo check`) and Phase C (`cargo test --no-run`).

#### 2f. `--allowedTools` scope

Keep current restricted toolset (no `--dangerously-allow-all`):

```bash
--allowedTools "Bash(cargo *)" "Bash(git diff *)" "Bash(git status *)" "Bash(grep *)" "Bash(rg *)" "Bash(find *)" "Read" "Edit"
```

Notes:
- `Bash(cargo *)` already covers `cargo test`, `cargo check`, etc. — no need for separate `Bash(cargo test *)`.
- Added `Bash(git status *)` and `Bash(rg *)` so Claude can better locate and diagnose issues.

#### 2g. Scope of AI fix vs human responsibility

The AI phases cover **compilation and test compilation** only. The workflow's full validation also includes targeted tests (`cargo test -p ...`) and integration test build. These are post-AI checks — if they fail, the PR stays draft with `needs-manual-migration` for human resolution.

Explicitly: `validation_status=passed` requires all of the following to pass:
1. No unresolved conflict markers (Phase A scope)
2. `cargo check --workspace` (Phase B scope)
3. `cargo test --workspace --no-run` (Phase C scope)
4. Targeted tests (human scope — not AI-fixed)
5. Integration test build (human scope — not AI-fixed)

If targeted tests or integration build fail after AI phases succeed, the PR is created as draft. Runtime test failures (as opposed to compilation failures) are left for human investigation.

### Pseudocode

```bash
AI_COMMIT_PATHS="op-reth/ mantle-reth/ patches/ Cargo.toml Cargo.lock .op-reth-base-tag"

# ── Phase A: Conflict Markers ──
markers_phase_result="skipped"
for attempt in $(seq 1 5); do
  [ "$SECONDS" -gt "$AI_DEADLINE_SECONDS" ] && break
  remaining="$(grep -IRl '<<<<<<<' op-reth/ mantle-reth/ patches/ Cargo.toml Cargo.lock 2>/dev/null || true)"
  [ -z "$remaining" ] && break
  markers_phase_result="partial"
  command -v claude >/dev/null 2>&1 || break
  claude -p "$marker_prompt" \
    --allowedTools "Bash(cargo *)" "Bash(git diff *)" "Bash(git status *)" "Bash(grep *)" "Bash(rg *)" "Bash(find *)" "Read" "Edit" \
    --max-turns 30
done
# Final recheck
remaining="$(grep -IRl '<<<<<<<' op-reth/ mantle-reth/ patches/ Cargo.toml Cargo.lock 2>/dev/null || true)"
if [ -z "$remaining" ]; then
  markers_phase_result="resolved"
fi
# Commit if phase made changes
if ! git diff --quiet HEAD -- $AI_COMMIT_PATHS 2>/dev/null; then
  git add $AI_COMMIT_PATHS
  if [ "$markers_phase_result" = "resolved" ]; then
    git commit -m "fix: resolve merge conflict markers from op-reth sync"
  else
    git commit -m "wip: partially resolve merge conflict markers from op-reth sync"
  fi
fi

# ── Phase B: Compilation ──
compilation_phase_result="skipped"
for attempt in $(seq 1 8); do
  [ "$SECONDS" -gt "$AI_DEADLINE_SECONDS" ] && break
  set +e; cargo check --workspace 2>&1; check_exit=$?; set -e
  [ "$check_exit" -eq 0 ] && break
  compilation_phase_result="partial"
  command -v claude >/dev/null 2>&1 || break
  check_output="$(cargo check --workspace 2>&1 | tail -200)"
  claude -p "$compile_prompt" \
    --allowedTools "Bash(cargo *)" "Bash(git diff *)" "Bash(git status *)" "Bash(grep *)" "Bash(rg *)" "Bash(find *)" "Read" "Edit" \
    --max-turns 30
done
# Final recheck
set +e; cargo check --workspace 2>&1; check_exit=$?; set -e
if [ "$check_exit" -eq 0 ]; then
  compilation_phase_result="resolved"
fi
if ! git diff --quiet HEAD -- $AI_COMMIT_PATHS 2>/dev/null; then
  git add $AI_COMMIT_PATHS
  if [ "$compilation_phase_result" = "resolved" ]; then
    git commit -m "fix: resolve compilation errors from op-reth sync"
  else
    git commit -m "wip: partially resolve compilation errors from op-reth sync"
  fi
fi

# ── Phase C: Test Build ──
test_build_phase_result="skipped"
for attempt in $(seq 1 5); do
  [ "$SECONDS" -gt "$AI_DEADLINE_SECONDS" ] && break
  set +e; cargo test --workspace --no-run 2>&1; test_exit=$?; set -e
  [ "$test_exit" -eq 0 ] && break
  test_build_phase_result="partial"
  command -v claude >/dev/null 2>&1 || break
  test_output="$(cargo test --workspace --no-run 2>&1 | tail -200)"
  claude -p "$test_prompt" \
    --allowedTools "Bash(cargo *)" "Bash(git diff *)" "Bash(git status *)" "Bash(grep *)" "Bash(rg *)" "Bash(find *)" "Read" "Edit" \
    --max-turns 30
done
# Final recheck
set +e; cargo test --workspace --no-run 2>&1; test_exit=$?; set -e
if [ "$test_exit" -eq 0 ]; then
  test_build_phase_result="resolved"
fi
if ! git diff --quiet HEAD -- $AI_COMMIT_PATHS 2>/dev/null; then
  git add $AI_COMMIT_PATHS
  if [ "$test_build_phase_result" = "resolved" ]; then
    git commit -m "fix: resolve test compilation errors from op-reth sync"
  else
    git commit -m "wip: partially resolve test compilation errors from op-reth sync"
  fi
fi
```

---

## Phase 3: CI Feedback Loop (Post-Push)

### Problem

Currently the workflow ends after push + PR creation. If CI fails on the PR, humans must investigate and fix. Tempo's workflow monitors CI and feeds failures back to AI for automated fixing.

### Design

#### 3a. State export and step separation

The CI feedback loop runs as a **separate workflow step** after the main "Sync op-reth and create PR" step. Since shell variables don't carry across steps, the main step must export required state via `$GITHUB_ENV`:

```bash
# At end of main step:
echo "VALIDATION_STATUS=${validation_status}" >> "$GITHUB_ENV"
echo "PR_NUMBER=${pr_number}" >> "$GITHUB_ENV"
echo "OLD_TAG=${OLD_TAG}" >> "$GITHUB_ENV"
echo "NEW_TAG=${NEW_TAG}" >> "$GITHUB_ENV"
echo "CURRENT_RETH_REV=${CURRENT_RETH_REV}" >> "$GITHUB_ENV"
echo "UPSTREAM_RETH_REV=${UPSTREAM_RETH_REV}" >> "$GITHUB_ENV"
echo "CLAUDE_AVAILABLE=$(command -v claude >/dev/null 2>&1 && echo true || echo false)" >> "$GITHUB_ENV"
```

Step guard:

```yaml
- name: CI feedback loop
  if: env.DRY_RUN != 'true' && env.CLAUDE_AVAILABLE == 'true' && env.VALIDATION_STATUS != 'failed' && env.PR_NUMBER != ''
  shell: bash
  run: |
    ...
```

#### 3b. CI poll-fix loop

```
while attempts < 5 && time < deadline:
  wait for CI to finish (poll gh pr checks every 60s)
  if all checks pass → done
  if any check fails or is canceled:
    fetch failed/canceled run logs (bound to PR head SHA)
    Claude: fix the failure
    if Claude made changes:
      commit + push → restart polling
    else:
      give up (prevent infinite loop)
  if checks are stuck:
    fetch in-progress run logs (bound to PR head SHA)
    Claude: investigate and fix likely hang
    if Claude made changes:
      commit + push → restart polling
    else:
      give up
```

#### 3c. Configuration and time budget

Current job `timeout-minutes: 120`. Estimated time budget:

| Stage | Estimated time |
|---|---|
| Checkout + cache + setup | ~2 min |
| Clone upstream + sync | ~5 min |
| AI fix phases (deadline) | 45 min |
| Local validation | ~10 min |
| Push + PR creation | ~1 min |
| **Available for CI loop** | **~57 min** |

CI feedback deadline should be **45 min** (not 60 min), leaving ~12 min buffer before job timeout:

| Parameter | Value | Rationale |
|---|---|---|
| Max fix attempts | 5 | Prevent infinite loops |
| CI poll interval | 60s | Balance responsiveness vs API rate |
| Stuck job timeout | 30 min | Detect hung CI jobs |
| **Total CI deadline** | **45 min** | Fits within 120 min job timeout |

If total time budgets change (e.g. larger runner, longer AI deadline), the CI deadline should be recalculated. Consider raising `timeout-minutes` to 180 if CI loop is frequently cut short.

#### 3d. Stuck job detection

If CI jobs run longer than 30 minutes without completing:
1. Fetch in-progress run logs via `gh api`
2. Grep for `panic|error|FAILED|timed out|deadlock`
3. Feed error context to Claude with investigation prompt
4. If Claude makes changes, push and restart

#### 3e. CI status checking — use `bucket` field

`gh pr checks --json` provides a `bucket` field that normalizes states into `pass`, `fail`, `pending`, `skipping`, `cancel`. Use this instead of raw `state`:

```bash
# Get CI status using bucket field
checks_json="$(gh pr checks "$PR_NUMBER" --repo "$REPO" --json name,bucket,link 2>&1)"

# Handle "no checks reported" (gh outputs non-JSON text)
if ! echo "$checks_json" | jq empty 2>/dev/null; then
  echo "::warning::No CI checks reported yet; waiting..."
  sleep 60
  continue
fi

# Check for terminal failures first
failed_or_canceled="$(echo "$checks_json" | jq -r '.[] | select(.bucket == "fail" or .bucket == "cancel") | .name')"
if [ -n "$failed_or_canceled" ]; then
  break
fi

# Check for pending
pending="$(echo "$checks_json" | jq '[.[] | select(.bucket == "pending")] | length')"
if [ "$pending" -gt 0 ]; then
  continue  # still running
fi

# Success only when no fail/cancel/pending buckets remain.
echo "All CI checks passed"
break
```

#### 3f. CI log extraction — bind to PR head SHA and run status

Don't use bare `gh run view --log-failed` — it's ambiguous. Instead, bind to the current PR head commit and fetch runs matching the relevant terminal/stuck status. Note the spelling difference: `gh pr checks` uses bucket `cancel`, while `gh run list --status` uses `cancelled`.

```bash
head_sha="$(gh pr view "$PR_NUMBER" --repo "$REPO" --json headRefOid --jq '.headRefOid')"

# Find failed/canceled/stuck runs for this exact commit
run_ids=""
for run_status in failure cancelled timed_out startup_failure action_required; do
  ids="$(gh run list \
    --repo "$REPO" \
    --branch "$BRANCH_NAME" \
    --commit "$head_sha" \
    --status "$run_status" \
    --json databaseId \
    --jq '.[].databaseId')"
  run_ids="${run_ids} ${ids}"
done

# Fetch logs for each run
ci_logs=""
for run_id in $run_ids; do
  run_log="$(gh run view "$run_id" --repo "$REPO" --log-failed 2>&1 | tail -300)"
  ci_logs="${ci_logs}\n--- Run ${run_id} ---\n${run_log}\n"
done
```

#### 3g. Claude prompt for CI failures or stalls

```
CI checks failed, were canceled, or stalled on PR #${PR_NUMBER}.
The PR syncs op-reth/ from ${OLD_TAG} to ${NEW_TAG}.
reth rev: ${CURRENT_RETH_REV} -> ${UPSTREAM_RETH_REV}.

CI logs:
${ci_logs}

Fix the failure or likely hang, then run the relevant cargo command to verify your fix.

Rules:
- Only edit op-reth/, mantle-reth/, patches/, Cargo.toml, Cargo.lock.
- Do not modify Cargo.toml git URLs, revs, or branch references.
- Do NOT suppress warnings with #[allow(...)] attributes.
```

#### 3h. Commit and push with explicit paths

After Claude makes changes, use the same explicit commit paths as Phase 2:

```bash
AI_COMMIT_PATHS="op-reth/ mantle-reth/ patches/ Cargo.toml Cargo.lock .op-reth-base-tag"

if ! git diff --quiet HEAD -- $AI_COMMIT_PATHS 2>/dev/null; then
  git add $AI_COMMIT_PATHS
  git commit -m "fix: resolve CI feedback from op-reth sync"
  git push origin "$BRANCH_NAME"
else
  echo "::warning::Claude made no changes; giving up on CI fix"
  break
fi
```

### Pseudocode

```bash
# ── CI Feedback Loop (separate workflow step) ──
AI_COMMIT_PATHS="op-reth/ mantle-reth/ patches/ Cargo.toml Cargo.lock .op-reth-base-tag"
CI_DEADLINE=$((SECONDS + 2700))  # 45 min
CI_MAX_ATTEMPTS=5
REPO="${GITHUB_REPOSITORY}"
ci_attempt=0

while [ "$ci_attempt" -lt "$CI_MAX_ATTEMPTS" ] && [ "$SECONDS" -lt "$CI_DEADLINE" ]; do
  echo "::group::CI feedback — waiting for checks (attempt $((ci_attempt + 1)))"

  # Poll until CI finishes or stuck timeout
  ci_start=$SECONDS
  ci_stuck=false
  checks_json=""
  while [ "$SECONDS" -lt "$CI_DEADLINE" ]; do
    sleep 60

    checks_json="$(gh pr checks "$PR_NUMBER" --repo "$REPO" --json name,bucket,link 2>&1)"

    # Handle non-JSON output (no checks reported yet)
    if ! echo "$checks_json" | jq empty 2>/dev/null; then
      echo "No CI checks reported yet; waiting..."
      continue
    fi

    failed_or_canceled="$(echo "$checks_json" | jq -r '.[] | select(.bucket == "fail" or .bucket == "cancel") | .name')"
    if [ -n "$failed_or_canceled" ]; then
      break
    fi

    pending="$(echo "$checks_json" | jq '[.[] | select(.bucket == "pending")] | length')"
    if [ "$pending" -gt 0 ]; then
      if [ $((SECONDS - ci_start)) -gt 1800 ]; then
        echo "::warning::CI stuck for 30min, attempting investigation"
        ci_stuck=true
        break
      fi
      continue
    fi
    break
  done

  if ! echo "$checks_json" | jq empty 2>/dev/null; then
    echo "::warning::CI checks did not report valid JSON before deadline"
    echo "::endgroup::"
    break
  fi

  # Check results
  failed_checks="$(echo "$checks_json" | jq -r '.[] | select(.bucket == "fail" or .bucket == "cancel") | .name')"
  if [ "$ci_stuck" != "true" ] && [ -z "$failed_checks" ]; then
    echo "All CI checks passed"
    echo "::endgroup::"
    break
  fi

  if [ "$ci_stuck" = "true" ]; then
    stuck_checks="$(echo "$checks_json" | jq -r '[.[] | select(.bucket == "pending") | .name] | join(", ")')"
    echo "Stuck checks: ${stuck_checks}"
  else
    echo "Failed or canceled checks: ${failed_checks}"
  fi

  # Fetch logs bound to PR head SHA
  head_sha="$(gh pr view "$PR_NUMBER" --repo "$REPO" --json headRefOid --jq '.headRefOid')"
  if [ "$ci_stuck" = "true" ]; then
    run_statuses="in_progress"
  else
    run_statuses="failure cancelled timed_out startup_failure action_required"
  fi
  run_ids=""
  for run_status in $run_statuses; do
    ids="$(gh run list \
      --repo "$REPO" \
      --branch "$BRANCH_NAME" \
      --commit "$head_sha" \
      --status "$run_status" \
      --json databaseId \
      --jq '.[].databaseId')"
    run_ids="${run_ids} ${ids}"
  done

  ci_logs=""
  for run_id in $run_ids; do
    if [ "$ci_stuck" = "true" ]; then
      run_log="$(gh run view "$run_id" --repo "$REPO" --log 2>&1 | grep -n -i -E 'panic|error\[|FAILED|timed out|deadlock|stack overflow' -B2 -A5 | head -200 || true)"
      run_tail="$(gh run view "$run_id" --repo "$REPO" --log 2>&1 | tail -200)"
      run_log="${run_log}\n--- last 200 lines ---\n${run_tail}"
    else
      run_log="$(gh run view "$run_id" --repo "$REPO" --log-failed 2>&1 | tail -300)"
    fi
    ci_logs="${ci_logs}\n--- Run ${run_id} ---\n${run_log}\n"
  done

  # Invoke Claude
  ci_fix_prompt="$(cat <<PROMPT_EOF
CI checks failed or stalled on PR #${PR_NUMBER}.
The PR syncs op-reth/ from ${OLD_TAG} to ${NEW_TAG}.
reth rev: ${CURRENT_RETH_REV:-unknown} -> ${UPSTREAM_RETH_REV:-unknown}.

Failed or canceled checks: ${failed_checks:-none}
Stuck checks: ${stuck_checks:-none}

Logs:
${ci_logs}

Fix the failures, then run the relevant cargo command to verify.

Rules:
- Only edit op-reth/, mantle-reth/, patches/, Cargo.toml, Cargo.lock.
- Do not modify Cargo.toml git URLs, revs, or branch references.
- Do NOT suppress warnings with #[allow(...)] attributes.
PROMPT_EOF
  )"

  set +e
  claude -p "$ci_fix_prompt" \
    --allowedTools "Bash(cargo *)" "Bash(git diff *)" "Bash(git status *)" "Bash(grep *)" "Bash(rg *)" "Bash(find *)" "Read" "Edit" \
    --max-turns 30
  set -e

  # Commit and push if Claude made changes
  if git diff --quiet HEAD -- $AI_COMMIT_PATHS 2>/dev/null; then
    echo "::warning::Claude made no changes; giving up on CI fix"
    echo "::endgroup::"
    break
  fi

  git add $AI_COMMIT_PATHS
  git commit -m "fix: resolve CI feedback from op-reth sync (attempt $((ci_attempt + 1)))"
  git push origin "$BRANCH_NAME"
  ci_attempt=$((ci_attempt + 1))
  echo "::endgroup::"
done
```

---

## Implementation Order

```
Phase 1 (manual conflicts) → Phase 2 (AI loops) → Phase 3 (CI feedback)
```

Each phase is independently testable with a dry run against `op-reth/v2.2.2`.

Phase 1 alone fixes the immediate issue (dry run reports `validation_status=passed` for v2.2.2).

## Verification Plan

### Phase 1

1. Dry run against `op-reth/v2.2.2` — expect `validation_status=passed`
2. Verify PR body shows auto-resolved conflicts under "Auto-Resolved Structural Changes (review recommended)"
3. Verify artifacts contain both `manual-conflicts-info.txt` and `manual-conflicts-unresolved.txt`
4. Verify runbook artifact table matches new file names

### Phase 2

1. Test against a version with known compilation breakage (e.g. `op-reth/v2.3.0`)
2. Verify Claude is invoked for each failing phase
3. Verify separate commits per phase (check that only `AI_COMMIT_PATHS` are staged)
4. Verify final recheck sets correct phase result even when loop counter exhausted
5. Verify global deadline stops all phases

### Phase 3

1. Run `dry_run=false` to create a real PR
2. If CI fails, verify Claude is invoked with correct logs (bound to head SHA)
3. Verify the fix-push-poll cycle produces new commits
4. Verify stuck job detection triggers after 30 min
5. Verify total workflow stays within `timeout-minutes: 120`

## Tempo Comparison

| Feature | Current | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|---|
| Manual conflicts block validation | Yes | **No** (only unresolved) | No | No |
| AI fix: conflict markers | Yes | Yes | Yes (dedicated phase) | Yes |
| AI fix: compilation | Yes (cargo check) | Yes | **Yes (dedicated phase)** | Yes |
| AI fix: test compilation | No | No | **Yes (new phase)** | Yes |
| AI fix: CI failures | No | No | No | **Yes** |
| AI fix: stuck jobs | No | No | No | **Yes** |
| Per-phase commits | No | No | **Yes** | Yes |
| Explicit commit paths | No | No | **Yes** | Yes |
| Post-loop final recheck | No | No | **Yes** | N/A |
| Independent phase deadlines | No | No | **Yes** | Yes |
| CI status via `bucket` field | N/A | N/A | N/A | **Yes** |
| CI logs bound to head SHA | N/A | N/A | N/A | **Yes** |

## Out of Scope

- **`cargo clippy` instead of `cargo check`**: Future goal when larger runners are available. Phase 2 is designed so the validation command can be swapped.
- **Targeted test / integration test AI fix**: AI phases only handle compilation. Runtime test failures from `cargo test -p ...` are left for human resolution. Can be added as a Phase D if needed.
- **Cron schedule**: Enable only after multiple successful runs per runbook.
- **`--dangerously-allow-all`**: Keeping restricted `--allowedTools` for security.
- **PR body AI summarization**: Tempo uses AI to summarize upstream changes. Nice-to-have, not critical.
- **Job timeout increase**: Current 120 min may be tight with all 3 phases. Monitor and adjust if needed.
