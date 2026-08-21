---
name: git-issue
description: Draft a clean, well-structured issue as markdown text for a code-hosting tracker (GitHub, GitLab, Jira, or similar). Use this whenever the user wants to file, write, open, or draft an issue, bug report, feature request, ticket, or task — including phrasings like "make an issue for this", "write up a bug report", "turn this into a ticket", "I need to file something for the flaky test", or when they paste an error/stack trace/PR discussion and ask for it to become an issue. Also use it when they hand you loose notes, a Slack thread, or a chat log and want a properly formatted issue out of it. Produces markdown ready to paste into the tracker's issue field, and after drafting asks whether to actually post/create the issue — posting it directly if a GitHub/GitLab connector or authenticated CLI is available, otherwise handing over a ready-to-run command.
---

# git-issue

Turn a rough problem, request, or pile of notes into a clean issue that a maintainer can triage in seconds and act on without a follow-up round of questions. The output is **markdown text** the user pastes into their tracker — you are not running `gh`, `glab`, or any API.

## The mental model

A good issue answers three questions fast: *what's wrong or wanted*, *how do I reproduce or understand it*, and *how do I know when it's done*. Everything below serves that. A maintainer skimming twenty issues should grasp yours from the title alone and know the next step from the body. Padding, ceremony, and restating the obvious all work against that, so keep it tight.

## Workflow

1. **Classify the issue.** Read what the user gave you and pick the type — bug, feature/enhancement, task/chore, docs, or question. If it's genuinely ambiguous, make your best guess and note it rather than stalling to ask; the user can correct a draft faster than they can answer an interview.
2. **Pull out what you already have.** Error text, stack traces, versions, repro steps, file paths, PR/commit links, the desired behavior — harvest these from whatever the user pasted. Don't make the user re-supply things that are already on screen.
3. **Flag real gaps, don't invent.** If a bug is missing its reproduction steps or a feature is missing its motivation, leave a short `> _TODO: ..._` marker in that spot rather than fabricating plausible-sounding details. A fabricated repro step is worse than an honest gap — it sends the maintainer down a wrong path.
4. **Write the title, then the body** using the matching template below.
5. **Suggest metadata.** Propose labels, and where relevant assignee/milestone/priority, as a short block. These are suggestions the user can keep or drop.
6. **Deliver as a `.md` file** so it's easy to copy or hand off, and mention anything you marked TODO so the user knows what to fill in.
7. **Ask whether to post it.** After delivering the draft, ask the user whether they'd like to actually create/post the issue on their tracker, or just keep the draft. Don't post anything unprompted — the draft may still have TODOs, and the user may want to tweak it first. See "Posting the issue" below for how to handle a yes.

## Titles

The title is the part read most and skimmed hardest. Make it a specific, scannable summary of the *problem or request*, not a vague category.

- Lead with the concrete symptom or ask: "Login button unresponsive on Safari after session timeout", not "Login bug".
- Keep it roughly under ~70 characters so it doesn't truncate in list views.
- Skip trailing punctuation and filler like "Issue with..." or "Problem where...".
- A conventional-commit-style prefix (`bug:`, `feat:`, `docs:`) is fine if the project uses them, but don't impose one unprompted.

**Examples**

Notes: "the CSV export is broken, gives empty file when there are unicode names" → `CSV export produces empty file when rows contain non-ASCII names`

Notes: "we should let people log in with google" → `Add Google OAuth as a sign-in option`

Notes: "readme still says node 14" → `README lists outdated Node 14 requirement`

## Body templates

Use the template matching the type. Include a section only when it carries weight — an empty "Screenshots" heading is noise. Where the user hasn't supplied something important, drop in a `> _TODO: ..._` line instead of guessing.

### Bug

```markdown
## Summary
<one or two sentences: what's broken and the impact>

## Steps to reproduce
1. <step>
2. <step>
3. <step>

## Expected behavior
<what should happen>

## Actual behavior
<what happens instead, including exact error text / stack trace in a code block>

## Environment
- Version/commit: <...>
- OS / browser / runtime: <...>

## Notes
<links to logs, related issues, first-guess at cause — omit if none>
```

### Feature / enhancement

```markdown
## Problem / motivation
<what the user is unable to do today, and why it matters — the pain, not the solution>

## Proposed solution
<what should exist; a sketch is fine, exhaustive design is not required>

## Acceptance criteria
- [ ] <observable, checkable outcome>
- [ ] <...>

## Alternatives considered
<other approaches and why they're worse — omit if none>

## Notes
<mockups, links, prior art — omit if none>
```

### Task / chore

```markdown
## Goal
<what needs doing and why now>

## Scope / checklist
- [ ] <concrete sub-task>
- [ ] <...>

## Out of scope
<what this deliberately does not cover — omit if obvious>
```

### Docs

```markdown
## What's wrong or missing
<the page/section and the specific inaccuracy or gap>

## Where
<file path, URL, or heading>

## Suggested fix
<the correct information, or a sketch of what to add>
```

### Question / discussion

```markdown
## Question
<the actual question, stated plainly>

## Context
<what you're trying to do and what you've already tried or read>
```

## Metadata block

After the body, add a short suggestions block. Keep labels to the ones that clearly apply — a wall of labels is as useless as none.

```markdown
---
**Suggested metadata**
- Labels: `bug`, `area/export`
- Priority: <if the project tracks it>
- Milestone / assignee: <only if the user indicated one>
```

Frame these as suggestions in your message, not decisions — you don't know the project's label taxonomy, so you're proposing sensible defaults.

## Platform notes

GitHub-flavored markdown is the safe default and renders acceptably almost everywhere. Adjust only when the user names a platform — see `references/platforms.md` for the specifics (task-list support, `@`/`#` autolink behavior, Jira's non-markdown wiki syntax, and how each treats the metadata block). Read it whenever the user mentions GitLab or Jira, or asks for a `.jira`/wiki-syntax version.

## Posting the issue

The skill's job is to draft — it doesn't have direct tracker access of its own. So once the draft is ready, ask the user how they want to proceed. Phrase it as a genuine choice, since posting is irreversible-ish and the draft may still need edits:

> Want me to post this as a new issue, or would you rather keep it as a draft to review first?

Handle the answer like this:

- **Keep as draft** — do nothing further; the `.md` file is the deliverable.
- **Post it, and a tracker tool/connector is available** (e.g. a GitHub/GitLab MCP connector, or a shell where `gh`/`glab` is authenticated) — use it to create the issue, then report back the new issue's URL/number. Confirm the target repo first if it's ambiguous.
- **Post it, but no tracker access is wired in** — you can't create it directly, so hand the user a ready-to-run command they can paste. Fill in the real title/body/labels; use the `.md` file for the body so nothing gets mangled:
  - GitHub: `gh issue create --title "<title>" --body-file <file>.md --label bug`
  - GitLab: `glab issue create --title "<title>" --description "$(cat <file>.md)" --label bug`
  Mention that they need the CLI installed and authenticated, and that they can drop the `--label` flags if the names don't match their project.

Before posting anything, make sure outstanding `> _TODO:_` markers are resolved or the user has explicitly okayed posting with them in place — filing an issue with `TODO: paste the traceback` still sitting in it defeats the purpose.

## Output

Write the finished issue to a `.md` file and present it. Lead with the title as an H1 or a clearly labeled "Title:" line so the user can grab it separately from the body (most trackers have a distinct title field). Keep your chat message brief: note the issue type, and call out any `TODO` markers the user still needs to fill in.
