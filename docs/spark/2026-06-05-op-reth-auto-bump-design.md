# op-reth Auto Bump — Design Spec

> Automated workflow to sync mantle/reth with upstream op-reth releases, using AI-assisted compilation fixing and human-gated PR merging.

## Background

Mantle maintains a downstream build of reth at `mantle-xyz/reth`. The repo has three distinct layers:

| Layer | Location | Source | Tracking mechanism |
|-------|----------|--------|-------------------|
| **reth core** | not in-tree | `paradigmxyz/reth` | ~73 git rev deps in `Cargo.toml` (rev `88505c7f...`) |
| **op-reth** | `op-reth/` | `ethereum-optimism/optimism` `rust/op-reth/` | vendored directory, tag `op-reth/v2.2.1` |
| **mantle-reth** | `mantle-reth/` | Mantle-authored | local path deps |
| **patches** | `patches/` | Mantle-authored | `[patch]` section in root `Cargo.toml` |

**Fork dependencies** (all git, branch-pinned):

| Repo | Branch | Crates |
|------|--------|--------|
| `mantle-xyz/revm` | `mantle-elysium` | revm, revm-bytecode, revm-state, revm-primitives, revm-interpreter, revm-inspector, revm-context, revm-context-interface, revm-database, revm-database-interface, op-revm |
| `mantle-xyz/revm-inspectors` | `mantle-elysium` | revm-inspectors |
| `mantle-xyz/evm` | `mantle-v0.34.0` | alloy-evm |
| `mantlenetworkio/mantle-v2` | `rust/upgrade-develop-20260511` | alloy-op-evm, alloy-op-hardforks, op-alloy, op-alloy-consensus, op-alloy-network, op-alloy-provider, op-alloy-rpc-types, op-alloy-rpc-types-engine, op-alloy-rpc-jsonrpsee |

Additionally, a `[patch.crates-io]` section forces all transitive dependencies to use the Mantle forks (revm, op-revm, revm-inspectors, alloy-evm, alloy-op-evm, alloy-op-hardforks, op-alloy) to prevent duplicate incompatible types.

**Dependency chain:**
```
paradigmxyz/reth (core, pinned by rev)
        ↑
ethereum-optimism/optimism (op-reth, vendored)
        ↑
mantle-xyz/reth (mantle-reth overlay + fork deps)
```

Mantle does **not** independently track `paradigmxyz/reth` latest. Instead, when syncing to a new op-reth tag, the reth rev is copied from upstream op-reth's workspace `Cargo.toml` (`rust/Cargo.toml`), ensuring Mantle stays on the exact reth version that upstream op-reth has validated.

**Reference implementation:** [tempoxyz/tempo `.github/workflows/update-reth.yml`](https://github.com/tempoxyz/tempo/blob/main/.github/workflows/update-reth.yml) — Tempo is also a consumer of reth via git rev deps, so their rev-bump mechanism is directly applicable. However, Tempo does not have a vendored op-reth layer, which is Mantle-specific.

## Local Development Harness

This workflow is developed from the `auto-bump-tools` workspace, with Mantle/reth mounted as a local reference target:

```
/Users/whisker/Work/research/work/auto-bump-tools/
└── Reference/
    └── mantle-reth -> /Users/whisker/Work/src/networks/mantle/reth
```

The symlink is only for local iteration and smoke testing. It lets the workflow script be developed in the tool workspace while operating against a real Mantle/reth checkout. It is not part of the GitHub Actions runtime.

The final workflow file must live inside Mantle/reth:

```
/Users/whisker/Work/src/networks/mantle/reth/.github/workflows/update-op-reth.yml
```

Local scripts may accept a `TARGET_REPO` variable and default it to `Reference/mantle-reth`, but the GitHub Actions implementation should operate on the checked-out repository root directly.

## Scope

**In scope:**
- Detect new `op-reth/vX.Y.Z` tags from `ethereum-optimism/optimism`
- Patch-based sync of `op-reth/` directory (preserving Mantle local modifications)
- Copy upstream op-reth's pinned `paradigmxyz/reth` rev into local `Cargo.toml` (~73 deps)
- Fork dependency compatibility detection (revm, alloy, op-alloy families)
- AI-assisted compilation error fixing via Claude Code CLI
- Lightweight validation gate (cargo check + targeted tests)
- Always push branch and create/update PR (draft on failure, ready on success)
- Lark webhook notification

**Out of scope:**
- Automatic bumping of Mantle fork dependencies (revm, alloy, etc.) — flagged in PR for human review
- Syncing reth core source code (Mantle uses git rev deps, not in-tree fork)
- Post-PR CI monitoring and auto-fix (future enhancement)
- Auto-merge — always requires human review

## Architecture

### Workflow File

`.github/workflows/update-op-reth.yml` — single GitHub Actions workflow.

### Trigger

```yaml
on:
  workflow_dispatch:
    inputs:
      base_branch:
        description: 'Base branch to sync onto'
        required: true
        default: 'mantle-elysium'
        type: string
      target_tag:
        description: 'Optional upstream op-reth tag override for testing, e.g. op-reth/v2.3.0'
        required: false
        type: string
      dry_run:
        description: 'Run sync and validation without pushing or creating/updating PR'
        required: false
        default: true
        type: boolean
```

Phase 1: manual `workflow_dispatch` only with configurable base branch. `target_tag` and `dry_run` are required for safe first-run validation. After stabilization, add cron schedule (e.g., weekly `0 3 * * 1` — Monday 3:00 AM UTC) using the default base branch.

### Runner

Initial implementation uses `ubuntu-latest` (GitHub-hosted standard runner). The workflow relies on GNU coreutils/grep (e.g., `grep -P` for PCRE, `grep -I` for binary skip). Ubuntu runners satisfy this; if porting to macOS or Alpine, replace with `file --mime` or equivalent.

The standard runner is only the MVP execution target. Rust validation for this workspace is resource-intensive, so the final production shape should use a larger runner:

| Stage | Runner | Purpose |
|-------|--------|---------|
| MVP | GitHub-hosted `ubuntu-latest` | Prove branch creation, op-reth sync, reth rev copy, PR body, and draft/ready state transitions |
| Production | 16+ core larger runner, Depot runner, or self-hosted Linux runner | Run full targeted Mantle/op-reth tests and optionally clippy before PR handoff |

The workflow should be written so the runner label is a single variable or small matrix entry, making it easy to move from `ubuntu-latest` to a stronger runner after the logic is proven.

### Concurrency

Group: `update-op-reth`, `cancel-in-progress: false`.

### Job Timeout

120 minutes (Rust compilation + AI fix loops are time-intensive).

### Secrets Required

| Secret | Purpose |
|--------|---------|
| `BOT_GITHUB_TOKEN` | GitHub PAT for push and PR creation (scopes: `repo`, `workflow`) |
| `ANTHROPIC_API_KEY` | API key for Claude sub2api proxy |
| `ANTHROPIC_BASE_URL` | Base URL for Claude sub2api proxy |
| `LARK_WEBHOOK_URL` | Lark group Incoming Webhook URL |

### Base Tag Tracking

File `.op-reth-base-tag` at project root stores the current upstream op-reth tag (e.g., `op-reth/v2.2.1`). This file must be created manually before the first run and is committed to the repository.

---

## Phase 1: Detect Upstream Update

**Goal:** Determine if `ethereum-optimism/optimism` has published a newer op-reth tag than our current base.

### Steps

1. Read current base tag from `.op-reth-base-tag`.
2. Determine the target tag:
   - If `workflow_dispatch.inputs.target_tag` is set, use it as `LATEST_TAG`. This is for deterministic testing against a known upstream tag.
   - Otherwise query latest stable op-reth tag via GitHub API (using `git/refs/tags` for complete results, filtering out pre-releases):
   ```bash
   if [ -n "${TARGET_TAG:-}" ]; then
       LATEST_TAG="$TARGET_TAG"
   else
       LATEST_TAG=$(gh api repos/ethereum-optimism/optimism/git/refs/tags --paginate \
         --jq '[.[].ref | ltrimstr("refs/tags/")
               | select(startswith("op-reth/v"))
               | select(test("-rc|-dev|-pr") | not)]
               | sort_by(ltrimstr("op-reth/v") | split(".") | map(tonumber))
               | last')
   fi
   ```
3. Validate `LATEST_TAG` starts with `op-reth/v` and exists in `ethereum-optimism/optimism`; fail fast before editing files if invalid.
4. Compare:
   - `CURRENT_TAG == LATEST_TAG` → set `has_update=false`, skip to Phase 8 (notify "already up to date").
   - `CURRENT_TAG != LATEST_TAG` → set `has_update=true`, record `OLD_TAG=$CURRENT_TAG`, `NEW_TAG=$LATEST_TAG`, continue.

### Output Variables

- `has_update`: boolean
- `OLD_TAG`: e.g., `op-reth/v2.2.1`
- `NEW_TAG`: e.g., `op-reth/v2.3.0`

---

## Phase 2: Branch Management

**Goal:** Ensure `op-reth-auto-bump` branch is in a clean state based on the configured base branch.

### Logic

```bash
BASE_BRANCH="${{ inputs.base_branch }}"

if git ls-remote --heads origin op-reth-auto-bump | grep -q op-reth-auto-bump; then
    git checkout op-reth-auto-bump

    # Attempt rebase; abort on any conflict and recreate
    if ! git rebase "origin/${BASE_BRANCH}"; then
        git rebase --abort
        echo "::warning::Rebase conflict — recreating branch from ${BASE_BRANCH}"
        git checkout -B op-reth-auto-bump "origin/${BASE_BRANCH}"
    fi
else
    git checkout -b op-reth-auto-bump "origin/${BASE_BRANCH}"
fi

# Record base branch SHA for PR metadata
BASE_SHA=$(git rev-parse "origin/${BASE_BRANCH}")
```

### Rationale

Rebase conflicts indicate the base branch has diverged significantly from prior sync attempts. Rather than attempting complex ours/theirs resolution (where rebase semantics are easy to get wrong), we simply abort and recreate. The trade-off is losing prior AI fixes, but those are likely stale if the base has moved enough to conflict. The full sync runs fresh anyway.

---

## Phase 3: Sync op-reth/ Directory

**Strategy:** Patch-based three-way merge preserving Mantle's local modifications.

### Path Mapping

| Upstream (`ethereum-optimism/optimism`) | Local (`mantle-xyz/reth`) |
|----------------------------------------|--------------------------|
| `rust/op-reth/crates/<name>/` | `op-reth/crates/<name>/` |
| `rust/op-reth/bin/` | `op-reth/bin/` |

Note: upstream may have sub-crates not yet present locally (e.g., `tests/`). New upstream crates are copied in; the AI fix step handles adding them to workspace members if needed.

### Steps

1. **Sparse checkout upstream at both tags:**

   ```bash
   git clone --no-checkout --filter=blob:none \
     https://github.com/ethereum-optimism/optimism.git /tmp/upstream
   cd /tmp/upstream
   git sparse-checkout set rust/op-reth rust/Cargo.toml

   git checkout "$OLD_TAG"
   cp -r rust/op-reth /tmp/op-reth-old
   cp rust/Cargo.toml /tmp/upstream-cargo-old.toml

   git checkout "$NEW_TAG"
   cp -r rust/op-reth /tmp/op-reth-new
   cp rust/Cargo.toml /tmp/upstream-cargo-new.toml
   ```

2. **Per-file three-way merge using `git merge-file`:**

   `git merge-file` performs a true three-way merge on standalone files without requiring blobs in the git object database. This is the correct tool for merging vendored directories where LOCAL has diverged from BASE.

   ```bash
   cd "$GITHUB_WORKSPACE"
   > /tmp/patch-conflicts.txt       # marker-based conflicts (AI can attempt to resolve)
   > /tmp/manual-conflicts.txt      # non-marker conflicts requiring human intervention

   # Enumerate all files across all three versions
   (cd /tmp/op-reth-old && find . -type f) > /tmp/files-old.txt 2>/dev/null || true
   (cd /tmp/op-reth-new && find . -type f) > /tmp/files-new.txt 2>/dev/null || true
   (cd op-reth && find . -type f) > /tmp/files-local.txt 2>/dev/null || true

   # Helper: detect binary files (contains NUL byte)
   is_binary() { head -c 8000 "$1" | grep -qP '\x00'; }

   # Process substitution avoids subshell — variables survive the loop
   while read -r f; do
       LOCAL="op-reth/$f"
       OLD="/tmp/op-reth-old/$f"
       NEW="/tmp/op-reth-new/$f"

       if [ -f "$NEW" ] && [ ! -f "$OLD" ]; then
           # --- New file in upstream: copy in ---
           mkdir -p "$(dirname "$LOCAL")"
           cp "$NEW" "$LOCAL"

       elif [ -f "$OLD" ] && [ ! -f "$NEW" ]; then
           # --- Deleted in upstream ---
           if [ -f "$LOCAL" ]; then
               if diff -q "$LOCAL" "$OLD" > /dev/null 2>&1; then
                   rm "$LOCAL"
               else
                   echo "$LOCAL (upstream deleted, local modified)" >> /tmp/manual-conflicts.txt
               fi
           fi

       elif [ -f "$OLD" ] && [ -f "$NEW" ]; then
           # --- File exists in both tags ---
           if diff -q "$OLD" "$NEW" > /dev/null 2>&1; then
               continue  # No upstream change, skip
           fi

           if [ ! -f "$LOCAL" ]; then
               mkdir -p "$(dirname "$LOCAL")"
               cp "$NEW" "$LOCAL"
               echo "$LOCAL (local deleted, upstream modified — took upstream)" >> /tmp/manual-conflicts.txt
               continue
           fi

           # Binary file handling (e.g., .tar, .bin)
           if is_binary "$OLD" || is_binary "$NEW" || is_binary "$LOCAL"; then
               if diff -q "$LOCAL" "$OLD" > /dev/null 2>&1; then
                   # Local unchanged from OLD → safe to take upstream NEW
                   cp "$NEW" "$LOCAL"
               else
                   # Local also modified → cannot merge binary, log conflict
                   echo "$LOCAL (binary conflict: local and upstream both modified)" >> /tmp/manual-conflicts.txt
               fi
               continue
           fi

           # Text three-way merge: LOCAL ← merge(OLD, NEW)
           # git merge-file returns 0 on clean, >0 on conflicts, <0 on error
           # Temporarily disable set -e: conflict exit codes are expected, not errors
           set +e
           git merge-file "$LOCAL" "$OLD" "$NEW"
           merge_exit=$?
           set -e

           if [ $merge_exit -gt 0 ]; then
               echo "$LOCAL" >> /tmp/patch-conflicts.txt
           elif [ $merge_exit -lt 0 ]; then
               echo "$LOCAL (merge-file error, exit $merge_exit)" >> /tmp/patch-conflicts.txt
           fi
       fi
   done < <(sort -u /tmp/files-old.txt /tmp/files-new.txt /tmp/files-local.txt)

   MARKER_CONFLICTS=$(wc -l < /tmp/patch-conflicts.txt | tr -d ' ')
   MANUAL_CONFLICTS=$(wc -l < /tmp/manual-conflicts.txt | tr -d ' ')
   TOTAL_CONFLICTS=$((MARKER_CONFLICTS + MANUAL_CONFLICTS))
   echo "::notice::Three-way merge complete: $MARKER_CONFLICTS marker conflict(s), $MANUAL_CONFLICTS manual conflict(s)"
   ```

3. **Update base tag:**
   ```bash
   echo "$NEW_TAG" > .op-reth-base-tag
   ```

4. **Commit only if no conflicts of either kind:**
   ```bash
   if [ "$TOTAL_CONFLICTS" -eq 0 ]; then
       git add op-reth/ .op-reth-base-tag
       git commit -m "deps: sync op-reth from $OLD_TAG to $NEW_TAG ($DATE)"
       sync_committed=true
   else
       echo "::warning::$TOTAL_CONFLICTS conflict(s) — deferring commit to after AI fix"
       sync_committed=false
   fi
   ```

   **Two types of conflicts:**
   - **Marker conflicts** (`/tmp/patch-conflicts.txt`): files with `<<<<<<<` markers from `git merge-file`. AI can attempt to resolve these.
   - **Manual conflicts** (`/tmp/manual-conflicts.txt`): structural issues (upstream deleted + local modified, binary conflicts) that cannot be expressed as merge markers. These require human judgment and force a draft PR.

---

## Phase 4: Copy Upstream Pinned Reth Rev

**Goal:** Update the ~73 `paradigmxyz/reth` git rev references in `Cargo.toml` to match what upstream op-reth pins at `$NEW_TAG`.

Mantle does not independently choose a reth version — it follows whatever rev upstream op-reth has validated.

### Steps

1. **Extract reth rev from upstream workspace Cargo.toml with validation:**
   ```bash
   rev_extraction_failed=false

   # || true prevents set -e abort if grep finds no match
   UPSTREAM_RETH_REV=$(grep -m1 'paradigmxyz/reth' /tmp/upstream-cargo-new.toml \
     | grep -oP 'rev = "\K[^"]+' || true)

   if ! echo "$UPSTREAM_RETH_REV" | grep -qP '^[0-9a-f]{40}$'; then
       echo "::error::Failed to extract valid reth rev from upstream Cargo.toml (got: '$UPSTREAM_RETH_REV')"
       rev_extraction_failed=true
   fi
   ```

2. **Read and validate current local rev:**
   ```bash
   CURRENT_RETH_REV=$(grep -m1 'paradigmxyz/reth' Cargo.toml \
     | grep -oP 'rev = "\K[^"]+' || true)

   if ! echo "$CURRENT_RETH_REV" | grep -qP '^[0-9a-f]{40}$'; then
       echo "::error::Failed to extract valid local reth rev (got: '$CURRENT_RETH_REV')"
       rev_extraction_failed=true
   fi

   # Verify all local paradigmxyz/reth revs are consistent
   UNIQUE_REVS=$(grep 'paradigmxyz/reth' Cargo.toml | grep -oP 'rev = "\K[^"]+' | sort -u | wc -l || true)
   if [ "$UNIQUE_REVS" -gt 1 ]; then
       echo "::error::Local Cargo.toml has inconsistent reth revs"
       rev_extraction_failed=true
   fi
   ```

   If `rev_extraction_failed=true`, the workflow skips the sed replacement and forces the final validation status to `failed` regardless of compilation outcome.

3. **Replace all occurrences if different:**
   ```bash
   if [ "$CURRENT_RETH_REV" != "$UPSTREAM_RETH_REV" ] && [ "$rev_extraction_failed" != true ]; then
     sed -i "s|${CURRENT_RETH_REV}|${UPSTREAM_RETH_REV}|g" Cargo.toml
     reth_rev_bumped=true
   else
     reth_rev_bumped=false
   fi
   ```
   Note: the rev appears in every `paradigmxyz/reth` dep line (~73 occurrences). The `[patch."https://github.com/paradigmxyz/reth"]` section only contains path patches (no rev), so it is unaffected. A global find-and-replace on the full 40-char SHA is safe because the hex string is unique within the file.

4. **Regenerate lockfile:**
   ```bash
   lock_update_failed=false
   if [ "$reth_rev_bumped" = true ]; then
     # Full cargo update — necessary because changing a git rev source
     # may cascade into transitive dependency resolution changes.
     # This may update non-reth deps; the diff is visible in the PR.
     if ! cargo update 2>&1; then
         echo "::warning::cargo update failed — lockfile may be stale"
         lock_update_failed=true
     fi
   fi
   ```
   If `cargo update` fails, the failure is recorded and surfaced in the PR body under "Workflow Warnings". The workflow continues — `cargo check` in the validation phase will surface any lockfile issues.

5. **Commit (only if rev changed):**
   ```bash
   if [ "$reth_rev_bumped" = true ]; then
     git add Cargo.toml Cargo.lock
     git commit -m "deps: bump reth rev to ${UPSTREAM_RETH_REV:0:7} (from upstream $NEW_TAG, $DATE)"
   fi
   ```

---

## Phase 5: AI-Assisted Compilation Fix

**Tool:** Claude Code CLI (`claude`)

### Fix Loop

```bash
MAX_ATTEMPTS=8
DEADLINE=$((SECONDS + 2700))   # 45 minutes
compilation_fixed=false

for attempt in $(seq 1 $MAX_ATTEMPTS); do
    if [ $SECONDS -ge $DEADLINE ]; then
        echo "::warning::Deadline exceeded after $attempt attempts"
        break
    fi

    # Step 1: Check for unresolved merge conflict markers
    CONFLICTS=$(grep -IRl '<<<<<<<' op-reth/ mantle-reth/ 2>/dev/null || true)
    if [ -n "$CONFLICTS" ]; then
        CONFLICT_PROMPT="The following files have unresolved merge conflict markers
from syncing upstream op-reth ($OLD_TAG → $NEW_TAG):

$(echo "$CONFLICTS" | head -20)

Please resolve these conflicts. Mantle's local modifications (marked as 'ours')
should generally be preserved unless the upstream change makes them obsolete.
After resolving, run: cargo check --workspace"

        env ANTHROPIC_BASE_URL="${ANTHROPIC_BASE_URL}" \
            ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY}" \
            claude -p "$CONFLICT_PROMPT" \
              --allowedTools "Bash(cargo *)" "Bash(git diff *)" "Bash(grep *)" "Bash(find *)" "Read" "Edit" \
              --max-turns 20
    fi

    # Step 2: Compilation check
    set +e
    cargo check --workspace 2>&1
    check_exit=$?
    set -e

    if [ $check_exit -eq 0 ]; then
        compilation_fixed=true
        break
    fi

    # Step 3: AI fix
    set +e
    CHECK_OUTPUT=$(cargo check --workspace 2>&1 | tail -200)
    set -e

    FIX_PROMPT="This Rust workspace just synced op-reth/ from upstream $OLD_TAG to $NEW_TAG,
and the paradigmxyz/reth rev was bumped from ${CURRENT_RETH_REV:0:7} to ${UPSTREAM_RETH_REV:0:7}.
cargo check is failing with these errors:

$CHECK_OUTPUT

Please fix the compilation errors. Rules:
- Fix files in op-reth/ and mantle-reth/ — these are the op-reth and Mantle overlay crates
- reth core crates come from paradigmxyz/reth via git dep — you cannot modify them
- Do NOT modify Cargo.toml git URLs, revs, or branch references
- Do NOT suppress warnings with #[allow(...)]. Migrate to new APIs instead
- Run cargo check --workspace to verify your fixes
- Focus only on compilation errors, not warnings"

    env ANTHROPIC_BASE_URL="${ANTHROPIC_BASE_URL}" \
        ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY}" \
        claude -p "$FIX_PROMPT" \
          --allowedTools "Bash(cargo *)" "Bash(git diff *)" "Bash(grep *)" "Bash(find *)" "Read" "Edit" \
          --max-turns 30
done
```

### Deferred Sync Commit

If Phase 3 deferred the sync commit due to conflicts, commit now (after AI has resolved them):

```bash
if [ "$sync_committed" != true ]; then
    # Check both marker conflicts and manual (structural) conflicts
    REMAINING_MARKERS=$(grep -IRl '<<<<<<<' op-reth/ 2>/dev/null || true)
    REMAINING_MANUAL=$(wc -l < /tmp/manual-conflicts.txt | tr -d ' ')

    if [ -z "$REMAINING_MARKERS" ] && [ "$REMAINING_MANUAL" -eq 0 ]; then
        git add op-reth/ .op-reth-base-tag
        git commit -m "deps: sync op-reth from $OLD_TAG to $NEW_TAG ($DATE)"
        sync_committed=true
    else
        MARKER_COUNT=$(echo "$REMAINING_MARKERS" | grep -c . 2>/dev/null || echo "0")
        echo "::warning::Sync commit deferred — ${MARKER_COUNT} file(s) with conflict markers, ${REMAINING_MANUAL} manual conflict(s)"
    fi
fi
```

### Fix Commit

If AI made additional source changes beyond conflict resolution:

```bash
# Safety check: never commit files with unresolved conflict markers
REMAINING_MARKERS=$(grep -IRl '<<<<<<<' op-reth/ mantle-reth/ patches/ 2>/dev/null || true)

if [ -n "$REMAINING_MARKERS" ]; then
    echo "::warning::Skipping fix commit — unresolved conflict markers in:"
    echo "$REMAINING_MARKERS"
    has_unresolved_conflicts=true
elif [ -n "$(git diff --name-only op-reth/ mantle-reth/ patches/)" ]; then
    git add op-reth/ mantle-reth/ patches/
    git commit -m "fix: resolve breaking changes from op-reth sync ($DATE)"
    has_unresolved_conflicts=false
else
    has_unresolved_conflicts=false
fi
```

---

## Phase 6: Fork Dependency Compatibility Check

Detects whether the upstream op-reth's dependency versions have drifted from Mantle's forked dependency branches. This check is informational — results are embedded in the PR body for human reviewers.

### Steps

1. **Parse upstream dependency versions** from `/tmp/upstream-cargo-new.toml`:
   ```bash
   # || true on each pipeline: if upstream doesn't declare one of these deps,
   # grep exits non-zero which would abort under set -e
   UPSTREAM_REVM=$(grep -A3 '"https://github.com/bluealloy/revm"' /tmp/upstream-cargo-new.toml \
     | grep -oP '(tag|rev)\s*=\s*"\K[^"]+' | head -1 || true)
   UPSTREAM_ALLOY_EVM=$(grep -A3 'alloy-evm' /tmp/upstream-cargo-new.toml \
     | grep -oP 'version\s*=\s*"\K[^"]+' | head -1 || true)
   UPSTREAM_OP_ALLOY=$(grep -A3 'op-alloy' /tmp/upstream-cargo-new.toml \
     | grep -oP 'version\s*=\s*"\K[^"]+' | head -1 || true)
   ```

2. **Parse Mantle's current fork references** from workspace `Cargo.toml`:
   ```bash
   MANTLE_REVM_REF=$(grep -A2 'mantle-xyz/revm"' Cargo.toml \
     | grep -oP '(branch|tag)\s*=\s*"\K[^"]+' | head -1 || true)
   MANTLE_EVM_REF=$(grep -A2 'mantle-xyz/evm"' Cargo.toml \
     | grep -oP '(branch|tag)\s*=\s*"\K[^"]+' | head -1 || true)
   MANTLE_OP_ALLOY_REF=$(grep -A2 'mantlenetworkio/mantle-v2"' Cargo.toml \
     | grep -oP '(branch|tag)\s*=\s*"\K[^"]+' | head -1 || true)
   ```

3. **Generate compatibility report** → `/tmp/compat-report.txt`:
   ```markdown
   ### Fork Dependency Compatibility

   | Dependency | Upstream version | Mantle fork ref | Source |
   |-----------|-----------------|-----------------|--------|
   | revm | $UPSTREAM_REVM | branch `$MANTLE_REVM_REF` | mantle-xyz/revm |
   | alloy-evm | $UPSTREAM_ALLOY_EVM | branch `$MANTLE_EVM_REF` | mantle-xyz/evm |
   | op-alloy | $UPSTREAM_OP_ALLOY | branch `$MANTLE_OP_ALLOY_REF` | mantlenetworkio/mantle-v2 |

   > ⚠️ If upstream versions have changed significantly, Mantle fork branches
   > may need rebasing before this PR can be merged.
   ```

4. **No commit** — informational only, included in the PR body.

---

## Phase 7: Validation & PR

### Validation

MVP validation is intentionally lightweight enough for a standard GitHub runner:

```bash
# All cargo commands use set +e wrapping because non-zero exits are
# expected failures (not script errors) that we capture into variables.
set +e

# Step 1: Workspace compilation
cargo check --workspace 2>&1
check_passed=$?

# Step 2: Build all test binaries (without running)
cargo test --workspace --no-run 2>&1
test_build_passed=$?

# Step 3: Run targeted tests for synced layers
if [ $test_build_passed -eq 0 ]; then
    cargo test \
      -p reth-optimism-node \
      -p reth-optimism-evm \
      -p reth-optimism-consensus \
      -p mantle-reth-cli \
      --lib --all-features 2>&1
    targeted_tests_passed=$?

    # Build mantle integration tests (expensive, just verify they compile)
    cargo test -p mantle-reth-integration-tests --no-run 2>&1
    integration_build_passed=$?
else
    targeted_tests_passed=1
    integration_build_passed=1
fi

set -e

# Manual conflicts (structural issues) always force failure — they can't be auto-resolved
has_manual_conflicts=false
if [ -s /tmp/manual-conflicts.txt ]; then
    has_manual_conflicts=true
fi

# Aggregate — hard failures from earlier phases override compilation results
if [ "$rev_extraction_failed" = true ] || [ "$has_unresolved_conflicts" = true ] \
   || [ "$has_manual_conflicts" = true ]; then
    validation_status="failed"
elif [ $check_passed -eq 0 ] && [ $test_build_passed -eq 0 ] \
   && [ $targeted_tests_passed -eq 0 ] && [ $integration_build_passed -eq 0 ]; then
    validation_status="passed"
elif [ $check_passed -eq 0 ]; then
    validation_status="partial"
else
    validation_status="failed"
fi
```

Full CI (clippy with `-D warnings`, complete test suite, integration test execution) runs as normal PR checks after the PR is created. The workflow gate is intentionally lightweight to stay within standard runner limits.

### Target Production Validation

Once the workflow is stable and moved to a larger runner, the auto-bump workflow should promote the targeted Mantle/op-reth package tests from post-PR CI into the workflow gate:

```bash
cargo test \
  -p mantle-reth-chainspec \
  -p mantle-reth-rpc-ext \
  -p mantle-reth-eth-api \
  -p mantle-reth-cli \
  -p reth-optimism-consensus \
  -p mantle-reth-integration-tests
```

Optional production gate if runner capacity allows:

```bash
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

The MVP should not block on these heavier checks. It should create a PR with enough context for human review, while normal PR CI or a larger-runner version of this workflow handles the full test surface.

### Push

Always push, regardless of validation status:

```bash
if [ "$DRY_RUN" = "true" ]; then
    echo "::notice::dry_run=true; skipping push and PR creation"
else
    git push origin op-reth-auto-bump --force-with-lease
fi
```

### PR Creation / Update

**Always create/update a PR unless `dry_run=true`.** On failure, create as draft with labels indicating manual intervention needed.

```bash
TITLE="deps: sync op-reth ${OLD_TAG} → ${NEW_TAG} ($(date -u +%Y-%m-%d))"

# Build validation section based on status
case "$validation_status" in
    passed)
        VALIDATION_SECTION="### Validation Results
- ✅ Workspace compilation
- ✅ Test build
- ✅ Targeted op-reth + mantle-reth tests
- ✅ Integration test build"
        ;;
    partial)
        VALIDATION_SECTION="### Validation Results
- ✅ Workspace compilation
- ⚠️ Some tests failed — manual review required"
        ;;
    failed)
        set +e
        LAST_ERRORS=$(cargo check --workspace 2>&1 | tail -100)
        set -e
        VALIDATION_SECTION="### Validation Results
- ❌ Compilation failed (AI fix exhausted $MAX_ATTEMPTS attempts)
- Manual migration required

<details><summary>Last compilation errors</summary>

\`\`\`
${LAST_ERRORS}
\`\`\`

</details>"
        ;;
esac

# Build PR body
cat > /tmp/pr-body.txt << BODY_EOF
## Automated op-reth Sync

Synced \`op-reth/\` from upstream \`${OLD_TAG}\` to \`${NEW_TAG}\`.

**Base branch:** \`${BASE_BRANCH}\` (at \`${BASE_SHA:0:7}\`)

### Upstream op-reth Changes
[Compare: ${OLD_TAG}...${NEW_TAG}](https://github.com/ethereum-optimism/optimism/compare/${OLD_TAG}...${NEW_TAG})

### Reth Core Rev
$(if [ "$reth_rev_bumped" = true ]; then
echo "Bumped from \`${CURRENT_RETH_REV:0:7}\` to \`${UPSTREAM_RETH_REV:0:7}\` ([compare](https://github.com/paradigmxyz/reth/compare/${CURRENT_RETH_REV}...${UPSTREAM_RETH_REV}))"
else
echo "Unchanged (\`${CURRENT_RETH_REV:0:7}\`)"
fi)

$(cat /tmp/compat-report.txt 2>/dev/null || echo "_(fork dependency check skipped)_")

${VALIDATION_SECTION}

$(if [ -s /tmp/patch-conflicts.txt ]; then
echo "### Marker Conflicts (AI-resolvable)"
echo "Files with \`<<<<<<<\` merge conflict markers:"
echo "\`\`\`"
cat /tmp/patch-conflicts.txt
echo "\`\`\`"
fi)

$(if [ -s /tmp/manual-conflicts.txt ]; then
echo "### Manual Conflicts (requires human review)"
echo "Structural conflicts that cannot be auto-merged:"
echo "\`\`\`"
cat /tmp/manual-conflicts.txt
echo "\`\`\`"
fi)

$(# Collect workflow warnings into a section
WARNINGS=""
if [ "$rev_extraction_failed" = true ]; then
  WARNINGS="${WARNINGS}\n- ❌ **Rev extraction failed** — could not parse reth rev from upstream or local Cargo.toml"
fi
if [ "$lock_update_failed" = true ]; then
  WARNINGS="${WARNINGS}\n- ⚠️ **cargo update failed** — Cargo.lock may be stale"
fi
if [ "$has_unresolved_conflicts" = true ]; then
  WARNINGS="${WARNINGS}\n- ❌ **Unresolved conflict markers** — some files still contain \`<<<<<<<\` markers"
fi
if [ "$has_manual_conflicts" = true ]; then
  WARNINGS="${WARNINGS}\n- ❌ **Manual conflicts** — structural conflicts (deleted/binary) require human resolution"
fi
if [ "$sync_committed" != true ]; then
  WARNINGS="${WARNINGS}\n- ⚠️ **Sync commit deferred** — op-reth/ changes not yet committed cleanly"
fi
if [ -n "$WARNINGS" ]; then
  echo "### Workflow Warnings"
  echo -e "$WARNINGS"
fi)

---
🤖 Generated by op-reth-auto-bump workflow | [Run #${GITHUB_RUN_NUMBER}](${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}/actions/runs/${GITHUB_RUN_ID})
BODY_EOF

if [ "$DRY_RUN" = "true" ]; then
    echo "::notice::dry_run=true; skipping PR creation/update"
    cat /tmp/pr-body.txt
    PR_URL=""
    PR_NUMBER=""
    exit 0
fi

# Check for existing PR
EXISTING_PR=$(gh pr list --head op-reth-auto-bump --base "${BASE_BRANCH}" \
  --json number --jq '.[0].number // empty')

if [ -n "$EXISTING_PR" ]; then
    gh pr edit "$EXISTING_PR" --title "$TITLE" --body-file /tmp/pr-body.txt

    # Clean up stale state from prior runs
    if [ "$validation_status" = "passed" ]; then
        gh pr ready "$EXISTING_PR" 2>/dev/null || true
        gh pr edit "$EXISTING_PR" --remove-label "blocked,needs-manual-migration" 2>/dev/null || true
        gh pr edit "$EXISTING_PR" --add-label "A-dependencies" 2>/dev/null || true
    else
        gh pr edit "$EXISTING_PR" --add-label "A-dependencies,needs-manual-migration" 2>/dev/null || true
        # Convert to draft via GraphQL (REST API does not support setting draft on update)
        PR_NODE_ID=$(gh pr view "$EXISTING_PR" --json id --jq '.id')
        gh api graphql -f query='
          mutation($id: ID!) {
            convertPullRequestToDraft(input: {pullRequestId: $id}) {
              pullRequest { isDraft }
            }
          }' -f id="$PR_NODE_ID" 2>/dev/null || true
    fi
else
    if [ "$validation_status" = "passed" ]; then
        gh pr create \
          --base "${BASE_BRANCH}" \
          --head op-reth-auto-bump \
          --title "$TITLE" \
          --body-file /tmp/pr-body.txt \
          --label "A-dependencies"
    else
        gh pr create \
          --base "${BASE_BRANCH}" \
          --head op-reth-auto-bump \
          --title "$TITLE" \
          --body-file /tmp/pr-body.txt \
          --label "A-dependencies,needs-manual-migration" \
          --draft
    fi
fi

PR_URL=$(gh pr view op-reth-auto-bump --json url --jq '.url' 2>/dev/null || echo "")
PR_NUMBER=$(gh pr view op-reth-auto-bump --json number --jq '.number' 2>/dev/null || echo "")
```

---

## Phase 8: Lark Notification

Sends an Interactive Card to Lark via Incoming Webhook. Runs `if: always()` — fires on success, failure, cancellation, and "no update available".

### Message Variants

**Success (validation passed, ready-for-review PR):**
```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "✅ op-reth 自动同步成功" },
      "template": "green"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**op-reth:** OLD_TAG → NEW_TAG\n**reth rev:** OLD_REV → NEW_REV\n✅ 编译通过 | ✅ 测试通过\n**PR:** [#NUMBER](URL) — 待 review"
        }
      }
    ]
  }
}
```

**Needs manual migration (draft PR created):**
```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "⚠️ op-reth 自动同步需要人工介入" },
      "template": "orange"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**op-reth:** OLD_TAG → NEW_TAG\n❌ FAILURE_REASON\n**Draft PR:** [#NUMBER](URL) — 需要人工修复\n**Workflow Run:** [#RUN](URL)"
        }
      }
    ]
  }
}
```

**No update:**
```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "ℹ️ op-reth 已是最新版本" },
      "template": "blue"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "当前版本: CURRENT_TAG\n无需更新"
        }
      }
    ]
  }
}
```

### Implementation

```bash
if [ -n "$LARK_WEBHOOK_URL" ]; then
    curl -sf -X POST "$LARK_WEBHOOK_URL" \
      -H "Content-Type: application/json" \
      -d @/tmp/lark-payload.json || echo "::warning::Lark notification failed"
else
    echo "::warning::LARK_WEBHOOK_URL not set, skipping notification"
fi
```

---

## Commit History on op-reth-auto-bump

Each sync run produces 2-3 commits:

```
deps: sync op-reth from op-reth/v2.2.1 to op-reth/v2.3.0 (2026-06-05)
deps: bump reth rev to a1b2c3d (from upstream op-reth/v2.3.0, 2026-06-05)   # only if rev changed
fix: resolve breaking changes from op-reth sync (2026-06-05)                  # only if AI made fixes
```

---

## Edge Cases

1. **No new tag available:** Phase 1 short-circuits to Phase 8 notification. No branch created, no PR.

2. **Reth rev unchanged between tags:** Phase 4 skips the replacement and its commit. The op-reth directory sync still proceeds.

3. **AI cannot fix all errors within budget:** Branch is pushed and a **draft PR** is created with `needs-manual-migration` label, compilation errors in the PR body, and an orange Lark notification. Human can take over from the branch.

4. **Patch conflicts too complex for AI:** Same as above — draft PR with conflict file list in the body.

5. **op-reth source paths change between tags:** The upstream paths (`rust/op-reth/crates/`, `rust/op-reth/bin/`) are hardcoded. If `ethereum-optimism/optimism` restructures, the workflow needs manual adjustment. A pre-check validates the paths exist before proceeding.

6. **Fork dependency version mismatch:** Detected and reported in PR body as a compatibility table. Not auto-fixed. Human reviewer decides whether to update fork branches before merging.

7. **Rebase conflicts on existing branch:** Immediate abort and branch recreation from base branch. Simple and predictable — no risk of silent data loss.

8. **Concurrent runs:** The concurrency group `update-op-reth` with `cancel-in-progress: false` queues new runs behind the current one.

9. **New upstream sub-crates:** Copied in by the new-file handling step. The AI fix step handles adding them to `Cargo.toml` workspace members if needed.

10. **patches/ directory compatibility:** The `patches/reth-trie-common` local patch overrides `reth-trie-common` via `[patch."https://github.com/paradigmxyz/reth"]` with a path dependency. After a reth rev bump, this local patch source may be incompatible with the new reth version. The AI fix loop or human reviewer must verify patch compatibility.

11. **Rev extraction failure:** If `UPSTREAM_RETH_REV` or `CURRENT_RETH_REV` are empty, non-hex, or locally inconsistent, the rev bump is skipped entirely and the run is marked as failed with a draft PR explaining the extraction error.

12. **cargo update failure:** If `cargo update` fails after rev bump, the failure is recorded (`lock_update_failed=true`) and included in the PR body. The workflow continues — subsequent `cargo check` will surface any lockfile issues.

13. **PR state transitions:** When an existing PR is updated:
    - Prior failure → now passed: `gh pr ready` + remove `blocked`/`needs-manual-migration` labels
    - Prior success → now failed: convert to draft + add `needs-manual-migration`
    - Same state: just update title and body

---

## First-Run Checklist

Before the first workflow run:

1. [ ] Create local development symlink: `auto-bump-tools/Reference/mantle-reth -> /Users/whisker/Work/src/networks/mantle/reth`
2. [ ] Create `.op-reth-base-tag` in repo root with current tag: `op-reth/v2.2.1`
3. [ ] Verify upstream path mapping: `rust/op-reth/{bin,crates}/` → `op-reth/{bin,crates}/`
4. [ ] Add `BOT_GITHUB_TOKEN` secret to repository
5. [ ] Add `ANTHROPIC_API_KEY` secret
6. [ ] Add `ANTHROPIC_BASE_URL` secret
7. [ ] Add `LARK_WEBHOOK_URL` secret
8. [ ] Configure checkout to use `BOT_GITHUB_TOKEN`, `fetch-depth: 0`, and `GH_TOKEN=${{ secrets.BOT_GITHUB_TOKEN }}`
9. [ ] Install Claude Code CLI on the runner (add setup step to workflow)
10. [ ] Ensure labels exist in repo: `A-dependencies`, `needs-manual-migration`
11. [ ] Create workflow file at `.github/workflows/update-op-reth.yml`
12. [ ] Test with `workflow_dispatch`, `base_branch=mantle-elysium`, `dry_run=true`, and a known `target_tag`

---

## Implementation Test Plan

### Local Dry Run

Run this from the `auto-bump-tools` workspace while developing the workflow script:

```bash
ls -la Reference/mantle-reth
cd Reference/mantle-reth
git status --short --branch -uall
```

Required local environment:

| Variable | Value |
|----------|-------|
| `TARGET_REPO` | `/Users/whisker/Work/research/work/auto-bump-tools/Reference/mantle-reth` |
| `BASE_BRANCH` | `mantle-elysium` |
| `TARGET_TAG` | A known newer upstream tag, e.g. `op-reth/v2.3.0` |
| `DRY_RUN` | `true` |
| `ANTHROPIC_API_KEY` | Claude API key or test proxy key |
| `ANTHROPIC_BASE_URL` | Claude sub2api proxy URL |

Local dry-run acceptance checks:

1. The script reads `.op-reth-base-tag` and resolves `OLD_TAG`/`NEW_TAG`.
2. The script fetches `ethereum-optimism/optimism` tag content into `/tmp`.
3. `op-reth/` changes are produced using the per-file three-way merge.
4. The upstream pinned `paradigmxyz/reth` rev is copied from upstream `rust/Cargo.toml` into local `Cargo.toml`.
5. `Cargo.lock` is regenerated or a lockfile warning is reported.
6. `dry_run=true` prevents `git push`, `gh pr create`, and `gh pr edit`.
7. If conflicts remain, the working tree or generated artifact contains enough state for a human to inspect the attempted sync.

After the dry run, inspect:

```bash
git diff -- op-reth/ .op-reth-base-tag Cargo.toml Cargo.lock
git status --short
```

Do not commit dry-run output from the local reference checkout unless intentionally promoting it into a manual test fixture.

### GitHub Actions Smoke Test

After `.github/workflows/update-op-reth.yml` exists in Mantle/reth:

1. Configure repository secrets:
   - `BOT_GITHUB_TOKEN`
   - `ANTHROPIC_API_KEY`
   - `ANTHROPIC_BASE_URL`
   - `LARK_WEBHOOK_URL`
2. Ensure labels exist:
   - `A-dependencies`
   - `needs-manual-migration`
3. Manually trigger `workflow_dispatch` with:
   - `base_branch=mantle-elysium`
   - `dry_run=true`
   - `target_tag=<known newer op-reth tag>`
4. Confirm the run completes without pushing or creating a PR.
5. Re-run with `dry_run=false` on a disposable test branch before enabling it against the shared `op-reth-auto-bump` branch.

Smoke test acceptance checks:

1. The workflow checks out the full repository history (`fetch-depth: 0`).
2. `gh api` can read upstream Optimism tags.
3. `git push` can update `op-reth-auto-bump` when `dry_run=false`.
4. `gh pr create` or `gh pr edit` uses the bot token successfully.
5. Successful validation creates a ready PR with `A-dependencies`.
6. Failed validation creates a draft PR with `A-dependencies` and `needs-manual-migration`.
7. PR body includes upstream tag compare link, reth rev compare link, fork dependency report, validation results, workflow warnings, and conflict sections when applicable.

### Production Readiness Test

Before enabling cron, run one full manual workflow on the production runner shape:

```bash
cargo test \
  -p mantle-reth-chainspec \
  -p mantle-reth-rpc-ext \
  -p mantle-reth-eth-api \
  -p mantle-reth-cli \
  -p reth-optimism-consensus \
  -p mantle-reth-integration-tests
```

If the larger runner is available, also run:

```bash
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

Cron should only be enabled after:

1. Manual `dry_run=true` passes.
2. Manual `dry_run=false` creates a correct PR.
3. A failure-mode run creates a draft PR that a human can actually take over from the pushed branch or attached artifact.
4. Lark notification reports the correct PR URL and validation status.

---

## Future Enhancements

- **Cron schedule:** After stabilization, add `cron: '0 3 * * 1'` (weekly Monday 3 AM UTC).
- **Larger runner:** Switch to 16+ core runner, Depot runner, or self-hosted Linux runner to enable the full targeted Mantle/op-reth package test gate.
- **Full clippy gate:** Add `cargo clippy --workspace --all-targets --all-features -- -D warnings` after the larger runner is proven stable.
- **Post-PR CI monitoring:** Like Tempo, poll CI status after PR creation and use AI to fix failures.
- **Auto-update fork dependencies:** When compilation fails due to fork dependency version mismatch, attempt to bump those automatically.
- **TOML-based fork manifest:** Replace grep-based dependency extraction with a `.op-reth-auto-bump.toml` config that declares fork repos, current refs, and upstream comparison rules — more robust than parsing Cargo.toml with regex.
- **Codex CLI fallback:** Add OpenAI Codex CLI as a secondary AI fix agent if Claude Code fails.
