# multi-autoresearch

Multi-agent autoresearch: parallel experiments via git worktrees, with multi-perspective ideation when stuck.

The core loop is Karpathy's autoresearch (`edit → commit → run → measure → keep/discard`), extended to run multiple experiments per wave in isolated worktrees. A single orchestrator manages the wave lifecycle: dispatch experimenters, collect results, select the winner, cherry-pick it onto the main branch, and record everything.

## Constants

### Experiment Loop

- **TIMEOUT_MINUTES = 10** — Kill runs exceeding this. Clamped to remaining budget if set.
- **MAX_CONSECUTIVE_DISCARD = 3** — After this many all-discard waves, trigger multi-agent ideation.

### Wave Execution

- **WAVE_SIZE_DEFAULT = 1** — Sequential by default. User opts into parallelism during setup based on workload class.
- **EXPERIMENT_TIMEOUT_MULTIPLIER = 2** — Per-experiment timeout = baseline_runtime * this multiplier.

### Ideation

- **IDEATION_SUBAGENTS = 2** — Number of Claude subagents for divergent hypothesis generation.
- **IDEATION_MODE_DEFAULT = "on-stuck"** — Options: `"on-stuck"`, `"wave-0-only"`, `"periodic"`.

### Budget

- **EXPERIMENT_BUDGET_HOURS = 0** — Default: unbounded (loop until stopped). Override: `— budget: Nh`.

## State Schema (mar-state.json)

Single state file. Updated on every wave transition. Written atomically (temp + rename).

```
tag: string                               # branch tag, e.g. "mar26"
branch: "autoresearch/<tag>"              # experiment branch name
wave_size: number                         # experiments per wave
workload_class: "gpu-heavy" | "cpu-only" | "lightweight"
ideation_mode: "on-stuck" | "wave-0-only" | "periodic"
ideation_interval: number | null          # waves between periodic ideation (if periodic)
parallax: boolean                         # whether /relay is enabled for ideation
metric_direction: "lower" | "higher"      # which direction is better
current_wave: number                      # 0 = baseline, 1+ = experiment waves
phase: "setup" | "baseline" | "wave_running" | "between_waves" | "completed" | "failed"
best_commit: string                       # 7-char hash of current best
best_metric: number | null                # current best metric value
baseline_metric: number | null            # baseline value for delta reporting
baseline_runtime_sec: number | null       # used to compute experiment timeouts
consecutive_all_discard: number           # resets on any keep
timestamp: ISO 8601                       # updated on every phase transition
status: "in_progress" | "completed" | "failed"
budget_hours: number | null               # null = unbounded
start_time: ISO 8601                      # wall-clock start for budget tracking
in_flight: [                              # populated during wave_running, cleared after
  {
    experiment_id: string,                # e.g. "wave5-exp0"
    hypothesis: string,                   # one-line description
    hypothesis_source: "orchestrator" | "ideation" | "parallax",
    base_commit: string,                  # 7-char hash this experiment branched from
    branch: string,                       # worktree branch name
    worktree_path: string,                # path to worktree directory
    status: "dispatched" | "running" | "done" | "timeout" | "aborted",
    dispatched_at: ISO 8601
  }
]
```

## File Contracts

| File | Producer | Consumer | Notes |
|------|----------|----------|-------|
| `mar-state.json` | orchestrator | orchestrator (resume) | Atomic writes. Config + phase pointer. |
| `results.tsv` | orchestrator | orchestrator, nanoresearch write/review | Nanoresearch-compatible format. Committed after each wave. |
| `autoresearch.md` | orchestrator | orchestrator (resume), fresh agents | Recovery doc: objective, metric, command, files in scope. |
| `status.md` | orchestrator | user (morning read) | Human-readable progress summary. Updated each wave. |
| `experiment-result.json` | experimenter (in worktree) | orchestrator | Structured result per experiment. Lives in worktree, not committed to main. |
| `run.log` | experimenter (in worktree) | experimenter (diagnostics) | Command output. Not committed. |

## Recovery Protocol

1. Read `mar-state.json`. If missing or corrupt, start fresh.
2. If `phase == "wave_running"`: reconcile `in_flight` entries against filesystem:
   - For each entry: check if worktree exists AND `experiment-result.json` is present
   - If result exists: salvage (read result, include in wave evaluation)
   - If no result: mark `aborted`
   - Clean up all worktrees after reconciliation
3. If `phase == "between_waves"`: resume from `current_wave + 1`.
4. Derive `best_commit` and `best_metric` from last `keep` row in `results.tsv` (source of truth).
5. If `results.tsv` and `mar-state.json` disagree on best, trust `results.tsv`.
