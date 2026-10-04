# Ultra-Goal

[English](./README.md) | [中文](./README_cn.md)

> A task collaboration skill for agents such as Codex and Claude Code, designed to complete complex tasks lasting hours or even days: it researches first and draws up a plan, then proceeds autonomously in stages once the user confirms, pausing only when necessary.

## Installation

Ask an agent to install the skill directly:

> Install `ultra-goal` from <https://github.com/Azure99/ultra-goal>.

Alternatively, install it with the `skills` CLI:

```bash
npx skills add Azure99/ultra-goal --skill ultra-goal
```

## How It Works

The main agent is responsible for driving the entire process and can delegate work such as research and review to sub-agents. During the research period, it ascertains the current state, verifies the key approaches, and forms the overall plan. When the plan is ready, it pauses for the user's confirmation and at the same time asks whether stages that can be split off should run in parallel, and whether operations that need separate authorization, such as push and deployment, are authorized as well. After confirmation, it enters the implementation period and proceeds continuously, without asking permission to continue at each stage: every stage goes through the stage plan, implementation and verification, an independent review, and a local commit, in that order. When everything is done, it conducts the integration review and overall verification, then reports to the user.

The main agent pauses only at the following points:

* When raising a batch of questions during the research period;
* When the plan is ready;
* When items arise during implementation that affect subsequent work and require the user to act (pending questions, authorization needs, or blockers), in which case it first finishes the work that does not depend on them and then raises them all together;
* When everything is complete.

Each task gets its own directory, `ultragoal/YYYY-MM-DD-task-name/`: the top-level `README.md`, `decisions.md`, and `plan.md` are the entry point and the authoritative record, and detailed materials go in subdirectories. This directory is kept out of product commits and is used to resume work after an interruption and to look up past tasks.

A task that is small and well-defined, and that the user has asked to be completed directly, follows a simplified process: the main agent writes a brief plan and proceeds directly to implementation and verification, does one independent review after finishing, then commits and reports.

## Design Philosophy

Spec-Driven Development has become the de facto standard for AI-assisted development, but three pain points remain: key implementation decisions still have to be made by the user one by one; specifications for extremely long-horizon tasks cannot cover every detail before implementation begins; and when a plan proves infeasible midway through execution, the cost of rolling back and replanning is high.

Ultra-Goal addresses these problems as follows:

* **Research first; raise questions all together.** The main agent checks for itself any facts that can be checked, and makes and records routine professional judgments on its own; it asks the user only about information only the user has, trade-offs that would materially change the outcome, and authorization. Decision questions come with options and a recommended option, and the user can reply “go with your recommendations for the rest” without having to rule on them one by one.
* **Let the overall plan set the direction; leave the details to each stage.** The details of a complex task are hard to write out completely at the outset, so the overall plan sets down what will be delivered, the acceptance criteria, and the stage breakdown, while each stage plan is written when that stage begins, based on the actual state at the time. When reality differs from the plan during implementation without affecting the confirmed scope, the main agent adjusts the subsequent stages itself and records the reason; only when the scope is affected does it refer the matter to the user for a ruling.
* **Proceed autonomously after confirmation; pause only when necessary.** The user's confirmation to proceed with implementation authorizes the main agent to make changes and commit locally stage by stage; when it encounters a technical obstacle, it first investigates on its own and tries alternatives, and it pauses only for items that the user alone can act on.
* **Match the amount of process to the size of the task.** Time and tokens are both costs, and the quantity of documentation, reviews, and tests is not a measure of quality; how far to take the work is determined by the user's requirements, the project's conventions, and the actual usage scenarios, and the main agent does not, on its own initiative, make the work more complex or simpler than this calls for.
* **Keep the goal appropriately abstract.** Users need only provide a clear goal and acceptance criteria; they do not need to design every implementation detail in advance.

## Suitable Use Cases

Starting a workflow consumes tokens for research, review, and record-keeping, so local changes such as “add a button” or “fix a typo” are not worth the overhead. Ultra-Goal is suitable for:

* Complex tasks lasting hours or even days: major refactors, cross-stack migrations, or building a complete module from scratch, as well as non-code deliverables such as research reports and technical proposals;
* Situations where the user cannot monitor the task continuously: questions are answered all together, and once the plan is confirmed, work proceeds autonomously, pausing only when necessary;
* Running multiple projects in parallel while reducing ongoing decision-making and supervision overhead.

## Model Selection

The main agent should use the most capable model currently available, such as `Claude Code + Opus 5.5` or `Codex + GPT-6 Astra`. If the model is not capable enough, the entire workflow can easily go off track. By default, the platform decides which model sub-agents use; if cost is a concern, explicitly specify a smaller model in the request.

## Usage Recommendations

1. **The goal should be clear, but should not include implementation details.** A suitable goal would be: “Migrate this Express project to Fastify while keeping the existing API behavior unchanged. All existing integration tests and E2E tests must continue to pass.” If the goal is overly vague (“Make some changes to the project.”), the main agent first asks one or two direction-setting questions separately during the research period, then raises the remaining questions all together; writing it as concrete implementation steps (“First, modify `server.ts`...”) leaves less room to explore and plan, and may reduce the quality of the result.
2. **Provide end-to-end acceptance checks whenever possible.** Unit tests alone are usually not enough: an agent could make the tests pass while the actual functionality remains unusable. For web applications, have an agent use `playwright-cli` to drive a real browser through critical paths; for CLI tools, run real commands and verify stdout, exit codes, and generated files; for server-side APIs, start a real service process and use `curl` to verify key endpoints.
3. **Explicit invocation is required, and code-changing tasks must run inside a Git repository.** This skill is not triggered automatically, by design: its runtime cost is high, so the user must initiate it explicitly. During the implementation period, the main agent commits locally stage by stage, so a Git repository is needed; if it is not in a Git repository, it asks in the first batch of questions whether to initialize one, without interrupting the research. Non-code tasks use checkpoints instead of commits.

## Upgrading from v1.x

* **`prepare-ultra-goal` and `ultra-goal-heavy` have been removed.** Goal clarification has been merged into the research period of `ultra-goal`. Multiple independent planning tracks and cross-review are no longer offered separately; if more review is needed, say so in the request.
* **Questions are now raised all together, with pauses as needed.** In v1.x, once execution began, the workflow never asked the user any questions; the new version raises questions all together during the research period, waits for confirmation once the plan is ready, and also pauses during implementation when something comes up that requires the user to act.
* **The task directory is now `ultragoal/YYYY-MM-DD-task-name/`.** Its structure differs from that of `ultragoal/<goal-name>/` in v1.x.

Uninstall the old skills (add `-g` if they were installed globally):

```bash
npx skills remove prepare-ultra-goal ultra-goal-heavy
```
