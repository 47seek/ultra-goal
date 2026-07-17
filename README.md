# Ultra-Goal

[English](./README.md) | [中文](./README_cn.md)

> A multi-agent orchestration skill for agents such as Codex and Claude Code, designed to keep long-running, large-scale engineering tasks moving forward unattended.

## Installation

Ask an agent to install the three skills directly:

> Install `prepare-ultra-goal`, `ultra-goal` and `ultra-goal-heavy` from <https://github.com/Azure99/ultra-goal>.

Alternatively, install them with the `skills` CLI:

```bash
npx skills add Azure99/ultra-goal \
  --skill prepare-ultra-goal \
  --skill ultra-goal \
  --skill ultra-goal-heavy
```

## The Three Skills

| Skill | Role | Description |
| --- | --- | --- |
| `prepare-ultra-goal` | Goal preparation | Uses read-only exploration and alignment with the user to refine a raw goal into an independently verifiable `final-goal.md`, then hands it off for execution after confirmation |
| `ultra-goal` | Standard execution | The Lead Agent plans and implements directly, delegating plan and acceptance reviews to sub-agents. This option costs less and is recommended by default |
| `ultra-goal-heavy` | Heavy execution | The Orchestrator generally does not implement directly and focuses on coordination and arbitration: two plans are developed independently and cross-reviewed, implementation is delegated to an independent sub-agent, and multiple sub-agents conduct independent acceptance reviews followed by cross-review |

How to choose:

* If the goal is ambiguous, complex, or high-risk, use `prepare-ultra-goal` first to clarify and confirm it, then have it hand the goal off for execution.
* If the expected outcomes, boundaries, constraints, and acceptance criteria are clear and the risk is low, invoke `ultra-goal` directly.
* Choose `ultra-goal-heavy` only when there is clear evidence that multiple independent planning tracks and cross-review can materially reduce execution risk.

## Design Philosophy

Spec-Driven Development has become the de facto standard for AI-assisted development, but three pain points remain: key implementation decisions still have to be made by the user one by one; specifications for extremely long-horizon tasks cannot cover every detail before implementation begins; and when a plan proves infeasible midway through execution, the cost of rolling back and replanning is high.

Ultra-Goal addresses these problems as follows:

* **Front-load decisions, then execute unattended.** Trade-offs requiring human judgment are resolved together during the `prepare-ultra-goal` phase. Once execution begins, the workflow does not ask the user questions. Disputes arising during planning and acceptance are reviewed by sub-agents and adjudicated by the orchestrator; unresolvable blockers are recorded and work continues.
* **Use a stage loop; let the plan evolve.** Work is organized into stages, each following the same cycle: planning → implementation → acceptance → commit. Every stage that passes acceptance produces a Git commit; after each stage, the roadmap is revisited and subsequent stages may be dynamically added, removed, merged, split, or reordered. Changes are recorded in `roadmap-changelog.md`.
* **Keep the goal appropriately abstract.** Users need only provide a clear goal and acceptance criteria; they do not need to design every implementation detail in advance.

Both execution variants share the same stage loop but differ in level of effort:

| | `ultra-goal` | `ultra-goal-heavy` |
| --- | --- | --- |
| Planning | The Lead Agent creates the plan; ≥1 sub-agent reviews it | Two sub-agents each produce an independent plan; after cross-review, the Orchestrator adjudicates |
| Implementation | The Lead Agent implements directly | Delegated to 1 sub-agent |
| Acceptance | ≥1 sub-agent independently reviews | 2–3 sub-agents conduct independent reviews, followed by cross-review |
| Must-fix severities | `blocker` | `blocker` and `high` |

## Suitable Use Cases

Starting an execution workflow consumes a large number of tokens for planning, review, and arbitration, so local changes such as “add a button” or “fix a typo” are not worth the overhead. Ultra-Goal is suitable for:

* Large tasks lasting hours or even days: major refactors, cross-stack migrations, or building a complete module from scratch;
* Situations where the user cannot monitor the task continuously and wants it to progress unattended;
* Running multiple projects in parallel while reducing ongoing decision-making and supervision overhead.

## Model Selection

The lead agent should use the most capable model available (the `Codex + gpt-5.5 xhigh` combination performs well). If the model is not capable enough, the entire workflow can easily go off track. Sub-agents use the lead agent's model by default; if cost is a concern, explicitly specify a smaller model in the goal, such as `gpt-5.4 mini`.

## Usage Recommendations

1. **The goal should be clear, but should not include implementation details.** A suitable goal would be: “Migrate this Express project to Fastify while keeping the existing API behavior unchanged. All existing integration tests and E2E tests must continue to pass.” An overly vague goal (“Make some changes to the project.”) should first be clarified with `prepare-ultra-goal`; writing it as concrete implementation steps (“First, modify `server.ts`...”) leaves sub-agents less room to explore and plan, and may reduce the quality of the result.
2. **Provide end-to-end acceptance checks whenever possible.** Unit tests alone are usually not enough: a sub-agent could make the tests pass while the actual functionality remains unusable. For web applications, have an acceptance agent use `playwright-cli` to drive a real browser through critical paths; for CLI tools, run real commands and verify stdout, exit codes, and generated files; for server-side APIs, start a real service process and use `curl` to verify key endpoints.
3. **Explicit invocation and a Git repository are required.** The three skills are not triggered automatically, by design: their runtime cost is high, so the user must initiate them explicitly. The execution workflows create one commit for each stage that passes acceptance, so they must run inside a Git repository.
