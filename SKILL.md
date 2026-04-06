---
name: codex-executor
description: "Use when you want to delegate coding implementation to Codex CLI while keeping Opus for planning and review, reducing Opus token spend by 70-85%."
version: 1.0.0
requires:
  - codex exec (codex-cli >= 0.118.0)
  - git (for diff-based review)
---

# Codex Executor

Hybrid workflow: **Opus plans and reviews, Codex CLI executes.**
Keeps Opus token spend minimal while leveraging Codex for the bulk of implementation work.

## Intent

Route coding implementation through `codex exec` non-interactively.
Opus owns the plan (what to do, in what order, how to verify) and the review (did it actually work).
Codex owns the execution (file edits, shell commands, test runs).

## Trigger Keywords

- trigger: /codex, codex-executor, save opus tokens
- trigger: delegate to codex, use codex for implementation
- trigger: opus plan codex execute, hybrid workflow
- trigger: large implementation task, reduce token cost
- trigger: 用codex执行, 交给codex, codex实现

## Preconditions

1. `codex` binary is reachable: `codex --version` must succeed
2. Working directory is a git repository (required for diff-based review)
3. User has specified the task clearly enough for Opus to write a detailed plan
4. Codex config at `~/.codex/config.toml` has `approval_policy = "never"` for non-interactive use

Check preconditions:
```bash
codex --version && echo "OK: codex available" || echo "FAIL: codex not found"
git rev-parse --git-dir 2>/dev/null && echo "OK: git repo" || echo "WARN: not a git repo"
```

## Decision Boundary

**Use this skill when:**
- Implementation task is well-scoped with clear success criteria
- Task involves multiple file edits that would burn Opus tokens on boilerplate
- Working codebase is in a git repo (so `git diff` can serve as ground truth)
- You want to save Opus tokens on bulk coding work

**Do NOT use this skill when:**
- Task requires Opus-level architectural reasoning throughout execution (not just at plan time)
- Codebase is not in a git repo (review phase cannot verify changes reliably)
- Task is a one-line fix (three-phase overhead not worth it)
- Interactive debugging is needed that Codex cannot resolve autonomously

## Core Workflow

### Phase 1: Opus Plans (you are Opus)

Analyze the user's request and produce a structured plan. Write it to a temp file:

**Plan file path:** `C:/Users/Administrator/.codex/tmp/codex-plan-<YYYYMMDD-HHmmss>.md`

**Plan format:**

```markdown
## CODEX EXECUTION PLAN

### Context
- Working directory: <absolute path>
- Tech stack: <key technologies>
- Test command: <exact command to run tests, or NONE>
- Lint command: <exact command to lint, or NONE>
- Must not break: <files or interfaces that must stay intact>

### Objective
<One sentence: what this achieves and how success is verified>

### Success Criteria
- [ ] <Specific verifiable outcome 1>
- [ ] <Specific verifiable outcome 2>
- [ ] All existing tests pass after changes

### Implementation Steps
1. <Action on specific file path>
   - Change: <what to add/modify/delete>
   - Verify: <command that confirms this step worked>
2. <Next action>
   ...
N. Run full test suite: <test command>
   Expected: <expected output pattern>

### Constraints
- Do not modify: <protected files/dirs>
- Do not change: <protected interfaces>
- Must preserve: <backwards compatibility notes>
```

**Complexity routing:**

| Signal | Profile | Reasoning |
|--------|---------|-----------|
| Single file, no cross-module deps, docs/formatting | `speed` | Low reasoning, fast |
| Multi-file, cross-module, has test requirements | `auto-max` | High reasoning, thorough |

### Phase 2: Codex Executes

Build and invoke the Codex command:

```bash
# Variables
TMP="C:/Users/Administrator/.codex/tmp"
TS=$(date +%Y%m%d-%H%M%S)
PLAN_FILE="$TMP/codex-plan-$TS.md"
EVENTS_FILE="$TMP/codex-events-$TS.jsonl"
SUMMARY_FILE="$TMP/codex-summary-$TS.txt"
WORKDIR="<from plan Context>"
PROFILE="<speed|auto-max from complexity routing>"

# Execute (timeout: speed=300s, auto-max=900s)
timeout ${TIMEOUT} codex exec \
  -C "$WORKDIR" \
  -p "$PROFILE" \
  --color never \
  --json \
  -o "$SUMMARY_FILE" \
  "$(cat $PLAN_FILE)" \
  > "$EVENTS_FILE" 2>&1

CODEX_EXIT=$?
```

**Important:**
- Run this via the Bash tool with `run_in_background: true` for long tasks
- Set appropriate timeout (speed: 300000ms, auto-max: 600000ms)
- After completion, read the output files

### Phase 3: Opus Reviews (you are Opus again)

Collect evidence from three sources, in priority order:

**Source 1 — Ground truth (highest weight):**
```bash
git -C "$WORKDIR" diff HEAD
```

**Source 2 — Codex's self-report:**
```bash
cat "$SUMMARY_FILE"
```

**Source 3 — Failed commands (find hidden failures):**
```bash
grep -i "exit_code" "$EVENTS_FILE" | grep -v '"exit_code":0' | head -20
```

**Review checklist:**
1. Does `git diff HEAD` show the changes described in the plan's Implementation Steps?
2. Do the Success Criteria now hold based on actual file state?
3. Are there any failed shell commands (non-zero exit codes) in the event stream?
4. If the plan included a test command, did it appear in the event stream with exit code 0?

**Verdict:**
- **PASS**: All criteria met, no failed commands, tests passed → Done!
- **PARTIAL**: Some steps done, recoverable gap → Generate PATCH and retry
- **FAIL**: Critical step missing, test failure, or no changes → Generate PATCH and retry

### Retry Cycle (max 3 attempts)

On PARTIAL or FAIL, generate a focused `PATCH_PROMPT` (NOT a full re-plan):

```markdown
## CODEX PATCH TASK

### What Went Wrong
<Specific finding from review — exact file, exact symptom>

### Current State (from git diff)
<Paste the relevant portion of git diff showing current state>

### Required Fix
<Minimal instruction to address the gap>

### Verification
Run: <exact command>
Expected: <expected output>
```

Invoke `codex exec` again with `PATCH_PROMPT` as a **fresh independent call** (no session reuse):

```bash
PATCH_FILE="$TMP/codex-patch-$TS-attempt$N.md"
# write PATCH_PROMPT to PATCH_FILE, then:
timeout ${TIMEOUT} codex exec \
  -C "$WORKDIR" \
  -p "$PROFILE" \
  --color never \
  --json \
  -o "$TMP/codex-patch-summary-$TS-attempt$N.txt" \
  "$(cat $PATCH_FILE)" \
  > "$TMP/codex-patch-events-$TS-attempt$N.jsonl" 2>&1
```

Re-run Phase 3 review. After 3 failed attempts: emit blocker report to user and stop.

## Profile Routing Reference

| Signal | Profile | Reasoning Level | Expected Duration |
|--------|---------|----------------|-------------------|
| SIMPLE (single file, no cross-module deps) | `speed` | low | 30-120s |
| COMPLEX (multi-file, cross-module, tests) | `auto-max` | xhigh | 120-600s |

Codex profiles are pre-configured in `~/.codex/config.toml`.

## Token Budget Comparison

| Phase | Model | Est. Tokens |
|-------|-------|-------------|
| Phase 1: Opus plans | Opus | ~3000-5000 |
| Phase 2: Codex executes | gpt-5.4 (cheap) | ~10000-50000 |
| Phase 3: Opus reviews | Opus | ~2000-5000 |
| **Opus total** | | **~5000-10000** |
| Pure Opus baseline | | ~30000-80000 |
| **Savings** | | **70-85% Opus tokens** |

## Red Flags

- NEVER skip Phase 3 review because Codex's summary says "done" — always read `git diff HEAD`
- NEVER pass the previous session's events file as context to a retry call — keep Codex exec stateless
- NEVER attempt more than 3 retries automatically — escalate to human after third failure
- NEVER use this skill without a git repo — review phase depends on `git diff`
- NEVER write plan files to the project working directory — use `~/.codex/tmp/` to avoid polluting the project
- ALWAYS set `--color never` — ANSI codes corrupt the JSONL event stream
- ALWAYS use `timeout` to prevent hung Codex processes

## Smoke Command

```bash
codex --version && echo "codex-executor: smoke OK"
```

## Verification

1. `codex --version` returns `codex-cli 0.118.0` or higher
2. Plan file generated at `~/.codex/tmp/codex-plan-*.md`
3. Events file has content: `wc -l "$EVENTS_FILE"` > 0
4. Phase 3 evidence includes `git diff HEAD` output (not just Codex self-report)
5. Review verdict is one of: PASS / PARTIAL / FAIL with specific evidence

## Output Contract

After workflow completion, report to user:

| Field | Description |
|-------|-------------|
| `plan_file` | Path to the plan written in Phase 1 |
| `codex_exit_code` | Exit code from `codex exec` |
| `events_file` | Path to JSONL event stream |
| `summary_file` | Path to Codex's last-message summary |
| `git_diff_summary` | Key changes from `git diff HEAD` |
| `review_verdict` | PASS / PARTIAL / FAIL with evidence |
| `retry_count` | Number of retry cycles used (0-3) |
| `final_status` | completed / blocked / partial |
| `next_actions` | Follow-up steps or human escalation path |
