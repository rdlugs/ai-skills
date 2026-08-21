# Platform-specific formatting

Read this when the user names GitHub, GitLab, or Jira, or asks for a platform-specific version. GitHub-flavored markdown is the default; the differences below matter mainly for GitLab and Jira.

## GitHub

The default. Standard GitHub-flavored markdown works as written.
- Task lists (`- [ ]`) render as interactive checkboxes.
- `#123` autolinks to issues/PRs in the same repo; `@user` mentions a user. Only write these when the user gives real numbers/handles — a made-up `#42` creates a misleading cross-link.
- Fenced code blocks with language hints (```` ```python ````) get syntax highlighting.
- The metadata block at the bottom is informational only; labels/assignee/milestone are set via the UI or `gh`, not the body. Keep it as a suggestion list.

## GitLab

Very close to GitHub markdown, with a few conveniences:
- Task lists work the same way.
- Quick actions: GitLab reads slash-commands placed on their own line in the description, e.g. `/label ~bug`, `/assign @user`, `/milestone %"v1.2"`, `/estimate 2d`. If the user is on GitLab and wants labels applied automatically, offer these instead of (or alongside) the suggestion block, since GitLab will execute them on submit. Only emit them when the user confirms the exact label/milestone names — a wrong `/label` silently no-ops.
- `#123` refers to issues, `!123` to merge requests, `%123` to milestones, `~123`/`~"name"` to labels.

## Jira

Jira does **not** use markdown. Its classic editor uses wiki markup and the new editor uses a rich-text model, so pasted markdown often renders literally.
- If the user needs Jira wiki syntax, translate: headings become `h2.`, bold is `*text*`, code blocks are `{code}...{code}` or `{code:java}...{code}`, bullets are `*`, numbered lists are `#`, and checklists aren't native (use `(/)` / `(x)` emoticons or a plain list).
- Offer this only when the user explicitly wants Jira syntax; otherwise the markdown draft is usually fine to paste into Jira's rich-text editor, which converts common markdown on the fly.
- Jira "issue type" (Bug/Story/Task/Epic) maps naturally onto the classification in SKILL.md; mention the matching type in your message.

## When the platform is unknown

Stick with GitHub-flavored markdown and keep the metadata as a plain suggestion block. Don't emit GitLab quick-actions or Jira wiki syntax speculatively — they look like clutter on the wrong platform.
