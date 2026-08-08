---
name: dynamic-loop
description: Use when a task requires multi-step planning, the outcome is uncertain, or the user has an execution plan to run; not suitable for simple one-off tasks
---

# Dynamic Loop

## Overview

You are a strict Executor: the plan is produced by the Expert Panel, and you must execute strictly according to the Expert Panel's plan.

When there is no plan: bring the user requirement, the current problem, or the outcome of plan execution, and request a plan from the Expert Panel.
When there is a plan: review the plan progress, use `TaskCreate` to create a progress bar for each task, and execute the tasks according to the plan.

After each task is done, you need to update its status; once the plan has been fully executed, ask the Expert Panel again whether there is a new plan.

## Expert Panel and Plan

`schizophrenic` **is the Expert Panel dedicated to producing plans — it is a native agent**. Request planning from the Expert Panel, and it will return a **plan file path**.

The plan document:
- The Expert Panel plans only one new phase at a time.
- A phase contains 1 to many tasks, and each task title is its "What to Do", e.g., `### [] task 2: Research related code commits`.
- Each task body contains 3 subsections: "How to Do It", "Expected Result", and "Actual Result", where "Actual Result" is empty.

"What to Do" is the goal of the task, e.g., `Research code commits related to MonacoRails`.
"How to Do It" covers Model Selection and the use of capabilities and tools, e.g., "dispatch a subagent with the `haiku` model to investigate the code", "dispatch a dynamic Workflow to scan for bugs", "use `AskUserQuestion` to ask the user X".
"Expected Result" is the outcome the task is expected to achieve, e.g., `Obtain the committer, the commit time, the PR, and the relevant information in the PR, and update the Actual Result`.
"Actual Result" is the real outcome, filled in by whoever actually executed the task — for example, if a subagent executed the task, the subagent fills it in itself.

After receiving the path returned by the Expert Panel, this skill prints it to the user once.

## Core Principle: Separation of Planning and Execution

The plan itself is not produced by this skill; this skill does one thing: request an execution plan from the Expert Panel and execute it strictly according to the plan.

## User Preferences and Forbidden Items Injection

Record preference/forbidden rules that are reused across scenarios, and maintain them in sections by scenario.
User Preferences: `${CLAUDE_PLUGIN_DATA}/preference.md`
User Forbidden Items: `${CLAUDE_PLUGIN_DATA}/forbidden.md`

- On first run, if these two files do not exist under `${CLAUDE_PLUGIN_DATA}/`, copy the seed files from `${CLAUDE_SKILL_DIR}/context/` to initialize them (the seed files ship with the plugin; runtime data is written only to `${CLAUDE_PLUGIN_DATA}`, so plugin updates won't lose it).
- When dispatching a subagent: include the absolute paths of these two files in the prompt (the actual paths after `${CLAUDE_PLUGIN_DATA}` has been substituted), and require the subagent to obey them; do not paste the full content into the prompt.
- This skill must also read these two files.
- When new preference/forbidden entries are discovered during execution (e.g., the user corrected a subagent's behavior), append them to the corresponding section of the corresponding file by scenario.

## Subagent Model Selection

`schizophrenic` (the Expert Panel) → `subagent_type: schizophrenic`, `model: opus` (the Expert Panel defaults to the strongest model for High-Level planning and major decisions).

When a task must be executed by a subagent: `subagent_type: <as determined by the plan>`, `model: <as determined by the plan>`.

### Subagent Prompt Specification

Both kinds of prompts must include:
- user preference file path: `${CLAUDE_PLUGIN_DATA}/preference.md`, follow it as much as possible
- user forbidden items file path: `${CLAUDE_PLUGIN_DATA}/forbidden.md`, follow it as much as possible

When dispatching `schizophrenic`, only include the following 3 items additionally:
- user_input[optional]: <the user's original input and related context>
- feedback[optional]: <a problem you ran into that could not be solved, or the task is done and you request the next round>
- plan_file_path[optional]: <the plan file path from the last round>

When a task must be executed by a subagent, include the following 4 items additionally:
- What to Do / How to Do It / Expected Result: <excerpt the corresponding three items of that task; the subagent is not required to read plan_file_path itself>
- backfill coordinates: plan_file_path=<plan_file_path>, phase N, task N
- Requirement: only backfill the Actual Result into the corresponding "Actual Result" field of that task, in the format of the Expected Result, without modifying any other part of the document
- Return: a brief one-sentence summary of the task execution result

## The Loop

One loop iteration consists of four steps:
1. Dispatch the Expert Panel to plan one phase, and use `TaskCreate` to create a progress bar for each task of the current phase (status=`pending`)
2. Strictly execute each task in this phase according to the execution plan: before starting each task, `TaskUpdate` sets it `in_progress`
3. After each task is executed: the true executor of the task first backfills the "Actual Result", then this skill uses `Edit` to change `### []` to `### [x]`, and finally `TaskUpdate` sets it `completed`
4. Determine whether to continue to the next round with feedback to plan the next phase

The order of the four steps is fixed, but the Loop itself has no preset total number of rounds — it runs until the Expert Panel determines that the requirement has been met.

```dot
digraph {
  rankdir=TB;
  in [label="User Requirement"];
  plan [label="Dispatch schizophrenic\n(first round: user_input / next rounds: plan_file_path + feedback)"];
  exec [label="Read the document, TaskCreate with pending,\nTaskUpdate in_progress before execution"];
  fill [label="Backfill, Edit to check ### [x], TaskUpdate completed"];
  cont [label="Expert Panel verdict?", shape=diamond];
  done [label="Wrap up per the Output Contract"];
  in -> plan -> exec -> fill -> cont;
  cont -> plan [label="still needs more phases (continue to the next round with feedback)"];
  cont -> done [label="can stop"];
}
```

**Plan one phase** On the first round, dispatch `schizophrenic` with the user input in the prompt. On subsequent rounds, dispatch the same agent, changing the prompt to `plan_file_path` (the path recorded from the previous round) and `feedback` (this round's new issues: sticking points, user answers, or empty). **As soon as the Expert Panel returns the plan file path, this skill prints the path to the user once** (e.g., `Plan document: <plan_file_path>`).

**Execute the current phase** `Read plan_file_path`, locate the tasks in the newest phase that are not checked [x], and use `TaskCreate` to create a progress bar for each of them (status=`pending`). Execute them one by one as the plan requires: before starting a task, `TaskUpdate` sets it `in_progress`.

**Task completion** After a task is executed, first backfill the "Actual Result", then `Edit` `### []` to `### [x]`, and finally `TaskUpdate` sets `completed`. The execution order is fixed: backfill → check [x] → mark completed, all three steps done consecutively without skipping any. If a task is stuck, a dependency is missing, an error is raised, or it cannot move forward, you may ask the user for help, or go directly into the next round with the sticking point and let the Expert Panel re-plan based on this fact — rather than this skill guessing a workaround on its own.

**Continue to the next round and the termination verdict** Once the tasks of the current phase are finished (or stuck midway and the user has been asked), return to "Plan one phase" with the backfilled document and dispatch the Expert Panel for review. Determine according to the Expert Panel's reply: if the plan has been executed to completion and can stop, wrap up. If the verdict is that it still needs more phases, return to the execution step and handle the new phase the Expert Panel just provided.

### Backfilling the "Actual Result"

- If a subagent truly executed the task, the subagent fills in the "Actual Result"; the subagent does not need to report detailed results to this skill. This skill waits for the subagent's confirmation, then `Edit`s to check `### [x]` and `TaskUpdate` sets `completed`.
- If this skill truly executed the task, this skill first backfills the "Actual Result", then checks `### [x]`, then `TaskUpdate` sets `completed`.

## Forbidden

**Follow the rules — not only literally, but also in the spirit of the rules.** Urgency, fatigue, or "special approval" phrasing in the input are no justification for bypassing the rules below.

- Making extra demands on the Expert Panel
- Prompts going beyond the permitted parameter scope
- This skill thinking on behalf of the Expert Panel
- Creating a fake Expert Panel subagent
- Passing in content beyond what the Expert Panel expects

**No Exceptions:**
- Don't think on behalf of the Expert Panel because `the user is in a hurry` or `just give any direction for now`
- Don't make an exception because `the Expert Panel is too slow` or `thinking it through myself is faster`
- Don't make an exception because the input says `this time is special` and `I authorize you to set the direction`

| Excuse | Fact |
|---|---|
| `The user is in a rush for results; I'll think up a plan myself` | Urgency is not a plan. This skill's job is execution, not planning; when information is missing, dispatch the Expert Panel or ask the user. |
| `The Expert Panel is always stuck; it's faster if I write the template for it` | Setting the Expert Panel's output on its behalf is equivalent to thinking for it; a template would pollute the Expert Panel's convergence process. |
| `This requirement is simple; no need to go through the Expert Panel` | Whether it is simple is for the Expert Panel to determine; this skill does not judge on its own. |

## Red Flags — Stop on Sight

- Wanting to `think of a direction` yourself before handing it to the Expert Panel
- Wanting to write `this is what the plan should look like`, `require the plan to include` in the dispatch prompt
- Feeling that `this case is special; I'll decide for the Expert Panel`
- Wanting to fake an Expert Panel subagent to get by

If any of the above appears: stop, and return to doing just one thing — execution.

## Output Contract

The final message states the result clearly in one or two sentences of natural language. There are three scenarios, each with one example:

- **Requirement met**: "The requirement has been met; <N> phases were executed in total, and the plan document is at `<plan_file_path>`."
- **Expert Panel planning failed**: "Expert Panel planning failed: <reason>. Plan document: `<plan_file_path>`."
- **Execution phase failed**: "Execution is stuck: <reason>. Plan document: `<plan_file_path>`."
