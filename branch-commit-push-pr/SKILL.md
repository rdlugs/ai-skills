---
description: Create a task-specific Git branch, commit the intended
  changes, push the branch, and open a pull request or merge request.
  Use when the user asks to publish work for review.
name: branch-commit-push-pr
---

# Branch, Commit, Push, and Open a PR

## Purpose

Safely publish the current task's changes for review by creating a new
branch, committing only the intended files, pushing the branch to its
remote, and opening a pull request (PR) or merge request (MR).

## Rules

-   Inspect the repository and its Git state before changing anything.
-   Never commit unrelated user changes, secrets, credentials, generated
    artifacts, or files that are excluded by repository instructions.
-   Do not discard, reset, stash, amend, or rewrite existing work unless
    the user explicitly asks.
-   Do not commit directly to `main`, `master`, `develop`, `production`,
    or another protected/default integration branch.
-   Do not use `git push --force` or force-push variants.
-   Follow repository instructions in `AGENTS.md`, `CLAUDE.md`,
    `CONTRIBUTING.md`, and relevant nested instruction files.
-   Do not invent test results. Run relevant checks when practical and
    report what was and was not run.
-   If authentication, permissions, or required metadata prevent
    completion, explain the blocker and provide the next step. Never
    claim a PR/MR was opened unless the hosting CLI or service confirms
    it.
-   Use the project's existing branch naming and commit-message
    conventions when available. Otherwise, use the defaults below.
-   If the requested code changes have not been made yet, do not create
    an empty or misleading commit. Complete the requested work first, or
    ask what should be committed.

## Workflow

### 1. Inspect repository state

Run:

``` bash
git rev-parse --show-toplevel
git status --short --branch
git branch --show-current
git remote -v
```

Then inspect applicable project instructions and the diff:

``` bash
git diff
git diff --cached
```

If the working tree contains changes, determine which belong to the
current task. Preserve all unrelated or pre-existing changes. If
ownership of a change is unclear and staging it could include someone
else's work, ask the user before proceeding.

### 2. Identify the target and hosting platform

-   Identify the current branch and the intended base branch from
    repository configuration, recent history, project instructions, or
    the user's request.
-   Prefer the repository's documented default branch. Do not assume the
    base is `main` if evidence says otherwise.
-   Detect the hosting platform from the configured remote URL.
-   For GitHub, use `gh` if installed and authenticated.
-   For GitLab, use `glab` if installed and authenticated.
-   If the repository uses another host, use an available supported CLI
    or open the relevant compare/new-PR page only if a verified URL can
    be derived.
-   Do not print tokens or credentials.

Check CLI availability and authentication as appropriate:

``` bash
gh --version
gh auth status
glab --version
glab auth status
```

Only run the commands relevant to the detected host.

### 3. Create a new branch

Choose a concise branch name that describes the task. Follow repository
conventions where present. Otherwise use one of these prefixes:

-   `feat/<short-description>` for a feature
-   `fix/<short-description>` for a bug fix
-   `refactor/<short-description>` for refactoring
-   `docs/<short-description>` for documentation
-   `chore/<short-description>` for maintenance

Use lowercase words separated by hyphens. Avoid spaces and sensitive
information.

Before creating the branch, verify that the chosen name does not already
exist locally or remotely. If it exists, choose a distinct name or ask
the user rather than switching to or overwriting an existing branch
unexpectedly.

Create the branch from the appropriate base. If the working tree
contains task changes, create the branch without losing those changes.
Do not switch branches if doing so would risk losing or misplacing user
work.

Example:

``` bash
git switch -c fix/short-description
```

If the intended base is not the current branch, first make a safe plan
to base the new branch on the correct commit. Do not blindly pull,
rebase, or reset a dirty working tree.

### 4. Review and stage the intended changes

Inspect the full diff, including untracked files:

``` bash
git status --short
git diff
git diff --cached
```

Run relevant tests, linters, formatters, or build checks when practical.
Review the result for accidental files, secrets, debug code, and
unrelated edits.

Stage only files that belong to the task. Prefer explicit paths:

``` bash
git add path/to/file1 path/to/file2
```

Do not default to `git add .` or `git add -A` when unrelated or
pre-existing changes may be present.

Inspect the staged diff before committing:

``` bash
git diff --cached
git status --short
```

If the staged changes include unrelated work, unstage only those
unrelated paths without deleting their working-tree changes.

### 5. Commit

Write a short, meaningful commit message that follows repository
conventions. If no convention exists, use an imperative summary, such
as:

``` text
fix: handle empty booking filters
feat: add review status indicator
docs: document local setup
```

Commit only after reviewing the staged diff:

``` bash
git commit -m "type: concise description"
```

If hooks fail, investigate and address the issue when possible. Do not
bypass hooks with `--no-verify` unless the user explicitly authorizes
it. If Git identity is missing or the commit is rejected, report the
blocker rather than changing global Git configuration without
permission.

After committing, verify:

``` bash
git status --short --branch
git log -1 --oneline
```

### 6. Push the branch

Push the newly created branch and configure its upstream:

``` bash
git push -u origin HEAD
```

Use the repository's correct remote name if it is not `origin`. Do not
force-push. If the push is rejected, inspect the reason and resolve it
safely; do not overwrite remote history.

### 7. Open a PR or MR

Create the review request using the detected host's CLI.

For GitHub:

``` bash
gh pr create --base <base-branch> --head <branch-name> --title "<title>" --body "<body>"
```

For GitLab:

``` bash
glab mr create --target-branch <base-branch> --source-branch <branch-name> --title "<title>" --description "<description>"
```

Use the actual branch and base names, not the placeholders. Prefer the
project's PR/MR template if one exists. Include:

-   **Summary:** what changed and why.
-   **Testing:** checks actually run and their outcomes.
-   **Notes:** relevant risks, follow-up work, or items not tested.

Keep the title concise and aligned with the commit. Do not claim tests
passed if they were not run. If the hosting CLI cannot create the
request, provide the verified URL or explain the exact blocker. Do not
fabricate a PR/MR URL.

Before creating the request, check whether a PR/MR already exists for
the branch to avoid duplicates. If one exists, report its URL instead of
creating another.

### 8. Verify and report

Confirm the final state using Git and the hosting CLI where possible.
Report:

-   Branch name
-   Commit hash and message
-   Push result
-   PR/MR URL and title, if successfully created
-   Tests or checks run and their outcomes
-   Any blocker or remaining action

Do not state that the work is complete until each completed step has
been verified.

## Suggested PR/MR body

``` markdown
## Summary
- Describe the main changes.
- Explain the reason for the change.

## Testing
- `command`: passed / failed / not run

## Notes
- Risks, limitations, or follow-up items.
```

## Completion checklist

-   [ ] Repository instructions reviewed
-   [ ] Existing changes and target files identified
-   [ ] New task-specific branch created
-   [ ] Only intended files staged
-   [ ] Staged diff reviewed
-   [ ] Commit created and verified
-   [ ] Branch pushed with upstream configured
-   [ ] Existing PR/MR checked
-   [ ] PR/MR opened and URL verified, or blocker clearly reported
-   [ ] Test results and final status summarized
