# User Forbidden Items

## Development Workflow

- Comments that merely restate the implementation / provide no help to code readers (repeating the "what" the code already expresses)
- Verbose, wordy comments / longer than 3 lines (if a comment can be omitted, omit it; if written, write only the key points)
- Casually refactoring unrelated code or sneaking in out-of-scope features
- Using English throughout the entire development workflow (code, comments, variable names, commit messages, Jira tasks, PR titles and bodies)

## Commits & PRs

- Attempting to bypass the pre-commit hook
- Using git add -A / adding everything at once with git add; list specific files instead

## Git & Worktrees

- Committing/checking out/modifying files/switching branches/pulling in the original repo working directory (all changes must stay isolated inside the git worktree)
- Using git worktree remove / rm -rf / deleting the worktree directory (leave it as is when done, hand it off to the downstream)
- Using git push --force / --force-with-lease (except when only adding commits without rewriting history)
- Using git reset --hard, force-pushing
- Using git commands outside a repo directory
- Hardcoding the main branch name as main/master
- Stashing/resetting/cleaning/checking out on your own when the working tree is dirty
- Using non-fast-forward merges (git pull must use --ff-only)

## Testing

- Writing tests that can never be triggered (if there is no reliable injection mechanism for an error path, don't write the test; don't add documentation-only tests for defensive code)
- Running the full test suite (only run tests/Lint/Check related to the changes)

## Investigation

- Stating conclusions without providing supporting evidence
- Knowing the answer is in the code but not looking it up
- Using vague platitudes for conclusions/insights (e.g., "this is important", "worth learning", "AI is the trend")
- Fabricating or padding when there are no findings/results (should honestly state "no issues found" / "only N items")
