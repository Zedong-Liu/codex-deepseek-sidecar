---
name: deepseek-codex-subagent
description: Delegate self-contained side tasks to a DeepSeek-backed Codex CLI subagent through a clean-environment wrapper with persistent sessions and an interactive Terminal chat window. Use for independent exploration, code review, debugging, log inspection, implementation attempts, or benchmark supervision where DeepSeek can work from a clear brief.
---

# DeepSeek Codex Subagent (Community)

## Installation

Clone into `~/.codex/skills/`:

```bash
git clone <repo-url> ~/.codex/skills/deepseek-codex-subagent
cd ~/.codex/skills/deepseek-codex-subagent
chmod +x scripts/*
```

Optional symlink for shorter invocation:

```bash
ln -s ~/.codex/skills/deepseek-codex-subagent/scripts/codex-deepseek-subagent \
  ~/.codex/bin/codex-deepseek-subagent
```

## Prerequisites

A working DeepSeek profile in `~/.codex/config.toml`. Run `--configure` once to set it up interactively (detects VibeAround proxy on `127.0.0.1:12358` or instructs CC switch setup), then verifies connectivity with `codex doctor`.

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --configure
```

## Workflow

Each execution opens a Terminal window that shows live task output, then becomes a `deepseek>` follow-up prompt. Type `/exit` to leave the session idle for later resume.

Set one stable project path and keep it across workflow steps:

```bash
PROJECT="/absolute/path/to/project"
```

### 1. Start a new session

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --cd "$PROJECT" \
  "Explore relevant implementations. Report key files, conclusions, and next steps."
```

A session ID is recorded to `/tmp/deepseek-subagent-sessions.txt` keyed by workdir.

For parallel subagents in the same project, assign a stable task ID:

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --cd "$PROJECT" \
  --task-id review-auth \
  "Review the auth module. Report findings with file and line references."
```

### 2. Check session status before follow-ups

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --cd "$PROJECT" --status
```

Check a named task:

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --cd "$PROJECT" \
  --task-id review-auth --status
```

List recorded sessions for the project:

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --cd "$PROJECT" --list
```

| Status | Action |
| ------ | ------ |
| `running` | Wait. Never resume a running session concurrently. |
| `idle` | Continue with `--resume` (step 3). |
| `missing` | No recorded session for this directory. Start new (step 1). |
| `untracked` | Session ID is known but lacks a workdir record. Resume with `--session-id <UUID> --cd <DIR>`. |

### 3. Resume an idle session

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --cd "$PROJECT" --resume \
  "Continue with existing context; run verification and report results."
```

Resume a named task:

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --cd "$PROJECT" \
  --task-id review-auth --resume \
  "Continue with existing context; run verification and report results."
```

Resume a specific session by ID from any directory:

```bash
"<path-to-skill>/scripts/codex-deepseek-subagent" --session-id <UUID> \
  "Continue the task and report results."
```

## Flags

| Flag | Effect |
| ---- | ------ |
| `--profile NAME` | Codex profile (default `ds-sidecar`). |
| `--configure` | Interactive profile setup + doctor verification. No prompt. |
| `--cd DIR` | Workdir for new sessions and `--status` / `--resume` lookup. |
| `--task-id NAME` | Name a task within `--cd DIR` for later `--status` / `--resume` lookup. Use letters, numbers, `.`, `_`, `:`, or `-`. |
| `--resume`, `-r` | Resume latest session for `--cd DIR`. |
| `--session-id UUID` | Resume a specific session. |
| `--status` | Report `running`/`idle`/`missing`/`untracked`. No model request. |
| `--list` | List recorded sessions for `--cd DIR`, optionally filtered by `--task-id`. No model request. |
| `--json` | JSONL event stream. |
| `--pass-env NAME` | Copy one UTF-8 env var into the isolated child. Repeatable. |
| `--no-monitor` | Suppress Terminal window (automation runs). |
| `--no-doctor-check` | Skip pre-flight `codex doctor` connectivity check. |
| `--verbose-stderr` | Show normally-filtered Codex startup warnings. |
| `-` | Read prompt from stdin. |

The wrapper isolates environment variables. Pass only what the task needs:

```bash
API_KEY="$SOME_SECRET" \
  "<path-to-skill>/scripts/codex-deepseek-subagent" \
  --cd "$PROJECT" --pass-env API_KEY "Run specified tests and report output."
```

## Prompt shape

Give the subagent a self-contained brief:

```
Task: [concrete, bounded goal]
Context: [paths, error snippet, constraints]
Expected: [conclusion, evidence, patch/test references if code was edited]
Constraints: [read-only scope, or explicit permission to edit/run tests]
```

Use the Terminal `deepseek>` prompt only after the subagent's initial task completes — the session persists and the window remains interactive.

This skill only provides subagent session execution and lifecycle controls. The calling agent is responsible for task decomposition, coordination, result synthesis, and deciding whether edits are allowed. For multiple concurrent subagents in one repository, prefer `--task-id` or the returned `--session-id`; do not rely on bare `--resume --cd` because it resumes the latest recorded session for that workdir.

## Error recovery

- **`tokio-runtime-worker` panic** or **`JoinError::Panic`** → infrastructure failure. Re-run with `--verbose-stderr` for diagnosis.
- **`connectivity check failed`** → profile is misconfigured or provider is unreachable. Run `--configure` to repair, or use `--no-doctor-check` if the check is overly strict (e.g., proxy returns 404 on the probe route).
- **`no previous session found`** → no recorded session for this workdir. Start a new session without `--resume`.
