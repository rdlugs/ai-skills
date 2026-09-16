---

name: mr-update-info
description: Update a GitLab merge request title and description based on the actual net changes between the source and target branches.
----------------------------------------------------------------------------------------------------------------------------------------

# Update Merge Request Metadata

Analyze the **net changes** of the current GitLab merge request and update its title and description so they accurately represent what the merge request will introduce when merged.

Do not rely primarily on commit messages or the existing MR description. Treat the actual diff between the target branch and source branch as the source of truth.

## When to Use

Use this skill when asked to:

* update an MR title
* update an MR description
* refresh MR metadata
* generate an MR title/description
* summarize the current MR
* update an MR based on its latest changes
* fix a stale MR description

## Objective

Given:

```text
target branch -> source branch
```

determine the effective changes that will be introduced by merging the source branch into the target branch.

Then:

1. Inspect the net diff.
2. Understand the purpose of the changes.
3. Generate an accurate MR title.
4. Generate a structured MR description.
5. Update the GitLab merge request.

## Source of Truth

Always prioritize the final branch diff.

Use:

```bash
git diff <target>...<source>
```

or, when working from the source branch:

```bash
git diff <target>...HEAD
```

Prefer the merge-base (`...`) diff because it represents what the source branch introduces relative to the target branch.

Before analyzing the diff, fetch the latest remote state when appropriate:

```bash
git fetch origin
```

For remote branches:

```bash
git diff origin/<target>...HEAD
```

Do NOT summarize:

```bash
git log <target>..<source>
```

as if it were the MR itself.

Commit history can provide context, but the final diff determines what belongs in the MR description.

## Analysis Process

First determine:

```text
source branch
target branch
GitLab project
merge request IID
```

When GitLab CLI is available, inspect the MR:

```bash
glab mr view
```

Otherwise obtain the information from the available GitLab integration or environment.

Then inspect the changed files:

```bash
git diff --stat origin/<target>...HEAD
git diff --name-status origin/<target>...HEAD
```

Inspect the actual patch:

```bash
git diff origin/<target>...HEAD
```

For large MRs, inspect important files individually rather than making conclusions only from `--stat`.

## Understand the Net Changes

Determine:

* what behavior was added
* what behavior was modified
* what behavior was removed
* bug fixes
* refactors
* database changes
* API changes
* UI changes
* configuration changes
* dependency changes
* tests added or modified
* operational or deployment implications

Ignore changes that existed temporarily in intermediate commits but are absent from the final diff.

For example:

```text
commit A: adds validation
commit B: removes validation
```

If the validation is absent from the final diff, do not mention it.

Likewise:

```text
commit A: implementation X
commit B: refactors X into Y
```

Describe the final implementation Y rather than narrating the intermediate history.

## Generate the Title

The title must summarize the primary purpose of the MR.

Prefer concise titles such as:

```text
feat: add customer approval workflow
fix: prevent duplicate booking report entries
refactor: simplify sales approval processing
perf: optimize CSI report queries
chore: update deployment configuration
```

Use the repository's existing title convention if one is clearly established.

Avoid titles such as:

```text
Update files
Fix issues
Changes
Multiple updates
Development changes
Latest changes
```

Do not include implementation details that are too granular for the MR title.

If the MR contains several related changes, describe their common purpose.

## Generate the Description

Use this structure unless the repository already has an MR template that should be preserved:

```markdown
## Summary

<1-3 sentences describing the purpose and outcome of the MR.>

## Changes

- <important net change>
- <important net change>
- <important net change>

## Technical Details

<Important implementation details that reviewers should know.>

## Testing

- <relevant tests or verification performed>

## Notes

<Migration, deployment, compatibility, follow-up, or reviewer notes when applicable.>
```

Do not create empty sections.

If there are no meaningful technical details, testing notes, or additional notes, omit those sections.

## Description Rules

Describe outcomes rather than listing files.

Bad:

```markdown
## Changes

- Updated UserController.php
- Modified user.js
- Changed routes.php
```

Better:

```markdown
## Changes

- Added server-side validation for customer account creation.
- Prevented duplicate account submissions.
- Added the account approval endpoint and corresponding frontend handling.
```

Mention filenames only when they help reviewers understand an architectural or implementation decision.

Keep descriptions concise enough to review quickly while covering all meaningful changes.

## Preserve Important Existing Information

Before replacing the MR description, inspect the existing description.

Preserve manually maintained information that cannot be reconstructed from the diff, including:

* issue references
* Jira ticket references
* testing instructions
* screenshots
* rollout instructions
* deployment requirements
* reviewer notes
* checklists

Do not blindly overwrite this information.

Integrate relevant existing information into the regenerated description.

Remove stale statements when the current diff clearly contradicts them.

## Safety Check Before Updating

Before modifying GitLab, verify that:

```text
1. The correct MR was identified.
2. The target branch is correct.
3. The source branch is correct.
4. The diff represents the MR's net changes.
5. The generated title reflects the primary purpose.
6. The description does not claim changes absent from the diff.
7. Important manually maintained MR information has been preserved.
```

If the MR cannot be identified reliably, stop and ask for the MR URL or IID.

## Update GitLab

When `glab` is available, update the MR using:

```bash
glab mr update <iid> \
  --title "<generated-title>" \
  --description "<generated-description>"
```

Prefer a temporary file for multiline descriptions when shell quoting could be unsafe:

```bash
cat > /tmp/mr-description.md <<'EOF'
<generated description>
EOF
```

Then use the appropriate `glab` option or GitLab API mechanism to submit the content safely.

If a GitLab MCP, API, or other authenticated GitLab integration is available, it may be used instead.

Never expose access tokens in commands, logs, or output.

## Final Response

After successfully updating the MR, report:

```text
Updated MR !<iid>

Title:
<new title>

Description updated based on the net changes between:
<target>...<source>
```

Briefly mention the major areas detected in the diff.

Do not dump the entire diff unless requested.

## Important Principle

The merge request describes the state being proposed for merge, not the journey used to produce it.

Therefore:

```text
MR metadata = final net diff + relevant human context
```

not:

```text
MR metadata = summary of commit messages
```
