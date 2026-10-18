# Ultra-Goal

[English](./README.md) | [中文](./README_cn.md)

> A task collaboration skill for agents such as Codex and Claude Code, designed to complete complex tasks lasting hours or even days. It researches first and draws up a plan, then, once the user confirms, proceeds autonomously in stages, pausing only when necessary.

## Installation

Install with the `skills` CLI:

```bash
npx skills add Azure99/ultra-goal --skill ultra-goal
```

Alternatively, ask an agent to install it directly:

> Install `ultra-goal` from <https://github.com/Azure99/ultra-goal>.

## Usage

This skill is never triggered automatically; the user must invoke it explicitly in the request, for example:

> Use ultra-goal to accomplish the following goal: migrate this project from Express to Fastify while keeping the existing API behavior unchanged. All existing integration tests and E2E tests must continue to pass.

Tasks that modify code must run inside a Git repository, because the implementation period commits locally stage by stage. If the working directory is not inside a Git repository, the main agent asks in its first batch of questions whether to initialize one. Non-code tasks use checkpoints in place of commits.

## Workflow

The main agent drives the entire process and may delegate research, review, and similar work to sub-agents.

1. **Research**: The main agent establishes the current state and verifies the key approaches. Questions that require the user's decision are raised together, each with candidate options and a recommended option; the user may reply "go with your recommendations for the rest."
2. **Plan confirmation**: The overall plan defines the deliverables, acceptance criteria, and stage breakdown, and is submitted to the user for confirmation. Operations that require separate authorization, such as push and deployment, may be authorized at the same time.
3. **Staged implementation**: Each stage proceeds through a stage plan, implementation and verification, an independent review, and a local commit, without requesting permission at each stage.
4. **Integration and delivery**: After an integration review and overall verification, the main agent reports to the user.

During implementation, the main agent pauses only when an item arises that the user must act on (a pending question, an authorization need, or a blocker), and it completes all work that does not depend on such items first. Tasks that are small and well-defined, and that the user has asked to be completed directly, follow a simplified process.

The records for each task are kept in `ultragoal/YYYY-MM-DD-task-name/`. They are used to resume work after an interruption and to look up past tasks, and are excluded from product commits.

## Suitable Use Cases

Ultra-Goal is intended for complex tasks lasting several hours or longer, such as major refactors, cross-stack migrations, and building complete modules from scratch, as well as non-code deliverables such as research reports and technical proposals. It is particularly suited to situations in which the user cannot monitor the task continuously. Each run consumes tokens for research, review, and record-keeping, so it is not recommended for local changes such as adding a button or fixing a typo.

The main agent should use the most capable model currently available; an insufficiently capable model may cause the workflow to go off track. By default, the platform determines which model sub-agents use; where cost is a concern, a smaller model may be specified in the request.

## Recommendations for Writing Goals

- **Define the goal and acceptance criteria without prescribing implementation steps.** Expressing the goal as concrete steps (for example, "First, modify `server.ts`") restricts exploration and planning and may reduce the quality of the result. If the goal is overly vague, the main agent first asks one or two direction-setting questions during the research period.
- **Provide end-to-end acceptance checks wherever possible.** Unit tests alone are usually insufficient, as passing tests do not guarantee that the functionality works. For web applications, use `playwright-cli` to drive a real browser through critical paths; for CLI tools, run real commands and verify stdout, exit codes, and generated files; for server-side APIs, start a real service process and verify key endpoints with `curl`.

## Design Philosophy

Spec-Driven Development has become the de facto standard for AI-assisted development, but three pain points remain. Ultra-Goal addresses them as follows:

| Pain point | Approach |
| --- | --- |
| Key implementation decisions must still be made by the user one by one. | The main agent verifies checkable facts itself and makes and records routine professional judgments on its own; it consults the user only on information only the user has, trade-offs that materially affect the outcome, and authorization. |
| Specifications for long-horizon tasks cannot cover every detail before implementation begins. | The overall plan sets the direction; the details of each stage are determined when that stage begins, based on the actual state at that time. |
| When a plan proves infeasible midway, rolling back and replanning is costly. | Deviations that do not affect the confirmed scope are adjusted by the main agent, with the reasons recorded; only deviations that affect the scope are referred to the user. |

## Upgrading from v1.x

<details>
<summary>Upgrade notes</summary>

- **`prepare-ultra-goal` and `ultra-goal-heavy` have been removed.** Goal clarification has been merged into the research period. Multiple independent planning tracks and cross-review are no longer offered separately; if additional review is needed, specify it in the request.
- **Questions are now raised together, with pauses as needed.** In v1.x, the workflow did not ask the user any questions once execution began. The current version raises questions together during the research period, waits for confirmation once the plan is ready, and pauses during implementation when an item requires the user's action.
- **The task directory is now `ultragoal/YYYY-MM-DD-task-name/`.** Its structure differs from that of `ultragoal/<goal-name>/` in v1.x.

Uninstall the old skills (add `-g` if they were installed globally):

```bash
npx skills remove prepare-ultra-goal ultra-goal-heavy
```

</details>
