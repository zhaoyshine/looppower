# User Preferences

## Development Workflow

- Good development workflow: investigate, user confirms the change plan, develop per the plan, run tests on changed files, run lint on changed files, create a PR per project requirements
- For projects managed with git, use git worktree for parallel development
- When writing code/tests, first read how existing files are written (factory, helper, naming conventions), mirror their style, don't invent new conventions
- Set manual confirmation checkpoints at key milestones (e.g., plan confirmation, diff confirmation); checkpoints cannot be skipped or merged, and do not proceed to the next step without explicit approval
- Write only important comments; keep them simple and clear, don't restate the "what"

## Commits & PRs

- Commit format uses conventional commits + Co-Authored-By trailer
- Project conventions (branch names / commit format / PR template) are read from the repo at runtime, never hardcoded
- PR body must be a strict verbatim copy of the repo's own PR template, without adding/removing/reordering sections, then fill in within the skeleton; every section must have substantive content or an explicit N/A
- Perform reversible operations first (prepare git worktree, local normalization), then irreversible ones (push, open PR, update Jira, mark ready)

## Git & Worktrees

- Ask the user whether to use Worktree
- Before reading code, first update the repo to the latest main branch and ensure the working area is clean
- When the working tree is not empty, do not stash/reset/clean on your own; leave it as is and return it to the user to handle
- Only use git pull --ff-only; non-fast-forward merges are forbidden

## Testing

- Prefer adding test code to existing test files; creating new ones is the last resort; reuse existing factory/helper/mock infrastructure
- One test case asserts only one dimension; group by dimension
- Write/confirm a failing reproduction (a test or reproduction case) before fixing a bug, so you know when it is actually fixed
- Test quality standard: Fast, Independent, Repeatable, Self-validating, Timely (F.I.R.S.T.)

## Investigation

- Every conclusion must cite its source (URL / file path / Jira key / PR number / log snippet / line number); no guessing
- Exhaust the call chain: ≥2 search commands per key symbol, ≥2 repos when cross-repo involvement is suspected, and each "exhausted" claim must be paired with ≥1 counter-evidence
- Follow up deeply: open and read every URL / ticket key / incident ID / error code / file path / PR number encountered
- When the same fact appears in multiple sources, take the lowest confidence tier (conservative principle)
- For key investigation targets, confirm coverage with at least two different methods / keyword variants; don't wrap up after a single command
- When encountering unfamiliar names/services/libraries/flags, proactively research their background instead of bypassing them
- Don't guess about uncertain things; if the answer cannot be obtained, ask the user, and leave a trace to avoid repeated deliberation
- Maintain a list of "common laziness excuses" (e.g., "it should all be there", "no time, hand in a simplified version"); when such thoughts arise, stop and re-check against the principles
- Before applying automated judgment rules, first consult an authoritative data source to verify the real status; don't conclude based solely on raw extracted fields

## Planning

- Each step in the plan is a single action (2-5 minute granularity); it should not be overly complex
- Merge mechanical changes that repeat the same pattern into one task; fold setup/config/scaffolding/documentation steps into the tasks that need them
- When consolidating/integrating subagent outputs, re-synthesize and distill; do not paste verbatim, and do not re-judge or rewrite the conclusions already given by the subordinates
- Supplementary notes/annotations must not be redundant summaries of existing content; write only additional information the main text does not cover
- Avoid common AI writing tells: high-frequency filler words ("furthermore", "crucially", "delve into", etc.), promotional exaggeration, vague attribution, formulaic "challenges and outlook" paragraphs, negated parallelism, avoiding copular verbs, mismatched-scale "from X to Y", overuse of dashes/bold/emojis, inline-heading rephrasing of the same point, collaborative spoken-language traces, knowledge-cutoff disclaimers, sycophantic tone, filler phrases, over-qualification, hollow positive conclusions

## Communication

- When intent/pattern cannot be determined, ask the user one precise clarifying question instead of listing many
- Socratic questioning: ask only when there is a real doubt, one question at a time, wait for the reply before asking the next, and skip confirmation entirely when there is no doubt

## Documentation

- Documentation explains the "why" (intent, rationale), and doesn't restate the "what" already shown in the steps
- Treat documentation and commit messages with the same care as writing prose; clarity and precision matter as much as content
- Build documentation as part of the work, not as an afterthought
