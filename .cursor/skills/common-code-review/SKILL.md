---
name: code-review
description: Use when reviewing code changes, pull requests, or verifying implementations against project standards and acceptance criteria. Triggers on review requests, PR reviews, diff analysis, or code quality checks.
---

# Code Reviewing

Dispatch the `code-reviewer` subagent for all code reviews. It runs in isolated context with the full review methodology (conventions discovery, category-based analysis, severity output, no code fixes).

## When to Use

- Reviewing pull requests or code diffs
- Verifying implementation against a plan or acceptance criteria
- Post-implementation review before merge
- After completing a major implementation step

## How to Dispatch

Use the Task tool to launch the `code-reviewer` subagent. Always pass:

1. **What to review** — PR number, branch diff command, or specific file paths
2. **Acceptance criteria / plan path** — if available, so the subagent can verify alignment
3. **Convention sources** — mention `.cursor/rules/` if the project has them

### Example dispatch prompt

```
Review the code changes on the current branch against main.

- Diff: `git diff main...HEAD`
- Implementation plan: docs/plans/feature-x.md (Task 3)
- Project conventions are in .cursor/rules/*.mdc

Follow your full review process. Return the structured review output.
```

## What the Subagent Does

1. Discovers project conventions (`.cursor/rules/`, `AGENTS.md`, `*.instructions.md`)
2. Scopes to changed code only (diff-based)
3. Verifies acceptance criteria / plan alignment if provided
4. Reviews by category: Security, Correctness, Input validation, Error handling, Architecture, Code quality, Performance, Conventions, Accessibility, Testing
5. Runs tests and linting
6. Outputs findings by severity (Critical > High > Medium > Low) with a verdict

**Key constraint:** The subagent provides directional guidance only — never ready-made code fixes.

## When NOT to Dispatch

- Trivial single-file changes where inline review is faster
- Quick lint/type-check only — just run the tools directly
