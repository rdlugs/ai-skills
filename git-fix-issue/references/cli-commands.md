# CLI Command Reference

Full command reference for `gh` (GitHub) and `glab` (GitLab). Read this when you need exact flags. The CLIs handle auth automatically when the user is logged in, so prefer them over the API.

## Auth checks

```bash
gh auth status      # GitHub — exits non-zero if not logged in
glab auth status    # GitLab — exits non-zero if not logged in
```

If either fails, fall back to the API (see `api-fallback.md`).

---

## GitHub (`gh`)

### Read an issue
```bash
gh issue view <number> --json title,body,labels,comments,url,state,assignees
```
Human-readable version (includes comments):
```bash
gh issue view <number> --comments
```

### Read a PR (and find the issue it closes)
```bash
gh pr view <number> --json title,body,url,state,closingIssuesReferences,files,headRefName,baseRefName,comments
```
`closingIssuesReferences` is the reliable way to get auto-linked issues. It looks like:
```json
{"closingIssuesReferences":[{"number":42,"title":"...","url":"..."}]}
```
Also scan the PR `body` for `Closes #N`, `Fixes #N`, `Resolves #N` (case-insensitive) in case linking wasn't set up.

### See what a PR changed
```bash
gh pr diff <number>
gh pr view <number> --json files    # just the file list
```

### Cross-repo / explicit repo
Add `--repo owner/name` to any command to target a specific repo:
```bash
gh issue view 42 --repo octocat/hello-world --json title,body
```

### Create a PR
```bash
gh pr create \
  --title "<title>" \
  --body "Closes #<issue>. <summary of changes>" \
  --base <default-branch>          # omit to use repo default
```
Add `--draft` for a draft PR, `--web` to open in browser instead of creating headlessly. If the branch has no upstream, `gh pr create` will offer to push; to be safe push first: `git push -u origin HEAD`.

---

## GitLab (`glab`)

### Read an issue
```bash
glab issue view <number>
glab issue view <number> --comments
```
For machine-readable output, glab supports `-F json` on many commands:
```bash
glab issue view <number> -F json
```

### Read an MR (and find linked issues)
```bash
glab mr view <number>
glab mr view <number> --comments
glab mr view <number> -F json
```
GitLab links issues via the MR **description** using `Closes #N`, `Fixes #N`, `Related to #N`. Scan the description text. The `-F json` output includes the description and refs.

### See what an MR changed
```bash
glab mr diff <number>
```

### Explicit repo
Add `-R owner/name` (or the full project path) to target a specific project:
```bash
glab issue view 42 -R mygroup/myproject
```

### Create an MR
```bash
glab mr create \
  --title "<title>" \
  --description "Closes #<issue>. <summary of changes>" \
  --target-branch <default-branch> \
  --source-branch <current-branch>
```
Useful flags: `--draft`, `--remove-source-branch`, `--assignee @me`, `--fill` (auto-fill title/description from commits). Push the branch first if it has no upstream: `git push -u origin HEAD`.

---

## Detecting the default branch

Needed for the `--base` / `--target-branch` of a PR/MR:

```bash
# Works on both; reads the remote HEAD
git symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null | sed 's@^origin/@@'
# Fallback
gh repo view --json defaultBranchRef -q .defaultBranchRef.name   # GitHub
glab repo view -F json | grep -o '"default_branch":"[^"]*"'       # GitLab
```

## Parsing a URL into number + repo

- Issue: `https://github.com/OWNER/REPO/issues/NUMBER`
- PR:    `https://github.com/OWNER/REPO/pull/NUMBER`
- GitLab issue: `https://gitlab.com/GROUP/PROJECT/-/issues/NUMBER`
- GitLab MR:    `https://gitlab.com/GROUP/PROJECT/-/merge_requests/NUMBER`

Extract OWNER/REPO (or GROUP/PROJECT) and NUMBER, then pass them via `--repo` / `-R`.