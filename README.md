# codex-deepseek-sidecar

[English](README.md) | [中文](README.zh-CN.md)

<p align="center">
  <img src="assets/codex-deepseek-sidecar-logo.png" alt="codex-deepseek-sidecar cyber logo" width="360">
</p>

Launch DeepSeek sidecar agents from Codex with one prompt.

**No external proxy required. Bring your own DeepSeek key. Codex handles the rest. 🚀**

`codex-deepseek-sidecar` is a Codex skill that lets your main Codex agent start cheaper DeepSeek-backed worker agents for bounded side tasks: long tests, log analysis, broad code exploration, independent review, or implementation attempts.

The point is simple: keep a premium GPT model as the main brain, let DeepSeek workers do the repetitive token-heavy work, and keep everything inside the Codex harness.

## One-Prompt Setup

Give Codex this repository URL and this prompt:

```text
Install and use https://github.com/Zedong-Liu/codex-deepseek-sidecar.
I have a DeepSeek API key; ask me for it if it is not already configured.
Configure the local proxy/profile if needed.
Then start a DeepSeek sidecar for this repo and use it for suitable long-running or log-heavy tasks.
```

That is the intended human workflow. You do not need to learn the sidecar flags, session IDs, or proxy commands. The skill is written for Codex to read and operate. On a fresh machine with Codex CLI, Python 3, and a DeepSeek key, Codex should be able to install the skill, configure the built-in proxy, and launch the first sidecar in about five minutes.

After setup, use natural prompts:

```text
Use a DeepSeek sidecar to run the slow tests while you inspect the code.
```

```text
Use a DeepSeek sidecar to analyze this CI log and summarize the real failure.
```

```text
You decide when to split this work into DeepSeek sidecars. Use them for long tests, log analysis, or broad exploration, organize the results, and give me one final plan.
```

## Why People Want It

- **Spend far less on worker tokens**: move repetitive file reading, logs, tests, and broad exploration from premium GPT tokens to DeepSeek worker tokens. Many workflows can target **80-90% lower token cost**.
- **One prompt, agent does the wiring**: Codex installs, configures, starts, assigns, checks, resumes, and summarizes.
- **Resume instead of restarting**: sidecar sessions persist, so Codex can continue the right worker after a long test, interrupted investigation, or follow-up task.
- **No external proxy required**: the repo includes a small Python proxy from Codex Responses API to DeepSeek Chat Completions.
- **Bring your own DeepSeek key**: no hosted middle layer is required.
- **GPT stays in charge**: the expensive model plans, judges, and synthesizes; DeepSeek handles bounded worker tasks. ⚡️
- **Codex harness stays intact**: sidecars still get Codex file access, command execution, sessions, and evidence-based reporting.

## Cost Shape

Prices below are per 1M tokens, checked on 2026-06-02 from the [OpenAI GPT-5.5 model page](https://developers.openai.com/api/docs/models/gpt-5.5/) and [DeepSeek pricing docs](https://api-docs.deepseek.com/quick_start/pricing). DeepSeek Pro input uses the cache-miss price for a conservative comparison.

| Model | Best role | Input | Output |
| ---- | ---- | ----: | ----: |
| GPT-5.5 | Main brain: planning, judgment, synthesis | $5.00 | $30.00 |
| DeepSeek V4 Pro | Strong worker: review, debugging, implementation attempts | $0.435 | $0.87 |
| DeepSeek V4 Flash | Fast worker: logs, broad exploration, cheap parallel passes | $0.14 | $0.28 |

Example at `1M input + 200K output`:

| Route | Approx cost | Token-cost reduction |
| ---- | ----: | ----: |
| All GPT-5.5 | $11.00 | baseline |
| DeepSeek V4 Pro worker tokens | $0.61 | ~94% |
| DeepSeek V4 Flash worker tokens | $0.20 | ~98% |

In real use, GPT still spends tokens coordinating and reviewing. That is the design: spend premium tokens where judgment matters and move repetitive worker-token budget to DeepSeek. For many agent workflows, **80-90% token-cost reduction** is a realistic target.

## What Codex Does Behind the Scenes

When asked to use this skill, Codex can:

- Install the repository as a Codex skill.
- Start the built-in lightweight proxy if no compatible DeepSeek provider exists.
- Configure a Codex profile once, without repeating setup for every task.
- Launch DeepSeek sidecars for self-contained tasks.
- Track task IDs and sessions so follow-ups resume the right worker.
- Collect sidecar findings and synthesize them into the main answer.

The operational details are intentionally kept out of this human README. Agent-facing instructions live in [SKILL.md](SKILL.md).

## Built-In Proxy

The included `deepseek-responses-proxy` is intentionally small: Python stdlib only, local-only by default, and designed for large Codex request bodies. It bridges function tools and ignores unsupported Responses built-in tools that Codex may attach by default. If a request explicitly requires an unsupported built-in tool, it returns a clear error.

If you already use VibeAround or another compatible provider, Codex can keep using that instead.

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
