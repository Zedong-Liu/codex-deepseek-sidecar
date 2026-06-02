# codex-deepseek-sidecar

[English](README.md) | [中文](README.zh-CN.md)

Run a DeepSeek-powered Codex sidecar next to your main Codex agent.

`codex-deepseek-sidecar` is a Codex skill and shell wrapper for delegating bounded tasks to a DeepSeek-backed Codex CLI session. The pattern is simple: keep a high-end GPT model as the main brain, then hand off parallel research, review, debugging, log inspection, or implementation attempts to lower-cost but still capable DeepSeek workers, all inside the Codex harness.

This is a cost-effective agent-era workflow: premium reasoning coordinates the work, economical sidecars do the heavy lifting, and Codex provides the sandbox, tools, session persistence, and command execution layer.

It also includes a small Python proxy that bridges Codex's Responses API wire format to DeepSeek's Chat Completions API, so users do not need a full provider switcher just to get started.

## The Cost Shape

Agent work burns tokens in a very specific way: lots of file reading, log scanning, test output, repeated attempts, and long follow-up context. That is exactly the work you do not always want to run on the most expensive frontier model.

Let the premium model be the **architect** and let DeepSeek sidecars be the **crew**.

Prices below are per 1M tokens, checked on 2026-06-02 from the [OpenAI GPT-5.5 model page](https://developers.openai.com/api/docs/models/gpt-5.5/) and [DeepSeek pricing docs](https://api-docs.deepseek.com/quick_start/pricing). DeepSeek Pro input uses the cache-miss price for a conservative comparison.

| Model | Best role in this workflow | Input | Output | Raw price gap vs GPT-5.5 |
| ---- | ---- | ----: | ----: | ----: |
| GPT-5.5 | Main brain: planning, judgment, synthesis | $5.00 | $30.00 | 1x |
| DeepSeek V4 Pro | Strong worker: code review, debugging, implementation attempts | $0.435 | $0.87 | ~13x cheaper blended |
| DeepSeek V4 Flash | Fast worker: search, logs, broad exploration, cheap parallel passes | $0.14 | $0.28 | ~39x cheaper blended |

Example blended at `1M input + 200K output`:

| Route | Approx cost | Token-cost reduction |
| ---- | ----: | ----: |
| All GPT-5.5 | $11.00 | baseline |
| All DeepSeek V4 Pro worker tokens | $0.61 | ~94% |
| All DeepSeek V4 Flash worker tokens | $0.20 | ~98% |

In a real sidecar setup, GPT still spends tokens on coordination, review, and final decisions. That is the point: spend premium tokens where judgment matters, and move the repetitive worker-token budget to DeepSeek. For many agent workflows, that makes an **80-90% token-cost reduction** a realistic target without giving up Codex's harness.

## Why It Exists

- **Use the right model for the right job**: GPT can stay focused on planning and synthesis while DeepSeek handles bounded worker tasks.
- **Keep Codex's harness**: sidecars still run through Codex CLI, so they can inspect files, run commands, use persistent sessions, and report evidence.
- **Parallelize safely**: name sidecar tasks with `--task-id`, list them, check status, and resume the right worker without guessing.
- **Reduce cost without giving up intelligence**: delegate repetitive exploration, reviews, and verification to a cheaper model while keeping a stronger model in charge.
- **Stay observable**: by default each run opens a Terminal monitor, then turns into an interactive follow-up prompt.
- **Bring your own DeepSeek key**: the included `deepseek-responses-proxy` is a lightweight local bridge, not a full provider platform.

## What It Does

- Starts DeepSeek-backed Codex CLI sidecar sessions.
- Bridges Codex Responses requests to DeepSeek Chat Completions with a small Python stdlib proxy.
- Records reusable session IDs by project and optional task ID.
- Supports `--status`, `--list`, `--resume`, and direct `--session-id` recovery.
- Runs children in a clean UTF-8 environment with explicit `--pass-env` forwarding.
- Prevents concurrent resume of the same session with a per-session lock.
- Keeps a compatibility entrypoint at `scripts/codex-deepseek-subagent`.

## Requirements

- macOS for the Terminal monitor window.
- [Codex CLI](https://github.com/openai/codex) installed and authenticated.
- Python 3.
- A DeepSeek API key in `DEEPSEEK_API_KEY`.
- A DeepSeek-compatible Codex profile, usually named `deepseek`. The `--configure` command can create one for the included local proxy.

## Install

Clone this repository as a Codex skill:

```bash
git clone https://github.com/Zedong-Liu/codex-deepseek-sidecar.git \
  ~/.codex/skills/codex-deepseek-sidecar
cd ~/.codex/skills/codex-deepseek-sidecar
chmod +x scripts/*
```

Optional command shortcut:

```bash
mkdir -p ~/.codex/bin
ln -s ~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  ~/.codex/bin/codex-deepseek-sidecar
```

Configure or verify the DeepSeek profile:

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar --configure
```

## 5-Minute Setup

Start the lightweight local proxy if you do not already have a DeepSeek-compatible Codex provider or local proxy:

```bash
export DEEPSEEK_API_KEY="sk-..."
~/.codex/skills/codex-deepseek-sidecar/scripts/deepseek-responses-proxy
```

Leave it running. Before starting another copy, check whether it is already alive:

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/deepseek-responses-proxy --status
```

In another terminal, configure Codex for the proxy. This is a one-time setup; do not repeat it for every sidecar task:

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar --configure
```

The generated provider points Codex at:

```toml
[model_providers.deepseek]
base_url = "http://127.0.0.1:12359/v1"
wire_api = "responses"
```

The proxy listens on its own port instead of reusing VibeAround's port. It accepts large request bodies by default (`--max-body-mb 128`) so long-context sidecar runs are not capped by a small proxy buffer. It bridges function tools and ignores unsupported Responses built-in tools that Codex may attach by default; if a request explicitly requires an unsupported built-in tool, it returns a clear error.

## Quick Start

Delegate one task:

```bash
PROJECT="/absolute/path/to/project"

~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  --cd "$PROJECT" \
  --task-id review-auth \
  "Review the auth module. Report findings with file and line references. Do not edit files."
```

Check the worker:

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  --cd "$PROJECT" --task-id review-auth --status
```

List sidecars for a project:

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  --cd "$PROJECT" --list
```

Resume the named worker:

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  --cd "$PROJECT" --task-id review-auth --resume \
  "Continue from your previous findings and verify the highest-risk issue."
```

## Organizer Pattern

This project intentionally stays small. It does not try to teach the main agent how to be an organizer. The calling agent remains responsible for task decomposition, permissions, conflict management, and synthesis.

Recommended parallel workflow:

1. Give each worker a stable `--task-id`, such as `review-auth`, `inspect-tests`, or `bench-sro`.
2. Make prompts self-contained: task, context, expected output, and whether edits are allowed.
3. Use `--list` and `--status` before follow-ups.
4. Resume by `--task-id` for normal operation.
5. Use `--session-id` only as a recovery escape hatch.

## Flags

| Flag | Purpose |
| ---- | ------- |
| `--cd DIR` | Project directory for session tracking. |
| `--task-id NAME` | Name a sidecar task within `--cd DIR` for status/resume lookup. |
| `--resume`, `-r` | Continue a recorded session. With `--task-id`, resumes that named task. |
| `--session-id UUID` | Resume a specific Codex session directly. Useful for recovery. |
| `--status` | Report `running`, `idle`, `missing`, or `untracked`. No model request. |
| `--list` | List recorded sidecar sessions for `--cd DIR`. No model request. |
| `--no-monitor` | Skip the Terminal monitor window. Useful for automation. |
| `--pass-env NAME` | Pass one UTF-8 environment variable into the isolated child. Repeatable. |
| `--profile NAME` | Use a non-default Codex profile. Default: `deepseek`. |
| `--json` | Request Codex JSONL event output. |
| `--no-doctor-check` | Skip the pre-flight `codex doctor` check. |

## Prompt Shape

```text
Task: [concrete, bounded goal]
Context: [paths, error snippet, constraints]
Expected: [conclusion, evidence, patch/test references if code was edited]
Constraints: [read-only scope, or explicit permission to edit/run tests]
```

## Repository Layout

```text
.
├── SKILL.md
├── agents/openai.yaml
├── scripts/codex-deepseek-sidecar
├── scripts/codex-deepseek-subagent
├── scripts/deepseek-responses-proxy
└── scripts/terminal-chat
```

## License

Apache-2.0. See [LICENSE](LICENSE).
