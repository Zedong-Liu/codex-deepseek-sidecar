# DeepSeek Codex Subagent

A [Codex CLI](https://github.com/openai/codex) skill that delegates tasks to a DeepSeek-backed subagent with persistent sessions and an interactive Terminal window.

Each run opens a Terminal showing live output. When the task finishes, the window becomes a `deepseek>` prompt for follow-ups — the session stays alive until you `/exit`.

## What it does

- **Delegate bounded tasks** to a DeepSeek subagent from within a Codex session
- **Persistent sessions** — resume a session later with full context intact
- **Interactive Terminal** — live log tailing, then a follow-up prompt
- **Isolated environment** — `env -i` child process, UTF-8 validated, no host env leakage
- **Concurrency guard** — per-session lock files prevent double-resume

## Prerequisites

- [Codex CLI](https://github.com/openai/codex) installed and authenticated
- A DeepSeek profile reachable from Codex (VibeAround proxy or CC switch)
- macOS (Terminal.app for the monitor window)

## Install

```bash
git clone https://github.com/<your-org>/deepseek-codex-subagent.git \
  ~/.codex/skills/deepseek-codex-subagent
cd ~/.codex/skills/deepseek-codex-subagent
chmod +x scripts/*
```

Optional: add a symlink so you can just type `codex-deepseek-subagent`:

```bash
ln -s ~/.codex/skills/deepseek-codex-subagent/scripts/codex-deepseek-subagent \
  ~/.codex/bin/codex-deepseek-subagent
```

## Quick start

Set up the DeepSeek profile (one-time):

```bash
~/.codex/skills/deepseek-codex-subagent/scripts/codex-deepseek-subagent --configure
```

Delegate a task:

```bash
~/.codex/skills/deepseek-codex-subagent/scripts/codex-deepseek-subagent \
  --cd /path/to/project \
  "Review src/auth.py for security issues. Report findings with line references."
```

For multiple parallel tasks in one project, name each task:

```bash
~/.codex/skills/deepseek-codex-subagent/scripts/codex-deepseek-subagent \
  --cd /path/to/project \
  --task-id review-auth \
  "Review src/auth.py for security issues. Report findings with line references."
```

Check if the session is still alive:

```bash
~/.codex/skills/deepseek-codex-subagent/scripts/codex-deepseek-subagent \
  --cd /path/to/project --task-id review-auth --status
```

List recorded sessions for a project:

```bash
~/.codex/skills/deepseek-codex-subagent/scripts/codex-deepseek-subagent \
  --cd /path/to/project --list
```

Resume for follow-up work:

```bash
~/.codex/skills/deepseek-codex-subagent/scripts/codex-deepseek-subagent \
  --cd /path/to/project --task-id review-auth --resume \
  "Address the first two findings from your previous review."
```

## Key flags

| Flag | Purpose |
| ---- | ------- |
| `--cd DIR` | Project directory for session tracking |
| `--task-id NAME` | Name a task within `--cd DIR` for status/resume lookup |
| `--resume`, `-r` | Continue the latest session in that directory |
| `--session-id UUID` | Resume a specific session |
| `--status` | Report `running` / `idle` / `missing` / `untracked` |
| `--list` | List recorded sessions for `--cd DIR` |
| `--no-monitor` | Skip the Terminal window (CI / automation) |
| `--no-doctor-check` | Skip connectivity check before launch |
| `--pass-env NAME` | Pass an env var into the isolated child |
| `--profile NAME` | Use a non-default Codex profile |
| `--json` | JSONL event stream |

See `SKILL.md` for the full agent-facing documentation and `--help` for all options.

## License

Apache 2.0 — see [LICENSE](./LICENSE).
