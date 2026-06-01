# Security Policy

## Supported Versions

The default branch is the supported version.

## Reporting a Vulnerability

Please report security issues privately through GitHub's private vulnerability reporting if enabled, or by opening a minimal issue that does not include exploitable secrets or credentials.

## Security Notes

`codex-deepseek-sidecar` launches Codex CLI sessions that may inspect files and run commands in the selected project directory. Treat sidecar prompts like delegated terminal work:

- Use read-only prompts for exploration and review tasks.
- Pass secrets only when required, and only with `--pass-env NAME`.
- Prefer `--task-id` or `--session-id` when resuming sidecars so the intended session is resumed.
- Review any code changes produced by a sidecar before committing them.
