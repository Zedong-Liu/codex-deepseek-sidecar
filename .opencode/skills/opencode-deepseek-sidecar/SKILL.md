---
name: opencode-deepseek-sidecar
description: Use when OpenCode should delegate a bounded task to a DeepSeek-backed OpenCode worker through this repository's OpenCode adapter.
---

# OpenCode DeepSeek Sidecar

Use this skill to run a self-contained OpenCode sidecar task with DeepSeek.
Keep this separate from the Codex skill entrypoint.

## Command

From the repository root:

```bash
.opencode/scripts/deepseek-sidecar --cd "$PROJECT" \
  --model deepseek/deepseek-v4-pro \
  "Task: [bounded task]. Expected: [short report]. Constraints: [scope]."
```

If the built-in local proxy is used, keep it running first:

```bash
DEEPSEEK_API_KEY="sk-..." scripts/deepseek-responses-proxy
```

OpenCode uses OpenAI-compatible chat completions here. It does not use Codex
profiles and does not need the Codex Responses translation path.

Use `--api-key-file` if the key should be read from a private local file:

```bash
.opencode/scripts/deepseek-sidecar --api-key-file ~/.codex/deepseek-sidecar.key \
  --cd "$PROJECT" "Task: inspect logs. Expected: root cause with evidence."
```
