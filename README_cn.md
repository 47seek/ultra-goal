# Ultra-Goal

[English](./README.md) | [中文](./README_cn.md)

> 为 Codex、Claude Code 等 Agent 设计的多智能体编排 Skill，用于在无人值守状态下持续推进长周期、大规模工程任务。

## 安装

直接让 Agent 安装：

> Install `prepare-ultra-goal`, `ultra-goal` and `ultra-goal-heavy` from <https://github.com/Azure99/ultra-goal>.

也可以通过 `skills` CLI 安装：

```bash
npx skills add Azure99/ultra-goal \
  --skill prepare-ultra-goal \
  --skill ultra-goal \
  --skill ultra-goal-heavy
```

## 三个 Skill

| Skill | 定位 | 说明 |
| --- | --- | --- |
| `prepare-ultra-goal` | 目标准备 | 通过只读探索与使用者对齐，把原始目标打磨成可独立验收的 `final-goal.md`，经确认后交接执行 |
| `ultra-goal` | 标准执行 | 主控 Agent 亲自规划与实现，委派子 Agent 评审计划与验收；成本较低，默认推荐 |
| `ultra-goal-heavy` | 重型执行 | 主控原则上不亲自动手，只负责协调与仲裁：双计划交叉评审、独立实现、多 Agent 交叉验收 |

如何选择：

* 目标存在歧义、较为复杂或风险较高：先用 `prepare-ultra-goal` 澄清并确认目标，再由它交接执行。
* 目标的预期结果、边界、约束和验收标准已清晰且风险较低：直接调用 `ultra-goal`。
* 仅当有明确证据表明多路独立规划与交叉评审能显著降低执行风险时，才选择 `ultra-goal-heavy`。

## 设计理念

Spec-Driven Development 已是 AI 辅助开发的事实标准，但仍有三个痛点：关键实现决策仍需使用者逐一裁定；超长程任务无法在动手前写尽 spec；中途发现计划不可行时，回退与重规划的成本很高。

Ultra-Goal 的应对方式：

* **决策前置，执行无人值守。** 需要人裁定的取舍集中在 `prepare-ultra-goal` 阶段一次性对齐；进入执行后全程不询问使用者，计划与验收中的争议由子 Agent 评审、主控仲裁，遇到无法解决的阻塞则记录后继续推进。
* **阶段循环，计划持续演化。** 工作被组织为「规划 → 实现 → 验收 → 提交」的阶段循环，每个通过验收的阶段产生一次 git commit；阶段结束后回顾 roadmap，动态新增、删除、合并、拆分或重排后续阶段，变更记录在 `roadmap-changelog.md` 中。
* **目标保持适度抽象。** 使用者只需给出清晰的目标与验收标准，无需预先设计完整的实施细节。

两个执行版本共享同一阶段循环，区别在于投入强度：

| | `ultra-goal` | `ultra-goal-heavy` |
| --- | --- | --- |
| 规划 | 主控制定计划，≥1 个子 Agent 评审 | 2 个子 Agent 独立出双计划，交叉评审后主控裁决 |
| 实现 | 主控亲自实现 | 委派 1 个子 Agent 实现 |
| 验收 | ≥1 个子 Agent 独立评审 | 2–3 个子 Agent 独立评审 + 交叉评审 |
| 必修问题 | blocker | blocker 与 high |

## 适用场景

启动一次执行流程会投入大量 token 用于规划、评审与仲裁，「添加一个按钮」「修复一个 typo」这类局部改动并不划算。Ultra-Goal 适合：

* 持续数小时乃至数天的大型任务：大型重构、跨技术栈迁移、从零搭建完整模块；
* 使用者无法持续盯着屏幕，希望任务无人值守推进；
* 并行推进多个项目，希望降低持续决策与监督成本。

## 模型选择

主控应尽可能选择能力强的模型（在 `Codex + gpt-5.5 xhigh` 组合下表现良好），如果模型能力不足，整个流程很容易失控。子 Agent 默认沿用主控模型，对成本敏感时可在目标中显式指定更小的模型，例如 `gpt-5.4 mini`。

## 使用建议

1. **目标应清晰，但不含实施细节。** 合适的目标如：「把这个 Express 项目迁移到 Fastify，保持现有 API 行为不变，所有现有的集成测试和 E2E 测试必须继续通过。」过于模糊的目标（「改一下项目」）应先经 `prepare-ultra-goal` 澄清；写成具体实施步骤（「第一步先改 `server.ts`……」）则会挤占子 Agent 的探索与规划空间，反而可能降低质量。
2. **尽量提供端到端的验收方法。** 仅依赖单元测试通常不够，子 Agent 完全可能让测试通过但功能实际不可用。Web 应用可让验收 Agent 用 `playwright-cli` 驱动真实浏览器验证关键路径；CLI 工具执行真实命令，校验 stdout、退出码和生成文件；服务端 API 启动真实服务进程，用 `curl` 验证关键 endpoint。
3. **必须显式调用，且需在 git 仓库内执行。** 三个 Skill 都不会被自动触发，这是有意为之：运行成本较高，应由使用者主动发起。执行流程会为每个通过验收的阶段创建 commit，因此需要在 git 仓库中运行。