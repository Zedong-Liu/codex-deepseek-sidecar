# codex-deepseek-sidecar

[English](README.md) | [中文](README.zh-CN.md)

在你的 Codex 主 agent 旁边，运行一个由 DeepSeek 驱动的 Codex sidecar。

**不需要额外代理。带上你自己的 DeepSeek key 就能跑。**

`codex-deepseek-sidecar` 是一个 Codex skill，用来把边界清晰的任务委派给 DeepSeek 后端的 Codex CLI session。让高级 GPT 模型继续做主大脑，负责规划、判断和汇总；把并行探索、代码审查、调试、日志排查、实现尝试交给成本更低但依然聪明的 DeepSeek 工人，同时继续使用 Codex harness。

项目内置一个很小的 Python 代理，把 Codex Responses API 桥接到 DeepSeek Chat Completions API。新机器只需要 Python 3 和 `DEEPSEEK_API_KEY`。如果你已经在用 VibeAround 或其他兼容 provider，也可以继续用原来的。

如果你是人类安装者：把这个仓库 clone 成 Codex skill，然后告诉你的 Codex agent 使用 `codex-deepseek-sidecar`。这个 skill 主要是写给 agent 执行的，所以 Codex 可以自己配置 profile、按需启动代理，并在大约五分钟内拉起第一个 sidecar。

## 为什么值得

下面价格按每 1M tokens 计算，基于 2026-06-02 查询到的 [OpenAI GPT-5.5 model page](https://developers.openai.com/api/docs/models/gpt-5.5/) 和 [DeepSeek pricing docs](https://api-docs.deepseek.com/quick_start/pricing)。DeepSeek Pro 输入价格使用 cache-miss 价格，属于偏保守比较。

| 模型 | 最适合的角色 | 输入 | 输出 | 原始价差 |
| ---- | ---- | ----: | ----: | ----: |
| GPT-5.5 | 主大脑：规划、判断、综合结论 | $5.00 | $30.00 | 1x |
| DeepSeek V4 Pro | 强工人：review、调试、实现尝试 | $0.435 | $0.87 | 混合后约便宜 13x |
| DeepSeek V4 Flash | 快工人：日志、广泛探索、廉价并行 pass | $0.14 | $0.28 | 混合后约便宜 39x |

按 `1M input + 200K output` 粗略混合计算：

| 路由方式 | 近似成本 | token 成本降低 |
| ---- | ----: | ----: |
| 全部使用 GPT-5.5 | $11.00 | baseline |
| 工人 token 全部使用 DeepSeek V4 Pro | $0.61 | 约 94% |
| 工人 token 全部使用 DeepSeek V4 Flash | $0.20 | 约 98% |

真实 sidecar 工作流里，GPT 仍然会花 token 做协调、审查和最终决策。这正是这个模式的重点：把昂贵 token 花在判断力上，把重复性的 worker-token 预算迁移到 DeepSeek。对很多 agent 工作流来说，**80-90% 的 token cost save** 是一个现实目标，同时仍然保留 Codex harness。

- **把模型用在合适的位置**：GPT 专注于规划和综合判断，DeepSeek 处理边界清晰的工人任务。
- **继续使用 Codex harness**：sidecar 仍然跑在 Codex CLI 里，可以读文件、执行命令、保持 session，并输出可验证证据。
- **更适合并行**：用 `--task-id` 命名 sidecar，支持列出、查状态、恢复指定工人，避免恢复错 session。
- **不需要外部代理**：内置 `deepseek-responses-proxy`，本地桥接 Codex Responses 请求。
- **过程可观察**：每次运行都可以打开 Terminal 监控窗口，任务完成后变成可继续追问的交互 prompt。

## 功能

- 启动 DeepSeek 后端的 Codex CLI sidecar session。
- 用一个 Python stdlib 小代理把 Codex Responses 请求转换成 DeepSeek Chat Completions。
- 按项目和可选 `task-id` 记录可复用 session。
- 支持 `--status`、`--list`、`--resume` 和直接 `--session-id` 恢复。
- 使用干净的 UTF-8 环境运行子进程，并通过 `--pass-env` 显式传递环境变量。
- 用 per-session lock 防止同一个 session 被并发 resume。
- 保留兼容入口 `scripts/codex-deepseek-subagent`。

## 人类安装方式

把仓库交给 Codex，作为一个 skill：

```bash
git clone https://github.com/Zedong-Liu/codex-deepseek-sidecar.git \
  ~/.codex/skills/codex-deepseek-sidecar
cd ~/.codex/skills/codex-deepseek-sidecar
chmod +x scripts/*
```

然后告诉你的 Codex agent：

```text
使用 codex-deepseek-sidecar skill。我有 DeepSeek API key。
如果需要，请配置本地代理/profile，然后为当前仓库启动一个 sidecar。
```

要求：已安装并登录 Codex CLI、Python 3、一个 DeepSeek API key。macOS 更适合使用 Terminal 监控窗口。

## Agent 启动方式

如果本地还没有 DeepSeek-compatible Codex provider，先启动内置轻量代理：

```bash
export DEEPSEEK_API_KEY="sk-..."
~/.codex/skills/codex-deepseek-sidecar/scripts/deepseek-responses-proxy
```

让它保持运行。再次启动前，可以先检查代理是否已经活着：

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/deepseek-responses-proxy --status
```

在另一个终端里配置 Codex。这个配置是一次性的，不要每次 sidecar 任务都重复配置：

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar --configure
```

生成的 provider 会让 Codex 指向：

```toml
[model_providers.deepseek]
base_url = "http://127.0.0.1:12359/v1"
wire_api = "responses"
```

这个代理使用自己的端口，不复用 VibeAround 的端口。默认支持较大的请求体（`--max-body-mb 128`），避免长上下文 sidecar 因为代理 buffer 太小被卡住。它会桥接 function tools，并忽略 Codex 默认附带但 DeepSeek Chat 不支持的 Responses built-in tools；如果请求明确要求某个不支持的 built-in tool，则返回明确错误。

## 快速开始

委派一个任务：

```bash
PROJECT="/absolute/path/to/project"

~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  --cd "$PROJECT" \
  --task-id review-auth \
  "Review the auth module. Report findings with file and line references. Do not edit files."
```

查看状态：

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  --cd "$PROJECT" --task-id review-auth --status
```

列出当前项目的 sidecar：

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  --cd "$PROJECT" --list
```

恢复指定工人：

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  --cd "$PROJECT" --task-id review-auth --resume \
  "Continue from your previous findings and verify the highest-risk issue."
```

## Organizer 模式

这个项目刻意保持小而稳定。它不试图教主 agent 如何做 organizer。调用方仍然负责拆任务、分配权限、处理冲突和综合结论。

推荐的并行工作流：

1. 给每个工人一个稳定的 `--task-id`，例如 `review-auth`、`inspect-tests`、`bench-sro`。
2. prompt 写成自包含 brief：任务、上下文、期望输出、是否允许改文件。
3. follow-up 前先用 `--list` 和 `--status`。
4. 日常恢复使用 `--task-id`。
5. `--session-id` 只作为故障恢复兜底。

## 参数

| 参数 | 用途 |
| ---- | ---- |
| `--cd DIR` | 用于 session 记录的项目目录。 |
| `--task-id NAME` | 在 `--cd DIR` 内命名一个 sidecar 任务，用于查状态和恢复。 |
| `--resume`, `-r` | 继续一个已记录的 session。和 `--task-id` 一起使用时恢复指定任务。 |
| `--session-id UUID` | 直接恢复某个 Codex session，主要用于故障恢复。 |
| `--status` | 输出 `running`、`idle`、`missing` 或 `untracked`，不发起模型请求。 |
| `--list` | 列出 `--cd DIR` 下记录过的 sidecar session，不发起模型请求。 |
| `--no-monitor` | 不打开 Terminal 监控窗口，适合自动化。 |
| `--pass-env NAME` | 向隔离子进程传递一个 UTF-8 环境变量，可重复使用。 |
| `--profile NAME` | 使用非默认 Codex profile，默认是 `deepseek`。 |
| `--json` | 请求 Codex JSONL 事件输出。 |
| `--no-doctor-check` | 跳过启动前的 `codex doctor` 检查。 |

## Prompt 形状

```text
Task: [concrete, bounded goal]
Context: [paths, error snippet, constraints]
Expected: [conclusion, evidence, patch/test references if code was edited]
Constraints: [read-only scope, or explicit permission to edit/run tests]
```

## 仓库结构

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

Apache-2.0，见 [LICENSE](LICENSE)。
