---
name: deepseek-codex-subagent
description: Delegate bounded side tasks to a DeepSeek-backed Codex CLI subagent through the official DeepSeek API, persistent sessions, and an interactive Terminal monitor.
---

# DeepSeek Codex Subagent

Use the stable entrypoint; never handwrite a `codex exec` invocation:

```bash
<skill-dir>/scripts/codex-deepseek-subagent
```

It forwards to the official-API wrapper. The default Flash profile talks
directly to DeepSeek's native Responses API at `https://api.deepseek.com/` with
a Keychain-backed credential; only the Pro/Beta profiles still use the local
`127.0.0.1:12359` bridge until DeepSeek officially enables Pro for Codex. It
never uses VibeAround or another third-party gateway.

## Installation and safe setup

```bash
git clone <repo-url> ~/.codex/skills/deepseek-codex-subagent
cd ~/.codex/skills/deepseek-codex-subagent
chmod +x scripts/*
ln -sf "$PWD/scripts/codex-deepseek-subagent" ~/.codex/bin/codex-deepseek-subagent
```

Store the DeepSeek API key in the macOS Keychain item
`codex-deepseek-official` for the current user (or export `DEEPSEEK_API_KEY` /
`DEEPSEEK_API_KEY_FILE` on other platforms). Do not put an API key in a
repository, prompt, shell history, or profile file. Then generate/validate the
profiles:

```bash
scripts/codex-deepseek-subagent --configure
```

The Flash profile's provider uses Codex's official `auth.command` support to
fetch the key from the Keychain on demand, so no secret is written to
`config.toml`. `--configure` never modifies the top-level model/auth keys of
`~/.codex/config.toml`, so your ChatGPT/Codex login state is untouched.

Do **not** run DeepSeek's one-click setup script
(`bash <(curl -fsSL https://cdn.deepseek.com/api-docs/codex-deepseek-setup.sh)`)
against your real `~/.codex`: it rewrites the global `model` /
`model_provider` / auth keys and hides the ChatGPT session group. This skill
implements the same official provider settings inside its own profile files
instead.

The Pro/Beta profiles still need the launchd proxy. For login startup use the
user LaunchAgent (`com.captainliu.deepseek-sidecar-proxy`); if the agent uses a
non-default launchd label, set `DEEPSEEK_SIDECAR_LAUNCHD_LABEL` before running
the wrapper. Readiness:

```bash
curl -fsS http://127.0.0.1:12359/v1/ready
```

`--configure` refreshes a local model catalog from the installed Codex cache;
run it after a Codex upgrade. The Flash native route checks
`https://api.deepseek.com/models` directly instead of the proxy `/ready`.

## Profiles and effort

| Profile | Model / route | Use |
| --- | --- | --- |
| `ds-sidecar-flash` | `deepseek-v4-flash`, official native Responses API, effort `max` | **Default.** Bounded coding, diagnosis, and low-cost worker tasks. |
| `ds-sidecar-local` | `deepseek-v4-pro`, local proxy bridge | Opt-in for correctness-sensitive or long-horizon work (legacy until DeepSeek enables Pro for Codex). |
| `ds-sidecar-beta` | `deepseek-v4-pro`, local proxy `/beta` | Opt-in only for beta strict function schemas. |

Default is Flash + `max`. Override with `--effort high` for cheaper/faster turns,
or `--profile ds-sidecar-local` when Pro is warranted.

```bash
PROJECT="/absolute/path/to/project"

# Default: Flash max
scripts/codex-deepseek-subagent --cd "$PROJECT" \
  "Find the root cause, implement the smallest fix, and run its tests."

# Cheaper Flash turn
scripts/codex-deepseek-subagent --effort high --cd "$PROJECT" \
  "Inspect the failing test and report evidence. Do not edit."

# Escalate to Pro
scripts/codex-deepseek-subagent --profile ds-sidecar-local --cd "$PROJECT" \
  "Cross-module fix with correctness checks."
```

## Session workflow

Every interactive execution opens a Terminal view with a wrapper-verified
model, effective effort, profile, project, live output, and a Readline-backed
`deepseek >` prompt after completion. The displayed route—not a model
self-report—is authoritative.

Hard rule for the calling agent: spawning a sidecar is not task completion.
Keep the main turn open and monitor with `--status` / `--wait` until the
session leaves `running`. Do not end the turn immediately after a handoff, and
do not treat bare `nohup ... &` as a substitute for waiting.

```bash
# Start a named session.
scripts/codex-deepseek-subagent --cd "$PROJECT" --task-id review-auth \
  "Review the auth module. Report files, evidence, and next steps."

# Check without a model request.
scripts/codex-deepseek-subagent --cd "$PROJECT" --task-id review-auth --status

# Block until an already-running named task becomes idle.
scripts/codex-deepseek-subagent --cd "$PROJECT" --task-id review-auth --wait

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

- Flash uses the official native route:
  `Codex -> https://api.deepseek.com/` (`wire_api = "responses"`), with the key
  supplied by a Keychain-backed `auth.command`. Pro/Beta still use
  `127.0.0.1:12359 -> https://api.deepseek.com`.
- The legacy proxy supports text, function tools, streaming, cache usage, and
  thinking continuation for tool-call chains. It intentionally drops image data
  URIs; describe image findings in text instead of forwarding raw/base64 image
  input.
- On the legacy proxy, V4 Flash can serialize a tool call as native DSML text
  rather than an API `tool_calls` field. The proxy converts both single- and
  double-delimiter DSML blocks into structured function calls before Codex sees
  them; streaming text is held until that check completes. The native Flash
  route returns structured tool calls from the official Responses API.
- Raw reasoning is hidden from Terminal output. It is preserved only when a
  tool-call continuation requires it; ordinary chat reasoning is discarded so
  it cannot inflate later prompts.
- Built-in hosted Responses tools are unsupported; function tools are the
  supported bridge surface on both routes.
- Pass a task variable only when needed: `--pass-env NAME`. Do not pass the
  DeepSeek credential to delegated processes.

## Health and upgrade check

```bash
scripts/codex-deepseek-subagent --configure
scripts/codex-deepseek-subagent --cd "$PROJECT" --task-id sidecar-upgrade-smoke \
  --no-monitor "Reply exactly SIDECAR_OK. Do not use tools."
# Pro/Beta proxy endpoints (Flash native does not use the proxy):
curl -fsS http://127.0.0.1:12359/v1/ready
curl -fsS http://127.0.0.1:12359/v1/metrics
```

`/metrics` contains aggregate cache hit/miss counts only, never prompt, tool,
or credential data. Cache matching is best effort and requires a repeated
prefix; use a new session when an old diagnostic session has accumulated
unrelated history. The native route reports cache usage through the official
Responses API usage fields.
