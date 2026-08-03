---
name: deepseek-codex-subagent
description: Delegate bounded side tasks to a DeepSeek-backed Codex CLI subagent through the official DeepSeek API, persistent sessions, and an interactive Terminal monitor.
---

# DeepSeek Codex Subagent

Use the stable entrypoint; never handwrite a `codex exec` invocation:

```bash
<skill-dir>/scripts/codex-deepseek-subagent
```

It forwards to the official-API wrapper, which uses a local Responses-to-Chat
bridge on `127.0.0.1:12359`. It does not use VibeAround or another third-party
gateway.

## Installation and safe setup

```bash
git clone <repo-url> ~/.codex/skills/deepseek-codex-subagent
cd ~/.codex/skills/deepseek-codex-subagent
chmod +x scripts/*
ln -sf "$PWD/scripts/codex-deepseek-subagent" ~/.codex/bin/codex-deepseek-subagent
```

Start the local proxy with a user-managed secret source. Do not put an API key
in a repository, prompt, shell history, or profile file. For development:

```bash
DEEPSEEK_API_KEY="..." scripts/deepseek-responses-proxy
```

For login startup, use a user LaunchAgent or secret manager. If the agent uses
a non-default launchd label, set `DEEPSEEK_SIDECAR_LAUNCHD_LABEL` before running
the wrapper. Then generate/validate the profiles:

```bash
scripts/codex-deepseek-subagent --configure
curl -fsS http://127.0.0.1:12359/v1/ready
```

`--configure` refreshes a local model catalog from the installed Codex cache;
run it after a Codex upgrade.

## Profiles and effort

| Profile | Model / route | Use |
| --- | --- | --- |
| `ds-sidecar-local` | `deepseek-v4-pro`, stable API | Default for complex coding and correctness-sensitive work. |
| `ds-sidecar-flash` | `deepseek-v4-flash`, stable API | Fast, low-cost bounded investigation. |
| `ds-sidecar-beta` | `deepseek-v4-pro`, `/beta` API | Opt-in only for beta strict function schemas. |

Use Flash `high` for clear, bounded work. Use Flash `max` only for a limited
but branching diagnosis or small implementation. Start a new Pro session if the
scope becomes cross-cutting, long-context, or high risk.

```bash
PROJECT="/absolute/path/to/project"

# Fast inspection
scripts/codex-deepseek-subagent --profile ds-sidecar-flash --effort high \
  --cd "$PROJECT" "Inspect the failing test and report evidence. Do not edit."

# Bounded multi-file diagnosis
scripts/codex-deepseek-subagent --profile ds-sidecar-flash --effort max \
  --cd "$PROJECT" "Find the root cause, implement the smallest fix, and run its tests."
```

## Session workflow

Every interactive execution opens a Terminal view with a wrapper-verified
model, effective effort, profile, project, live output, and a Readline-backed
`deepseek >` prompt after completion. The displayed route—not a model
self-report—is authoritative.

```bash
# Start a named session.
scripts/codex-deepseek-subagent --cd "$PROJECT" --task-id review-auth \
  "Review the auth module. Report files, evidence, and next steps."

# Check without a model request.
scripts/codex-deepseek-subagent --cd "$PROJECT" --task-id review-auth --status

# Resume an idle session. Its recorded profile and effort are restored unless
# explicitly overridden.
scripts/codex-deepseek-subagent --cd "$PROJECT" --task-id review-auth --resume \
  "Continue from the existing evidence and report the result."
```

Use `/exit` in Terminal to leave a session idle. Use `--no-monitor` only for
automation. Session mappings are stored under
`~/.codex/state/deepseek-sidecar/` with user-only permissions and survive
reboot. Never resume one session concurrently.

## Safety and transport boundary

- Use the official route: `127.0.0.1:12359 -> https://api.deepseek.com`.
- The proxy supports text, function tools, streaming, cache usage, and thinking
  continuation for tool-call chains. It intentionally drops image data URIs;
  describe image findings in text instead of forwarding raw/base64 image input.
- V4 Flash can serialize a tool call as native DSML text rather than an API
  `tool_calls` field. The proxy converts both single- and double-delimiter DSML
  blocks into structured function calls before Codex sees them; streaming text
  is held until that check completes, so a tool request is never mistaken for a
  final assistant answer.
- Raw reasoning is hidden from Terminal output. It is preserved only when a
  tool-call continuation requires it; ordinary chat reasoning is discarded so
  it cannot inflate later prompts.
- Built-in hosted Responses tools are unsupported. Function tools are the
  supported bridge surface.
- Pass a task variable only when needed: `--pass-env NAME`. Do not pass the
  DeepSeek credential to delegated processes.

## Health and upgrade check

```bash
curl -fsS http://127.0.0.1:12359/v1/ready
curl -fsS http://127.0.0.1:12359/v1/metrics
scripts/codex-deepseek-subagent --configure
scripts/codex-deepseek-subagent --cd "$PROJECT" --task-id sidecar-upgrade-smoke \
  --no-monitor "Reply exactly SIDECAR_OK. Do not use tools."
```

`/metrics` contains aggregate cache hit/miss counts only, never prompt, tool,
or credential data. Cache matching is best effort and requires a repeated
prefix; use a new session when an old diagnostic session has accumulated
unrelated history.
