---
name: git-fix-issue
description: Fetch a GitHub issue/PR or GitLab issue/MR, understand what needs fixing or building, draft an implementation plan, implement it on the current branch, then offer to open a PR/MR. Use this whenever the user references an issue, bug, ticket, feature request, PR, or MR (by number, URL, or "the issue linked to this PR") and wants it implemented or fixed — e.g. "fix issue #42", "implement the enhancement from this MR", "work on GH-1234", "grab the issue this PR closes and fix it", or points at a github.com/gitlab.com issue/PR/MR link. Prefer this skill over ad-hoc git+gh commands whenever the task starts from a tracked issue or PR/MR.
---

# Issue Implementer

Turn a tracked issue (or the issue behind a PR/MR) into a working code change on the current branch, with a plan the user approves first and an optional PR/MR at the end.

## Workflow overview

1. **Resolve** the source (issue / PR / MR) and read its full content.
2. **Understand** the codebase area involved.
3. **Plan** the fix or enhancement and get user approval.
4. **Implement** on the current branch.
5. **Verify** (build/tests/lint if available).
6. **Offer** to open a PR/MR — only create one if the user says yes.

Do not skip step 3's approval. Plans are cheap; wrong implementations are expensive.

---

## Step 1 — Resolve the source

The user may give you an issue number, a PR/MR number, a full URL, or a phrase like "the issue this PR closes." First figure out which platform and which host.

**Detect the platform.** Check what remote the current repo uses:

```bash
git remote get-url origin 2>/dev/null
```

- Contains `github.com` (or a GitHub Enterprise host) → GitHub, use `gh`.
- Contains `gitlab` → GitLab, use `glab`.
- If a URL was given, trust the URL's host over the remote.

**Check CLI availability, fall back to API.** Prefer the CLI because auth is already handled:

```bash
command -v gh    # GitHub
command -v glab  # GitLab
```

If the needed CLI is missing or not authenticated (`gh auth status` / `glab auth status` fails), fall back to the REST API with a token from the environment (`GITHUB_TOKEN`, `GH_TOKEN`, or `GITLAB_TOKEN`). See `references/api-fallback.md` for exact endpoints and curl commands. If no CLI and no token, stop and ask the user how they'd like to provide access (or to paste the issue text).

**Fetch the content.** See `references/cli-commands.md` for the full command reference. The essentials:

GitHub:
```bash
gh issue view <n> --json title,body,labels,comments,url
gh pr view <n> --json title,body,url,closingIssuesReferences,files,headRefName
```

GitLab:
```bash
glab issue view <n>
glab mr view <n>
```

**Resolve linked issues.** When the user points at a PR/MR, the real requirements often live in the issue it closes.
- GitHub: `closingIssuesReferences` in the `gh pr view --json` output lists auto-linked issues; also scan the PR body for `Closes #N` / `Fixes #N`.
- GitLab: scan the MR description for `Closes #N` / `Related to #N`, or use `glab mr view` and read the linked items.
Fetch those issues too and treat them as the source of truth for what "done" means.

---

## Step 2 — Understand the codebase

Before planning, ground yourself in the actual code:

- Read the issue/PR/MR fully, including comments — they often contain the real constraints, repro steps, or a maintainer's preferred approach.
- Locate the relevant files (grep for symbols, error strings, or feature names mentioned).
- For a bug: reproduce or trace the faulty path. For an enhancement: find where the new behavior should hook in and match existing patterns.
- Note the project's conventions (test framework, lint config, commit style) so your change fits in.

Confirm you're on the intended branch and it's clean enough to work on:

```bash
git branch --show-current
git status --short
```

If there are unrelated uncommitted changes, mention them and ask before proceeding — don't silently mix them into the work.

---

## Step 3 — Plan and get approval

Produce a concise, concrete plan. Structure it like this:

```
## Plan: <issue title>

**Source:** <issue/PR/MR ref + URL>
**Goal:** <one-sentence restatement of what needs to happen>

**Changes:**
1. <file> — <what changes and why>
2. <file> — <what changes and why>

**Tests:** <what you'll add/update, or why none>
**Risks / open questions:** <anything ambiguous, or "none">
```

Then ask the user to confirm or adjust before you write any code. If the issue is ambiguous (conflicting comments, unclear acceptance criteria), surface the ambiguity here rather than guessing. Keep the plan proportional to the change — a one-line typo fix doesn't need a six-part plan.

---

## Step 4 — Implement on the current branch

Implement the approved plan on the **current branch** — do not create a new branch unless the user asks. Match existing code style. Make focused commits with messages that reference the issue:

```bash
git commit -m "<type>: <summary> (#<issue-number>)"
```

Use the repo's existing commit convention if it has one (Conventional Commits, a `[JIRA-123]` prefix, etc.) — check recent `git log` to infer it.

---

## Step 5 — Verify

Run whatever the project provides, in this rough order of preference — tests, then lint/format, then a build. Detect from the repo (e.g. `package.json` scripts, `Makefile`, `pytest`, `cargo test`, `go test`). If a check fails, fix it before moving on. If you can't run checks (no test command, missing deps), say so explicitly rather than implying it's verified.

---

## Step 6 — Offer a PR/MR (don't auto-create)

Summarize what changed, then **ask** whether to open a PR/MR. Only proceed on a clear yes. When creating one, write a description that links the issue so it auto-closes:

GitHub:
```bash
gh pr create --title "<title>" --body "Closes #<issue>. <summary>" --base <default-branch>
```

GitLab:
```bash
glab mr create --title "<title>" --description "Closes #<issue>. <summary>" --target-branch <default-branch>
```

If the current branch has no upstream, push it first (`git push -u origin HEAD`). If the API fallback is in use, see `references/api-fallback.md` for the create-PR/MR endpoints. Report the resulting URL.

---

## Guardrails

- **Never force-push, never touch the default branch directly, never delete branches** unless the user explicitly asks.
- **Approval gate:** don't implement before the plan is approved; don't open a PR/MR before the user says yes.
- **Scope discipline:** implement what the issue asks. If you spot adjacent problems, note them for the user instead of expanding the change.
- **Secrets:** never print tokens. Read them from the environment; don't echo them back.
- **Uncertainty:** if you can't confidently resolve the issue, reproduce the bug, or find the right code, say so and ask — a wrong "fix" is worse than a question.