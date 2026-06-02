---
name: codex-deepseek-sidecar
description: Delegate self-contained side tasks to a DeepSeek-backed Codex CLI sidecar through a clean-environment wrapper with persistent sessions and an interactive Terminal chat window. Use for independent exploration, code review, debugging, log inspection, implementation attempts, or benchmark supervision where DeepSeek can work from a clear brief.
---

# Codex DeepSeek Sidecar

## Installation

Clone into `~/.codex/skills/`:

```bash
git clone https://github.com/Zedong-Liu/codex-deepseek-sidecar.git \
  ~/.codex/skills/codex-deepseek-sidecar
cd ~/.codex/skills/codex-deepseek-sidecar
chmod +x scripts/*
```

Optional symlink for shorter invocation:

```bash
ln -s ~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  ~/.codex/bin/codex-deepseek-sidecar
```

## Prerequisites

A working DeepSeek profile in `~/.codex/config.toml`. Run `--configure` once to create a default profile for the included local proxy if needed, then verify connectivity with `codex doctor`.

If no DeepSeek-compatible Codex provider or local proxy is already configured, start the lightweight proxy first:

```bash
DEEPSEEK_API_KEY="sk-..." "<path-to-skill>/scripts/deepseek-responses-proxy"
```

The proxy listens on `127.0.0.1:12359` by default and bridges Codex Responses requests to DeepSeek Chat Completions. It defaults to a 128 MB request body limit for long-context sidecar runs. Keep the proxy running and use `"<path-to-skill>/scripts/deepseek-responses-proxy" --status` before starting a duplicate.

Run configuration once, not before every sidecar task:

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --configure
```

## Workflow

Each execution opens a Terminal window that shows live task output, then becomes a `deepseek>` follow-up prompt. Type `/exit` to leave the session idle for later resume.

Set one stable project path and keep it across workflow steps:

```bash
PROJECT="/absolute/path/to/project"
```

### 1. Start a new session

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --cd "$PROJECT" \
  "Explore relevant implementations. Report key files, conclusions, and next steps."
```

A session ID is recorded to `/tmp/deepseek-sidecar-sessions.txt` keyed by workdir.

For parallel sidecars in the same project, assign a stable task ID:

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --cd "$PROJECT" \
  --task-id review-auth \
  "Review the auth module. Report findings with file and line references."
```

### 2. Check session status before follow-ups

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --cd "$PROJECT" --status
```

Check a named task:

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --cd "$PROJECT" \
  --task-id review-auth --status
```

List recorded sessions for the project:

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --cd "$PROJECT" --list
```

| Status | Action |
| ------ | ------ |
| `running` | Wait. Never resume a running session concurrently. |
| `idle` | Continue with `--resume` (step 3). |
| `missing` | No recorded session for this directory. Start new (step 1). |
| `untracked` | Session ID is known but lacks a workdir record. Resume with `--session-id <UUID> --cd <DIR>`. |

### 3. Resume an idle session

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --cd "$PROJECT" --resume \
  "Continue with existing context; run verification and report results."
```

Resume a named task:

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --cd "$PROJECT" \
  --task-id review-auth --resume \
  "Continue with existing context; run verification and report results."
```

Resume a specific session by ID from any directory:

```bash
"<path-to-skill>/scripts/codex-deepseek-sidecar" --session-id <UUID> \
  "Continue the task and report results."
```

## Flags

| Flag | Effect |
| ---- | ------ |
| `--profile NAME` | Codex profile (default `deepseek`). |
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
  "<path-to-skill>/scripts/codex-deepseek-sidecar" \
  --cd "$PROJECT" --pass-env API_KEY "Run specified tests and report output."
```

## Prompt shape

Give the sidecar a self-contained brief:

```
Task: [concrete, bounded goal]
Context: [paths, error snippet, constraints]
Expected: [conclusion, evidence, patch/test references if code was edited]
Constraints: [read-only scope, or explicit permission to edit/run tests]
```

Use the Terminal `deepseek>` prompt only after the sidecar's initial task completes. The session persists and the window remains interactive.

This skill only provides sidecar session execution and lifecycle controls. The calling agent is responsible for task decomposition, coordination, result synthesis, and deciding whether edits are allowed. For multiple concurrent sidecars in one repository, prefer `--task-id` or the returned `--session-id`; do not rely on bare `--resume --cd` because it resumes the latest recorded session for that workdir.

## Error recovery

- **`tokio-runtime-worker` panic** or **`JoinError::Panic`** → infrastructure failure. Re-run with `--verbose-stderr` for diagnosis.
- **`connectivity check failed`** → profile is misconfigured or provider is unreachable. Run `--configure` to repair, or use `--no-doctor-check` if the check is overly strict (e.g., proxy returns 404 on the probe route).
- **`no previous session found`** → no recorded session for this workdir. Start a new session without `--resume`.
