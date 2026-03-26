# multi-autoresearch

Multi-agent autoresearch: parallel experiments via git worktrees, with multi-perspective ideation when stuck.

Based on [Karpathy's autoresearch](https://github.com/karpathy/autoresearch). Extends the single-agent experiment loop with:

- **Parallel waves**: Run 2-3 experiments concurrently in isolated git worktrees
- **Multi-perspective ideation**: When stuck, dispatch multiple agents with different analytical lenses to generate diverse hypotheses
- **Structured results**: JSON experiment results, nanoresearch-compatible TSV logging
- **Morning summary**: Wake up to a clear status report of overnight progress

## Usage

```
/multi-autoresearch
```

Setup is interactive — the orchestrator asks for metric, command, files in scope, and workload class, then starts the loop.

### Overrides

```
/multi-autoresearch — wave: 3 — parallax: on — budget: 4h
```

| Override | Effect |
|----------|--------|
| `wave: N` | Set wave size (parallel experiments per wave) |
| `parallax: on` | Enable cross-model ideation via /relay |
| `budget: Nh` | Wall-clock budget for the loop |
| `ideation-interval: N` | Enable periodic multi-agent ideation every N waves |

## How it works

1. **Setup**: Establish metric, command, extraction, files in scope, constraints
2. **Baseline**: Run the command as-is, record as experiment #0
3. **Wave loop**: For each wave:
   - Generate hypotheses (orchestrator, or multi-agent when stuck)
   - Dispatch experimenter agents in isolated worktrees
   - Collect results, select winner, cherry-pick to main branch
   - Record all results, update morning summary
4. **Repeat** until budget expires or manually stopped

## Requirements

- Claude Code or Codex (any platform with git worktree support)
- Git repository with clean working tree
