[中文](./README_zh.md)

# looppower

A Claude Code plugin built on the strict separation of planning and execution: the Expert Panel plans, and The Loop executes. Each side plays its own role — the **schizophrenic** Expert Panel produces High-Level plans, and the **dynamic-loop** skill executes them phase by phase, until the Expert Panel determines the goal achieved.

## Quick Start

Install in Claude Code:

```
/plugin marketplace add zhaoyshine/looppower
/plugin install looppower
```

After installation, confirm that `looppower` appears via `/plugin`.

Requires Claude Code v2.1.196 or later (see [Environment Requirements](#environment-requirements)).

## Usage Examples

**Scenario: pre-release project check.** The user enters in Claude Code:

```
/looppower:dynamic-loop check this project before I publish it to GitHub
```

The Loop then begins: the Expert Panel plans one phase, the skill dispatches subagents in parallel to run each check task and backfill the results, and the Expert Panel reviews the outcomes and determines whether to continue or stop — until the requirement has been met. You can interrupt at any time to add information or adjust direction.

## How It Works

Most agentic workflows let the same agent both decide and execute, which means plans quietly go stale. looppower splits the responsibilities across two roles, each with its own scope:

- **Planning belongs to the Expert Panel.** The **schizophrenic** agent operates as an Expert Panel: it plans one phase at a time and produces a High-Level Execution Plan document, never touching execution. When a phase finishes, it reviews the feedback and plans the next phase — until it determines the goal achieved.
- **Execution belongs to The Loop.** The **dynamic-loop** skill executes strictly by the Expert Panel's plan. Every task has progress tracking; once a phase completes, it returns to the Expert Panel to request the next round. No improvisation, no deviation from the plan.

## Basic Workflow

1. When a task requires multi-step planning, or an execution plan already exists and is ready to run, the **dynamic-loop** skill takes over.
2. The skill requests a High-Level Execution Plan from the Expert Panel (**schizophrenic**).
3. The skill executes the plan task by task with `TaskCreate`/`TaskUpdate`, tracking progress along the way.
4. When a phase completes, the Expert Panel reviews the results and determines whether another phase is needed.
5. The Loop repeats until the Expert Panel declares the goal achieved.

Simple one-off tasks don't need this skill.

## What's Included

- **schizophrenic** — the Expert Panel agent that produces a High-Level Execution Plan for one phase at a time.
- **dynamic-loop** — the execution skill that runs the plan in a loop, tracks progress, and requests the next phase until the Expert Panel declares the goal achieved.

The skill is deliberately named **dynamic-loop** rather than looppower: the plugin is the whole, and the skill is one component inside it. Components are named by their responsibility, not after the plugin.

## Design Philosophy

**Model tiering.** The Expert Panel uses the strongest AI model (opus) by default and is responsible for exactly one thing: High-Level planning and major decisions. It doesn't read code or run tasks. The token-hungry work — researching code, running linters, mechanical execution — is split into small tasks delegated to cheaper models (haiku/sonnet). Model selection follows "opus for major decisions, haiku for simple things, sonnet as a backup for haiku," and opus usage is reduced by keeping task complexity small. Claude Code's plan mode has the main model plan while exploring, and exploration itself is expensive; this workflow keeps the most expensive model entirely out of execution and uses it only for decisions — for most tasks, the same results come at a lower overall cost.

**Automatic tool extensibility.** The skills and agents of this plugin live under `skills/` and `agents/` at the plugin root, where Claude Code discovers and triggers them automatically. New capabilities and tools shipped in Claude upgrades require no changes to this workflow; skills and agents you download and install yourself are discovered the same way and brought into execution.

**Adaptive process.** The process adapts dynamically as AI improves and as your preferences change. Planning is produced phase by phase: only one phase at a time, as much as current information can support, with the next phase planned only after real execution results are backfilled — the plan is always grounded in facts. The Loop has no preset number of rounds; it runs until the Expert Panel determines "the requirement has been met." Model selection, preference and forbidden rules, and the plan document directory are all configured to your environment and preferences.

## Environment Requirements

- Claude Code v2.1.196 or later. **schizophrenic**'s plan document path template uses the `${CLAUDE_PROJECT_DIR}` placeholder, which is only expanded starting from v2.1.196. On older versions, the plan path cannot be resolved.

## License

[MIT](LICENSE)
