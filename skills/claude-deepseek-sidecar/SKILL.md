---
name: claude-deepseek-sidecar
description: Use when Claude Code should delegate a bounded task through this repository's Claude-specific sidecar adapter.
version: 0.1.0
---

# Claude DeepSeek Sidecar

Use this skill only inside Claude Code. It launches Claude Code as the worker
harness and leaves model/provider configuration to Claude Code itself.

## Command

From the repository root:

```bash
.claude-plugin/scripts/deepseek-sidecar --cd "$PROJECT" \
  "Task: [bounded task]. Expected: [short report]. Constraints: [scope]."
```

For API-key based provider setups:

```bash
.claude-plugin/scripts/deepseek-sidecar --api-key-file ~/.claude/deepseek.key \
  --base-url "<anthropic-compatible-base-url>" \
  --cd "$PROJECT" "Task: inspect logs. Expected: root cause with evidence."
```

This adapter does not use the Codex Responses proxy. If Claude Code supports
the target DeepSeek-compatible provider directly, prefer Claude Code settings or
environment variables over an extra proxy layer.
