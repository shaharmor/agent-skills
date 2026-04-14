---
name: ado-management
description: Use when accessing, checking, viewing, showing, creating, querying, reviewing, or managing Azure DevOps work items, pull requests, PR comments/threads, or linking entities. Triggers on ADO, Azure DevOps, work item, PR, pull request, PR <number>, PR comments, PR threads, review comments, resolve comments, WI, bug, task, enabling specification, check PR, show PR, view PR, review PR, or any az boards/az repos/az devops invoke command.
---

# Azure DevOps CLI Management

## Overview

Reference for managing Azure DevOps work items and pull requests using `az boards` and `az repos` CLI commands. Covers creation, linking, and common pitfalls.

## When to Use

- Creating work items (bugs, tasks, enabling specifications, etc.)
- Creating pull requests from a branch
- Linking work items to PRs
- Querying or updating ADO entities

## Prerequisites

The user's ADO settings (org, project, area path) are stored in CLAUDE.md files. Check both locations, preferring project-level over user-level:

1. **Project-level** (preferred): `.claude/CLAUDE.md` in the current repo
2. **User-level** (fallback): `~/.claude/CLAUDE.md`

If both exist, project-level settings take precedence. Only ask the user if neither file contains ADO settings.

## Quick Reference

| Operation | Command |
|-----------|---------|
| Create work item | `az boards work-item create` |
| Create PR | `az repos pr create` |
| Link work item to PR | `az repos pr work-item add` |
| Update work item | `az boards work-item update` |
| Show work item | `az boards work-item show` |
| List PRs | `az repos pr list` |
| List PR threads | `az devops invoke --area git --resource pullRequestThreads` |
| Reply to PR thread | `az devops invoke --area git --resource pullRequestThreadComments --http-method POST` |
| Update PR thread | `az devops invoke --area git --resource pullRequestThreads --http-method PATCH` |

## Creating Work Items

```bash
az boards work-item create \
  --type "Enabling Specification" \
  --title "Title here" \
  --area "<area-path>" \
  --iteration "<iteration-path>" \
  --org <org-url> \
  --project <project>
```

**Default type: `Enabling Specification`.** Use this unless the user specifies otherwise.

Other work item types: `Bug`, `Task`, `User Story`, `Feature`.

The work item ID is returned in the `id` field of the JSON response.

## Creating Pull Requests

```bash
az repos pr create \
  --repository <repo-name> \
  --source-branch <branch> \
  --target-branch <target-branch> \
  --title "PR title" \
  --description "$(cat <<'EOF'
## Summary
- Change description

EOF
)" \
  --org <org-url> \
  --project <project>
```

The PR ID is returned in the `pullRequestId` field.

## Linking Work Items to PRs

**Use `az repos pr work-item add` — NOT `az boards work-item relation add`.**

```bash
az repos pr work-item add \
  --id <pr-id> \
  --work-items <work-item-id> \
  --org <org-url>
```

### Why Not `relation add`?

`az boards work-item relation add --relation-type "ArtifactLink"` fails for PR links because artifact links require a `name` attribute that the CLI doesn't expose. Using `az repos pr work-item add` is the correct and reliable approach.

## PR Threads (Comments)

**`az repos pr thread` does not exist.** Use `az devops invoke` with the Git API directly.

### Listing PR threads

```bash
az devops invoke \
  --area git \
  --resource pullRequestThreads \
  --route-parameters project=<project> repositoryId=<repo-name> pullRequestId=<pr-id> \
  --organization <org-url> \
  --output json
```

Response is `{ "value": [...threads] }`. Each thread has:
- `id` — thread ID
- `status` — `active`, `fixed`, `closed`, etc.
- `threadContext.filePath` — file the comment is on (if inline)
- `comments[]` — array of comments with `author.displayName`, `content`, `commentType`

Filter out `commentType: "system"` to skip auto-generated status comments.

### Replying to an existing PR thread

**Use `pullRequestThreadComments` (NOT `pullRequestComments`).**

```bash
az devops invoke \
  --area git \
  --resource pullRequestThreadComments \
  --route-parameters project=<project> repositoryId=<repo-name> pullRequestId=<pr-id> threadId=<thread-id> \
  --http-method POST \
  --in-file <(echo '{"content": "Reply text here", "parentCommentId": 1, "commentType": 1}') \
  --organization <org-url> \
  --output none
```

- `content` — the reply text (supports markdown)
- `parentCommentId` — ID of the comment being replied to (typically `1` for the first comment in the thread)
- `commentType` — `1` for regular text

### Resolving/updating a PR thread

```bash
az devops invoke \
  --area git \
  --resource pullRequestThreads \
  --route-parameters project=<project> repositoryId=<repo-name> pullRequestId=<pr-id> threadId=<thread-id> \
  --http-method PATCH \
  --in-file <(echo '{"status": "fixed"}') \
  --organization <org-url> \
  --output none
```

Valid status values: `active`, `fixed`, `wontFix`, `closed`, `byDesign`, `pending`.

**Note:** Must resolve threads one at a time — the API does not support batch updates. Loop over thread IDs:

```bash
for tid in 123 456 789; do
  az devops invoke \
    --area git \
    --resource pullRequestThreads \
    --route-parameters project=<project> repositoryId=<repo-name> pullRequestId=<pr-id> threadId=$tid \
    --http-method PATCH \
    --in-file <(echo '{"status": "fixed"}') \
    --organization <org-url> \
    --output none
done
```

## Reading PR Diffs and File Content

### Preferred: Use local git commands

If the current working directory is a clone of the PR's repository, **always prefer local git commands** over the ADO API. This is faster, simpler, and gives you full diff/log capabilities. You do NOT need a clean working tree or to check out the branch — just fetch and diff remote refs:

Get the PR's target branch from `az repos pr show --id <pr-id>` (the `targetRefName` field, strip the `refs/heads/` prefix). Then use it for diffs:

```bash
# Fetch both branches (does not affect working tree)
git fetch origin <source-branch> <target-branch>

# See the full diff (what the PR changes vs its target)
git diff origin/<target-branch>...origin/<source-branch>

# See just the list of changed files
git diff --name-status origin/<target-branch>...origin/<source-branch>

# See the PR's commits
git log origin/<target-branch>..origin/<source-branch> --oneline

# Read a specific file from the PR branch without checkout
git show origin/<source-branch>:path/to/file.ts
```

The three-dot `...` syntax shows changes on the branch since it diverged from the target — exactly what a PR diff should be.

**Only fall back to the ADO API** (below) when:
- The repo is not cloned locally
- The PR is in a different repository than the current working directory

### ADO API fallback

**`az repos pr diff` does not exist.** There is no built-in CLI command to get a PR diff.

### Getting PR iterations (commit history)

Use the `pullRequestIterations` resource to see all push iterations in a PR:

```bash
az devops invoke \
  --area git \
  --resource pullRequestIterations \
  --route-parameters project=<project> repositoryId=<repo-name> pullRequestId=<pr-id> \
  --organization <org-url> \
  --output json
```

Response is `{ "value": [...iterations] }`. Each iteration has:
- `id` — iteration number
- `description` — commit message
- `sourceRefCommit.commitId` — the commit SHA

### Reading a file from a specific branch

**Two-step process:** First get the blob SHA via the items API, then download the blob content.

**Step 1 — Get blob SHA:**

```bash
az devops invoke \
  --area git \
  --resource items \
  --route-parameters project=<project> repositoryId=<repo-name> \
  --query-parameters 'path=/path/to/file.ts&versionDescriptor.version=<branch-name>&versionDescriptor.versionType=branch' \
  --organization <org-url> \
  --output json
```

The blob SHA is in the `objectId` field of the response.

**Step 2 — Download blob content:**

```bash
az devops invoke \
  --area git \
  --resource blobs \
  --route-parameters project=<project> repositoryId=<repo-name> sha1=<blob-sha> \
  --query-parameters '$format=text' \
  --organization <org-url> \
  --out-file /tmp/output-file.ts
```

**IMPORTANT:** The blobs API returns raw text, not JSON. You **must** use `--out-file` to write to a file, then read the file. Without `--out-file`, the CLI errors with: `Response is not json, you need to provide --out-file`.

### Comparing two branches (PR diff)

To see the actual diff, download both versions of each file and use `diff -u`:

```bash
# Download file from source branch
az devops invoke --area git --resource items \
  --route-parameters project=<project> repositoryId=<repo-name> \
  --query-parameters 'path=/path/to/file.ts&versionDescriptor.version=<source-branch>&versionDescriptor.versionType=branch' \
  --organization <org-url> --output json
# → get objectId, then download blob to /tmp/file_pr.ts

# Download file from target branch (default branch)
az devops invoke --area git --resource items \
  --route-parameters project=<project> repositoryId=<repo-name> \
  --query-parameters 'path=/path/to/file.ts&versionDescriptor.version=<default-branch>&versionDescriptor.versionType=branch' \
  --organization <org-url> --output json
# → get objectId, then download blob to /tmp/file_target.ts

diff -u /tmp/file_target.ts /tmp/file_pr.ts
```

### Getting list of changed files in a PR

Use the iteration changes API to see which files were modified:

```bash
az devops invoke \
  --area git \
  --resource pullRequestIterationChanges \
  --route-parameters project=<project> repositoryId=<repo-name> pullRequestId=<pr-id> iterationId=<iteration-id> \
  --organization <org-url> \
  --output json
```

To get changes across all iterations, use iterationId from the latest iteration (highest `id` from the iterations API).

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `az boards work-item relation add` to link PRs | Use `az repos pr work-item add` instead |
| Using `az repos pr thread list` to read PR comments | This command does not exist. Use `az devops invoke --area git --resource pullRequestThreads` |
| Using `pullRequestComments` to reply to a thread | Wrong resource name. Use `pullRequestThreadComments` |
| Trying to batch-update multiple threads in one call | API only supports one thread per request. Loop over thread IDs |
| Forgetting `--org` flag | Always pass `--org <org-url>` |
| Forgetting `--project` on commands that need it | Required for `az boards` and `az repos pr create`; not needed for `az repos pr work-item add` |
| Using backslashes in area/iteration paths on some shells | Quote the path: `"Area\Path\Here"` |
