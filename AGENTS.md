# AGENTS.md

Dark factory: the human gives a goal once; agents deliver a reviewed PR. The human merges.

## Roles (one provider: Anthropic, newest generation only)

Prices are per MTok, input/output, from platform.claude.com/docs/en/about-claude/pricing (checked 2026-11-04). Models are ranked by price: Fable 5.1 > Opus 5.5 > Sonnet 5.5 > Haiku 5.5. Mythos 5.1 is left out because availability is limited. Fable 5.1 needs usage credits this account doesn't have, so the Auditor uses Opus 5.5, the same model as the Executor, under the "no alternative" exception (the human signed off on 2026-11-04). It still runs fresh and at high effort.

| Role | Model | Price | Effort |
|---|---|---|---|
| Orchestrator/Planner | `claude-sonnet-5-5` | $2 / $10 | high |
| Executor | `claude-opus-5-5` | $4 / $20 | default |
| Auditor | `claude-opus-5-5` | $4 / $20 | high |
| Tooler | `claude-haiku-5-5` | $0.10 / $0.50 (prompt ≤100k tokens); $0.50 / $2.50 (>100k) | default |

- **Orchestrator/Planner**: the main session (`claude --model claude-sonnet-5-5 --effort high`). Plans, splits the goal into tasks, dispatches, and decides. Never writes code.
- **Executor**: the best coding model that isn't the priciest. Writes the tests and the code for one task.
- **Auditor**: the most capable model at high effort, started fresh for every audit. Outputs ACCEPT or REJECT with reasons and never fixes anything. REJECT if:
  - tests echo the code instead of the goal;
  - anything is weakened or skipped;
  - a test is deleted;
  - a protected file changed;
  - a dependency is added or changed without being flagged in the PR body;
  - any rule here or in the goal is broken.

  The Auditor also tries to break the build with inputs the tests lack.
- **Tooler**: the cheapest model. Handles setup and CI on its own, with no Executor or Auditor.
- Don't reuse a model across roles unless there's no alternative. Ask the human only on a tie.

## Subagents

- Each subagent sees only its task: no conversation history and no other tasks.
- Subagents never spawn subagents. Start every one with `--disallowedTools Agent`.
- Independent tasks run in parallel, each in its own herdr pane:
  ```
  herdr pane split --current --direction right --cwd <worktree> --no-focus   # → .result.pane.pane_id
  herdr agent start <name> --kind claude --pane <pane_id> -- --model <model> [--effort high] --disallowedTools Agent   # name: <role>-<task>, [a-z][a-z0-9_-]{0,31}
  herdr agent prompt <name> "<task>" --wait
  ```
- Wait only with herdr's blocking wait (`agent prompt --wait` or `herdr agent wait <name>`). Never poll in a loop.

## Git, worktrees, PRs

- Use worktrees only via `wt`, never the primary checkout, and no hooks.
  - Create (every PR branch, from freshly fetched `main`): `git fetch origin main && wt switch --create <branch> --base origin/main --no-hooks --no-cd --format json`. The JSON output gives the worktree path.
  - Remove: `wt remove <branch> --no-hooks --foreground`
- Worktrees live under `.worktrees/` (set by `worktree-path` for this project in `~/.config/worktrunk/config.toml`), and a branch's first push is `git push --set-upstream origin <branch>`.
- `task gate` must pass locally before any PR.
- GitHub only via `gh`. Never push to `main`; the human merges.
- Open a PR only after the Auditor ACCEPTs. The PR body shows:
  - the `task gate` result;
  - a per-file coverage table: `go test -coverprofile=cover.out ./...` then `go tool cover -func=cover.out`, its rows grouped by file, with no custom tooling;
  - every dependency added or changed.

## Go tests

- One table-driven test per exported function, and each one calls `t.Parallel()`.
- One blackbox test file (`package <pkg>_test`) per source file.
- Tests assert the goal's behavior, not the code's output.
- Commit tests before the code they test.
- Keep the CLI `main` thin.
- Coverage ≥ 90%.
- Never weaken a test.

## Guardrails

- These rules hold unless the human explicitly allows an exception.
- Protected files need the human's sign-off: Taskfile, lint config, CI, Dependabot, `AGENTS.md` (and `CLAUDE.md`), `spec/target.html`, repo settings and rulesets.
- Same failure after two rounds: stop and escalate to the human.
