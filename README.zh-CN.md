# codex-deepseek-sidecar

[English](README.md) | [中文](README.zh-CN.md)

<p align="center">
  <img src="assets/codex-deepseek-sidecar-logo.png" alt="codex-deepseek-sidecar pixel logo" width="96">
</p>

用一句 prompt，让 Codex 自己启动 DeepSeek sidecar agents。

**不需要额外代理。带上你自己的 DeepSeek key，剩下交给 Codex。🚀**

`codex-deepseek-sidecar` 是一个 Codex skill，让你的 Codex 主 agent 可以启动更便宜的 DeepSeek 工人 agent，去处理边界清晰的 side task：长测试、日志分析、大范围代码探索、独立 review、实现尝试。

核心模式很简单：高级 GPT 继续做主大脑，负责规划、判断和综合；DeepSeek 工人负责重复、长上下文、token-heavy 的执行工作；所有事情仍然跑在 Codex harness 里。

## 🚀 一句话启动

把这个仓库链接和下面这句话交给 Codex：

```text
安装并使用 https://github.com/Zedong-Liu/codex-deepseek-sidecar。
我有 DeepSeek API key；如果本机还没有配置，请向我索取。
如果需要，请配置本地代理/profile。
然后为当前仓库启动一个 DeepSeek sidecar，用它处理适合分派的长任务或日志任务。
```

这才是面向人类的使用方式。你不需要学习 sidecar 参数、session ID、代理命令。这个 skill 是写给 Codex 读和执行的。在一台已有 Codex CLI、Python 3、DeepSeek key 的机器上，Codex 应该能在大约五分钟内完成安装、配置内置代理，并拉起第一个 sidecar。

之后正常用自然语言说就行：

```text
用一个 DeepSeek sidecar 跑慢测试，你同时检查代码。
```

```text
用一个 DeepSeek sidecar 分析这份 CI log，找出真正失败原因。
```

```text
你自己判断什么时候把这件事拆给 DeepSeek sidecars。适合长测试、日志分析、大范围探索时就自动分派、组织结果，最后给我一个综合方案。
```

## ✨ 为什么用户会想用

- 💸 **大幅降低 worker token 成本**：把重复读文件、看日志、跑测试、大范围探索从昂贵 GPT token 迁移到 DeepSeek worker token。很多工作流可以瞄准 **80-90% 更低 token 成本**。
- 🪄 **一句 prompt，agent 自己接管**：Codex 安装、配置、启动、分派、检查、恢复、汇总。
- 🔁 **支持 resume，不用重来**：sidecar session 会持久保存，长测试、被打断的调查、后续追问都可以恢复到正确工人。
- 🔌 **不需要外部代理**：仓库内置一个小型 Python 代理，把 Codex Responses API 桥接到 DeepSeek Chat Completions。
- 🔑 **Bring your own DeepSeek key**：不需要托管中间层。
- 🧠 **GPT 仍然做主脑**：昂贵模型负责规划、判断、综合；DeepSeek 负责边界清晰的工人任务。
- 🛠️ **继续使用 Codex harness**：sidecar 仍然具备 Codex 的文件访问、命令执行、session 和证据汇报能力。

## 💸 成本形状

下面价格按每 1M tokens 计算，基于 2026-06-02 查询到的 [OpenAI GPT-5.5 model page](https://developers.openai.com/api/docs/models/gpt-5.5/) 和 [DeepSeek pricing docs](https://api-docs.deepseek.com/quick_start/pricing)。DeepSeek Pro 输入价格使用 cache-miss 价格，属于偏保守比较。

| 模型 | 最适合的角色 | 输入 | 输出 |
| ---- | ---- | ----: | ----: |
| GPT-5.5 | 主大脑：规划、判断、综合结论 | $5.00 | $30.00 |
| DeepSeek V4 Pro | 强工人：review、调试、实现尝试 | $0.435 | $0.87 |
| DeepSeek V4 Flash | 快工人：日志、广泛探索、廉价并行 pass | $0.14 | $0.28 |

按 `1M input + 200K output` 粗略计算：

| 路由方式 | 近似成本 | token 成本降低 |
| ---- | ----: | ----: |
| 全部使用 GPT-5.5 | $11.00 | baseline |
| DeepSeek V4 Pro 工人 token | $0.61 | 约 94% |
| DeepSeek V4 Flash 工人 token | $0.20 | 约 98% |

真实使用里，GPT 仍然会花 token 做协调和审查。这正是设计目标：把昂贵 token 花在判断力上，把重复性的 worker-token 预算迁移到 DeepSeek。对很多 agent 工作流来说，**80-90% 的 token cost save** 是现实目标。

## 🧠 Codex 会在背后做什么

当你要求 Codex 使用这个 skill 时，它可以：

- 把仓库安装成 Codex skill。
- 如果本地没有可用 DeepSeek provider，就启动内置轻量代理。
- 一次性配置 Codex profile，避免每次任务重复设置。
- 为边界清晰的任务启动 DeepSeek sidecar。
- 跟踪 task ID 和 session，确保 follow-up 恢复到正确工人。
- 收集 sidecar 结论，并综合进主回答。

这些操作细节不应该塞给人类读者。Agent-facing instructions 放在 [SKILL.md](SKILL.md)。

## 🔌 内置代理

内置的 `deepseek-responses-proxy` 刻意保持很小：只用 Python 标准库，默认只监听本地，并针对 Codex 的大请求体设计。它会桥接 function tools，并忽略 Codex 默认附带但 DeepSeek Chat 不支持的 Responses built-in tools；如果请求明确要求某个不支持的 built-in tool，则返回明确错误。

如果你已经在用 VibeAround 或其他兼容 provider，Codex 也可以继续使用原来的方案。

## 📦 仓库结构

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
