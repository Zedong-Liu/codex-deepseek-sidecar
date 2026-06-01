# codex-deepseek-sidecar

[English](README.md) | [中文](README.zh-CN.md)

在你的 Codex 主 agent 旁边，运行一个由 DeepSeek 驱动的 Codex sidecar。

`codex-deepseek-sidecar` 是一个 Codex skill 和 shell wrapper，用来把边界清晰的任务委派给 DeepSeek 后端的 Codex CLI session。它的工作模式很直接：让高级 GPT 模型继续做主大脑，负责规划、判断和汇总；把并行探索、代码审查、调试、日志排查、实现尝试这类工作交给成本更低但依然聪明的 DeepSeek 工人，同时继续使用 Codex 的 harness。

这是一个非常有性价比的 agent 时代工作方式：昂贵的强模型负责协调，便宜的 sidecar 负责干活，Codex 提供沙箱、工具、持久 session 和命令执行层。

## 为什么需要它

- **把模型用在合适的位置**：GPT 专注于规划和综合判断，DeepSeek 处理边界清晰的工人任务。
- **继续使用 Codex harness**：sidecar 仍然跑在 Codex CLI 里，可以读文件、执行命令、保持 session，并输出可验证证据。
- **更适合并行**：用 `--task-id` 命名 sidecar，支持列出、查状态、恢复指定工人，避免恢复错 session。
- **降低成本但不牺牲太多智能**：把重复探索、review、验证等任务交给更便宜的模型。
- **过程可观察**：默认打开 Terminal 监控窗口，任务完成后变成可继续追问的交互 prompt。

## 功能

- 启动 DeepSeek 后端的 Codex CLI sidecar session。
- 按项目和可选 `task-id` 记录可复用 session。
- 支持 `--status`、`--list`、`--resume` 和直接 `--session-id` 恢复。
- 使用干净的 UTF-8 环境运行子进程，并通过 `--pass-env` 显式传递环境变量。
- 用 per-session lock 防止同一个 session 被并发 resume。
- 保留兼容入口 `scripts/codex-deepseek-subagent`。

## 要求

- macOS，用于 Terminal 监控窗口。
- 已安装并登录 [Codex CLI](https://github.com/openai/codex)。
- 一个可用的 DeepSeek Codex profile，通常命名为 `deepseek`。
- 可选：如果你的 DeepSeek profile 通过本地代理或 provider bridge 路由，需要先启动对应服务。

## 安装

把仓库克隆为 Codex skill：

```bash
git clone https://github.com/Zedong-Liu/codex-deepseek-sidecar.git \
  ~/.codex/skills/codex-deepseek-sidecar
cd ~/.codex/skills/codex-deepseek-sidecar
chmod +x scripts/*
```

可选命令快捷方式：

```bash
mkdir -p ~/.codex/bin
ln -s ~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar \
  ~/.codex/bin/codex-deepseek-sidecar
```

配置或验证 DeepSeek profile：

```bash
~/.codex/skills/codex-deepseek-sidecar/scripts/codex-deepseek-sidecar --configure
```

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
└── scripts/terminal-chat
```

## License

Apache-2.0，见 [LICENSE](LICENSE)。
