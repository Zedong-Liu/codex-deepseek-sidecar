# Contributing

Thanks for improving `codex-deepseek-sidecar`.

## Development

This repository is intentionally small. Keep changes focused on the sidecar wrapper, Codex skill metadata, and documentation.

Before opening a pull request:

```bash
bash -n scripts/codex-deepseek-sidecar
bash -n scripts/codex-deepseek-subagent
bash -n scripts/terminal-chat
bash -n .opencode/scripts/deepseek-sidecar
bash -n .claude-plugin/scripts/deepseek-sidecar
python3 -m py_compile scripts/deepseek-responses-proxy
scripts/deepseek-responses-proxy --self-test
```

If you have `shellcheck` installed, also run:

```bash
shellcheck scripts/codex-deepseek-sidecar scripts/codex-deepseek-subagent scripts/terminal-chat \
  .opencode/scripts/deepseek-sidecar .claude-plugin/scripts/deepseek-sidecar
```

## Design Principles

- Keep the main agent responsible for planning and synthesis.
- Keep this project responsible for sidecar session execution and lifecycle controls.
- Prefer small flags over broad framework behavior.
- Preserve backward-compatible behavior unless there is a clear reason to break it.
- Avoid passing host environment variables by default; use `--pass-env` for explicit needs.

## Pull Requests

Please include:

- What changed.
- Why it changed.
- How you tested it.
- Any compatibility notes for existing users.
