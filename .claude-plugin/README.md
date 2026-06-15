# Claude Code Adapter

This directory contains the Claude Code-facing adapter for
`codex-deepseek-sidecar`. It is intentionally separate from the Codex skill
entrypoint.

Claude Code already owns its model/provider configuration. This adapter does
not translate Codex Responses traffic and does not require
`deepseek-responses-proxy`.

## Quick Start

Configure Claude Code for your DeepSeek or Anthropic-compatible provider, then
run:

```bash
.claude-plugin/scripts/deepseek-sidecar --cd "$PWD" \
  --model "<claude-or-provider-model>" \
  "Task: inspect the failing tests and report the first failure."
```

For API-key based setups:

```bash
.claude-plugin/scripts/deepseek-sidecar --api-key-file ~/.claude/deepseek.key \
  --base-url "<anthropic-compatible-base-url>" \
  --cd "$PWD" "Task: inspect logs. Expected: root cause with evidence."
```

Use Claude Code settings for provider-specific details whenever possible.
