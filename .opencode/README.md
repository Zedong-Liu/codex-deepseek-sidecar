# OpenCode Adapter

This directory contains the OpenCode-facing adapter for `codex-deepseek-sidecar`.
It is intentionally separate from the Codex skill entrypoint.

OpenCode can discover project-local skills from `.opencode/skills/<name>/SKILL.md`.
Use `opencode-deepseek-sidecar` when you want OpenCode to launch a bounded
DeepSeek worker task from a shell command.

## Quick Start

Start an OpenAI-compatible DeepSeek endpoint first. The repository proxy can be
used because OpenCode speaks chat completions:

```bash
DEEPSEEK_API_KEY="sk-..." scripts/deepseek-responses-proxy
```

Then run:

```bash
.opencode/scripts/deepseek-sidecar --cd "$PWD" \
  --model deepseek/deepseek-v4-pro \
  "Task: inspect the failing tests and report the first failure."
```

This adapter does not use Codex profiles or `wire_api = "responses"`.
