---
name: create-github-pull-request-from-specification
description: Create GitHub Pull Request for feature request from specification file using pull_request_template.md template. Use when user asks to create a PR, open a pull request, or submit changes for review.
---

# Create GitHub Pull Request from Specification

## Prerequisites

Ensure `gh` is installed and authenticated. If not, install it globally and ask the user to authenticate.

## Workflow

### 1. Validate branch

```bash
git branch --show-current
git symbolic-ref refs/remotes/origin/HEAD --short | cut -d/ -f2
```

Abort if current branch equals default branch.

### 2. Check for existing PR

```bash
gh pr list --head <current-branch>
```

If a PR already exists, skip to step 5.

### 3. Load PR template

Read `${workspaceFolder}/.github/pull_request_template.md` to get the required body structure.

### 4. Generate diff & compose body

```bash
git diff origin/<default-branch>...HEAD
```

Fill the template with:

- Summary of key changes (concise bullet points)
- All significant changes included

### 5. Create PR

```bash
gh pr create \
  --title "<type>: <short description>" \
  --body "<filled-template>" \
  --base <default-branch>
```

Title format: `<type>: <short description>` (e.g., `feat: add user auth`)

### 6. Self-assign

```bash
gh pr edit <pr-number> --add-assignee @me
```

### 7. Return PR URL to user

## Requirements

- Single PR per specification
- Template sections must be filled (no empty placeholders)
- Verify no existing PR before creation
- Abort if already on default branch
