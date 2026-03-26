---
description: >
  Multi-agent autoresearch: parallel experiments via worktrees with multi-perspective
  ideation when stuck. Use when asked to "run multi-autoresearch", "parallel experiments",
  "multi-agent optimize", or invokes /multi-autoresearch. Also trigger when the user
  wants autoresearch with parallel execution or worktree isolation.
argument-hint: "[topic or overrides]"
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, Skill
---

# Multi-Agent Autoresearch

You are a completely autonomous researcher running parallel experiments. Try things out. If they work, keep. If they don't, discard. Never stop.

Read all constants from this project's **CLAUDE.md** before proceeding. Read the state schema and file contracts.

## Critical Invariants

These are non-negotiable. Violating any of them makes the run unreliable.

1. **Every experiment runs in its own isolated git worktree.** Do NOT run experiments in the orchestrator's main checkout. The orchestrator owns worktree creation, bookkeeping, and cleanup. If the platform provides first-class worktree-isolated workers (e.g., Claude Code's `Agent(isolation: "worktree")`), use them. Otherwise, create worktrees explicitly with `git worktree add` and dispatch workers into them.
2. **Maintain `mar-state.json` throughout.** Create it during setup, update it on every phase transition, write it atomically (temp + rename). This is required for crash recovery.
3. **Use the nanoresearch-compatible results.tsv format.** 7 TAB-separated columns: `#`, `timestamp`, `commit`, `metric`, `sanity`, `status`, `description`. Not 5 columns. Not CSV. Append `[wave:N]` tag to every description.
4. **Read `agents/experimenter.md` and use it as the prompt template** for experiment workers. Fill in the `$VARIABLE` placeholders with actual values. Do not improvise a different prompt.
5. **Cherry-pick, never merge.** Use plain `git cherry-pick <commit>`. No `-X theirs`. Fail loudly on conflict.

## Inline Overrides

Parse `$ARGUMENTS` by splitting on ` — `. First segment = topic/goal. Subsequent segments are `key: value` pairs or bare flags. Known overrides:

| Override | Effect |
|----------|--------|
| `wave: N` | Set wave size |
| `parallax: on/off` | Enable/disable cross-model ideation |
| `budget: Nh` or `Nm` | Wall-clock budget |
| `ideation-interval: N` | Enable periodic ideation every N waves |
| `workload: gpu-heavy/cpu-only/lightweight` | Skip workload class question |

Echo applied config at startup.

## Resumption

Check for `mar-state.json` in the project root:
- If absent → **fresh start**.
- If present but invalid JSON → rename to `mar-state.json.bak`, warn, **fresh start**.
- If `status` is `"completed"` or `"failed"` → **fresh start**.
- If `status` is `"in_progress"` AND `timestamp` > 24 hours old → **fresh start** (stale).
- If `status` is `"in_progress"` AND within 24 hours → **resume** from recorded phase.

**On fresh start:**
1. Clean stale artifacts: `rm -f mar-state.json status.md`.
2. Proceed to Setup.

**On resume:**
1. Read `branch` from `mar-state.json`. `git checkout <branch>`.
2. Follow the Recovery Protocol in the project's CLAUDE.md.
3. Jump to the recorded phase.

## Setup

Work with the user to establish these. Auto-detect from the repo aggressively (run scripts, Makefiles, pyproject.toml). Only ask about genuinely unresolved choices. Confirm in one round.

1. **Metric**: What to optimize, and the direction (lower/higher is better). Ask — do not guess direction.
2. **Sanity metric** (optional): A metric that must not degrade. Prevents proxy overfitting.
3. **Command**: The shell command that runs one experiment.
4. **Metric extraction**: How to get the metric from output.
5. **Sanity extraction** (if sanity metric defined): How to get the sanity metric.
6. **Comparison protocol**: What stays fixed across experiments (dataset, evaluation harness, seed policy).
7. **Files in scope**: Which files may be edited. Default to smallest coherent set.
8. **Constraints**: Hard rules. Default: no new dependencies, do not modify evaluation/data code.
9. **Guard** (optional): A command that must always pass. Protects existing behavior.
10. **Workload class**: Ask the user (unless overridden):
    - `gpu-heavy`: Experiments saturate GPU. Default wave_size=1.
    - `cpu-only`: CPU-bound experiments. Default wave_size=2.
    - `lightweight`: Fast, low-resource experiments. Default wave_size=3.
    - User can override wave_size regardless of class.

Then:

1. Resolve the default branch (`git symbolic-ref refs/remotes/origin/HEAD` or fall back to `main`). Create `git checkout -b autoresearch/<tag> <default-branch>`. Tag = today's date (e.g., `mar26`). If branch exists, append sequence number: `mar26-2`.
2. Read every in-scope file AND all relevant read-only files (evaluation code, data loading, configs).
3. **Preflight**: Verify prerequisites — data/artifacts exist, command is available, output paths are writable, `timeout` command exists (check for `gtimeout` on macOS). If anything is missing, tell the user.
4. Add `run.log` to `.gitignore` if not present. Commit.
5. Create `results.tsv` with header row (7 TAB-separated columns):
   ```
   #	timestamp	commit	metric	sanity	status	description
   ```
6. Create `mar-state.json` with ALL fields from the state schema in CLAUDE.md. Set `phase: "baseline"`, `status: "in_progress"`, `current_wave: 0`, `consecutive_all_discard: 0`, `start_time` to now. Include `wave_size`, `workload_class`, `ideation_mode`, `parallax`, `metric_direction`, `budget_hours`, `tag`, `branch`. Set `in_flight: []`.
7. Write atomically: `echo '<json>' > mar-state.json.tmp && mv mar-state.json.tmp mar-state.json`.

## Baseline

Run the command as-is, no modifications. Record as experiment #0 in `results.tsv`.

```bash
timeout ${TIMEOUT_MINUTES}m $COMMAND > run.log 2>&1
```

Extract metric. If baseline crashes: diagnose from `run.log`, fix environment (not in-scope code), retry up to 3 times. If still failing → set `status: "failed"` and stop.

Record baseline runtime for experiment timeout calculation:
```
baseline_runtime_sec = <measured wall-clock seconds>
```

Write `autoresearch.md` recovery doc:
```
# Autoresearch: <goal>
## Objective  ## Metric  ## Sanity Metric  ## Command
## Metric Extraction  ## Comparison Protocol
## Files in Scope  ## Off Limits  ## Constraints  ## Guard
## What's Been Tried — (update every ~5 experiments)
```

Update `mar-state.json`: `phase: "between_waves"`, `best_commit`, `best_metric`, `baseline_metric`, `baseline_runtime_sec`.

Commit: `git add results.tsv autoresearch.md mar-state.json .gitignore && git commit -m "baseline: metric=$METRIC"`.

Start the wave loop.

## The Wave Loop

**LOOP** until budget expires, wave cap reached, or manually stopped. Do NOT ask "should I continue?" — the user may be away.

### Before Each Wave

1. **Budget check**: If `budget_hours` is set, check `$(date +%s)` against `start_time`. If budget expired, finish and print summary.
2. **Ideation trigger check**:
   - Wave 0 (first after baseline): trigger ideation.
   - `consecutive_all_discard >= MAX_CONSECUTIVE_DISCARD`: trigger ideation.
   - `ideation_mode == "periodic"` AND `current_wave % ideation_interval == 0`: trigger ideation.
   - Otherwise: orchestrator generates hypotheses inline.

### Hypothesis Generation

**Inline (default, fast):** Generate `wave_size` hypotheses from:
- Recent results.tsv entries (last 10 rows)
- `autoresearch.md` "What's Been Tried" section
- Current in-scope file contents

Everything in scope is fair game: architecture, algorithms, hyperparameters, batch size, model size, optimizer, scheduling, radical restructuring. Do not limit to parameter tuning. Prioritize diversity — tag each hypothesis by category (architectural, parametric, algorithmic, simplification).

**Multi-agent ideation (on trigger):**

1. Build context: current best code, results.tsv summary (group by category: subsystem, parameter family, outcome), "What's Been Tried".
2. Dispatch 2 ideation workers concurrently using the platform's native concurrency mechanism. Assign distinct lenses:
   - **Architectural lens**: Propose experiments focused on structural changes (model architecture, data flow, system design). Each proposal: one-line hypothesis + `change_surface` + `expected_effect`.
   - **Parametric lens**: Propose experiments focused on hyperparameters, configuration, training dynamics. Same format.
3. If `parallax == true` and a cross-model relay transport is available: dispatch 1 **Contrarian** reviewer through that transport — propose experiments that deliberately contradict the current trajectory. Read `references/prompting-codex.md` before composing the request. Include anti-recursion constraints. If no relay transport, dispatch an additional same-model contrarian worker instead and note the downgrade.
4. Wait for all workers. Synthesize: deduplicate by subsystem + parameter family, pick `wave_size` diverse experiments.

### Wave Execution

For each wave:

**1. Record wave start.** Update `mar-state.json`: `phase: "wave_running"`, `current_wave++`. Populate `in_flight` array with one entry per hypothesis (experiment_id, hypothesis, base_commit, status: "dispatched"). Write atomically.

**2. Dispatch experimenters.** Read `agents/experimenter.md` ONCE at the start of the loop. For each hypothesis:

a. Create a dedicated worktree and branch:
```bash
git worktree add -b mar-wave${N}-exp${M} <worktree_path> <wave_base_commit>
```
Use a path like `.worktrees/mar-wave${N}-exp${M}` relative to the repo root.

b. Record `experiment_id`, `branch`, `worktree_path`, `base_commit`, and `status: "dispatched"` in `mar-state.json` (in the `in_flight` array). Write atomically.

c. Fill `agents/experimenter.md` with experiment-specific variables:
- `$HYPOTHESIS` → the one-line hypothesis for this experiment
- `$METRIC_NAME`, `$METRIC_DIRECTION` → from setup
- `$SANITY_METRIC`, `$SANITY_EXTRACTION` → from setup (or "none")
- `$COMMAND` → the shell command (prepend venv activation if needed)
- `$EXTRACTION` → metric extraction command
- `$FILES_IN_SCOPE` → the file list
- `$CONSTRAINTS` → constraint rules
- `$TIMEOUT_MINUTES` → `min(TIMEOUT_MINUTES, baseline_runtime_sec * EXPERIMENT_TIMEOUT_MULTIPLIER / 60)`
- `$BEST_METRIC`, `$BASE_COMMIT` → current best
- `$WAVE_NUMBER`, `$EXP_INDEX`, `$WAVE_SIZE` → wave metadata
- `$DESCRIPTION` → short description for the commit message
- `$WORKTREE_PATH` → the worktree path created in step (a)

d. Dispatch one worker with the filled prompt, giving it exclusive ownership of that worktree. Use the platform's native concurrency mechanism:
- **Claude Code**: `Agent(isolation: "worktree", run_in_background: true, prompt: <filled template>)` — the Agent tool can manage its own worktree, in which case skip step (a) and let the tool create the worktree. Record the returned worktree path in `in_flight`.
- **Codex / other**: Run the worker sequentially or via platform-native concurrency. The worker must execute all commands from `$WORKTREE_PATH`.

**Prepend venv activation** if the project uses a Python venv: add `source <venv_path>/bin/activate &&` before the command in `$COMMAND`. Detect venv at `.venv/`, `venv/`, or the path the user specified during setup.

**3. Wait for all experimenters.** Hard gate — do not proceed until every worker finishes. Use the platform's native wait/completion mechanism. If a worker exceeds the per-experiment timeout (tracked by `dispatched_at` + timeout), mark it `timeout`.

**4. Collect results.** For each worker:
- The orchestrator knows each experiment's `worktree_path` from `mar-state.json`.
- Read `$WORKTREE_PATH/experiment-result.json`.
- If file is missing: check if a commit was made (`git -C <worktree_path> log --oneline -1`). If commit exists but no result file, attempt metric extraction from `run.log` in the worktree. If still no result, mark `crash`.
- Verify `base_commit` matches expected `wave_base_commit`. If mismatch, mark `stale`.
- Update `in_flight` entry status to `done` or `crash`.

**5. Select winner.**
- Among experiments with `status: "candidate"` and valid metric:
  - Compare against current best (respecting `metric_direction`).
  - If sanity metric defined: discard any candidate whose sanity degraded beyond bound.
  - Best improver wins.
  - Ties: simplicity criterion — fewer `changed_files` wins. If still tied, shorter diff wins.
  - If none improved: all discard, increment `consecutive_all_discard`.
- If winner found: reset `consecutive_all_discard` to 0.

**6. Merge winner (if any).**
```bash
git cherry-pick <winning_head_commit>
```
- If cherry-pick succeeds cleanly: proceed to guard.
- If cherry-pick conflicts: `git cherry-pick --abort`. Log as `stale_merge`. Archive the patch: `git -C <worktree_path> diff <base>..<head> > patches/wave<N>-exp<M>.patch`. Treat as discard.

**Guard (if defined):**
- Run guard in current working tree (the main branch after cherry-pick).
- If guard passes: keep.
- If guard fails: `git reset --hard HEAD~1`. Log as `guard_fail`. Treat as discard. Do NOT attempt rework in V1.

**7. Record all results to results.tsv.**

**THIS FORMAT IS MANDATORY.** 7 TAB-separated columns. Do NOT use a different format.

For each experiment in the wave, append a row. Use actual TAB characters (`\t`), not spaces.

Header (written during setup):
```
#	timestamp	commit	metric	sanity	status	description
```

Example rows:
```
0	2026-03-26T10:00:05	a1b2c3d	0.9979	98.2	keep	baseline
1	2026-03-26T10:12:30	b2c3d4e	0.9932	98.1	keep	increase LR to 0.04 [wave:1]
2	2026-03-26T10:12:30	c3d4e5f	1.0050	98.3	discard	switch to GeLU [wave:1]
3	2026-03-26T10:12:30	d4e5f6g	NA	NA	crash	double model width (OOM) [wave:1]
```

- `#`: Sequential experiment number (0 = baseline, continues across waves).
- `timestamp`: ISO 8601 with seconds (e.g., `2026-03-26T10:12:30`).
- `commit`: 7-char git hash. For keeps: the main-branch commit (after cherry-pick). For discards/crashes: the worktree commit.
- `metric`: Numeric value or `NA` for crashes.
- `sanity`: Numeric value, `NA`, or empty if no sanity metric.
- `status`: `keep`, `discard`, or `crash`.
- `description`: What was tried. **MUST end with `[wave:N]`** tag (except baseline which is `[wave:0]` or just "baseline").

Commit: `git add results.tsv autoresearch.md mar-state.json && git commit -m "wave $N: $SUMMARY"`.

**8. Clean up worktrees.** For each worktree branch created during this wave:
```bash
git worktree remove <path> --force 2>/dev/null
git branch -D <branch> 2>/dev/null
```
If a worker exited before producing changes and the worktree is already absent, skip gracefully.

**9. Update status.md.** Write the morning summary:
```markdown
# Autoresearch Status — updated <timestamp>

## Progress
- **Best**: <best_metric> (baseline: <baseline_metric>, delta: <delta>)
- **Waves**: <N> completed | **Experiments**: <total> total
- **Keeps**: <K> | **Discards**: <D> | **Crashes**: <C>

## Recent Activity
| Wave | Winner | Metric | Description |
|------|--------|--------|-------------|
(last 5 waves)

## Next Direction
<current hypothesis strategy / what the orchestrator plans to try next>

## Crashes Worth Investigating
(any crashes from the last 3 waves with error_tail)
```

Commit: `git add status.md && git commit -m "status: wave $N"`.

**10. Update mar-state.json.** Set `phase: "between_waves"`, update `best_commit`, `best_metric`, `timestamp`. Clear `in_flight`. Write atomically.

**11. Report.** Every 3 waves, print: `=== Wave N: best at X.XX, K keeps / D discards / C crashes ===`

**12. Update autoresearch.md** "What's Been Tried" section every ~5 waves.

**13. Repeat.** Go to "Before Each Wave".

### When Stuck (orchestrator-level)

If `consecutive_all_discard >= MAX_CONSECUTIVE_DISCARD` and ideation was already triggered:

1. Re-read ALL in-scope files with fresh eyes.
2. Read papers or references in `papers/` or `literature/` if present.
3. Consider radical structural changes — not just parameter tweaks.
4. Reason about execution bottlenecks (memory, compute, cache).
5. As a last resort, if a previous keep introduced a suboptimal direction, consider reverting to an earlier keep and branching differently. Check `results.tsv` for the commit hash of a better starting point.

**NEVER give up. NEVER ask the user for ideas. Think harder.**

### Bounded Mode

When invoked with `— budget: Nh` or `— waves: N`:
- Budget mode: check wall clock before each wave. Stop when exceeded.
- Wave cap: stop after N waves.

Print final summary:
```
=== Multi-Autoresearch Complete ===
Baseline: <value> → Best: <value> (<delta>)
Waves: N | Experiments: M
Keeps: K | Discards: D | Crashes: C
Best experiment: #<n> — <description>
```

## What You Cannot Do

- Edit files outside scope.
- Install new packages or dependencies (unless user explicitly allowed).
- Modify the evaluation or measurement harness.
- Stop to ask the user for permission.
- Modify guard/test files — always adapt the implementation.

## NEVER STOP.

Once the loop has begun, do NOT pause to ask the human if you should continue. Do NOT ask "should I keep going?" or "is this a good stopping point?" The human might be asleep. They expect you to work indefinitely until manually stopped or budget expires.
