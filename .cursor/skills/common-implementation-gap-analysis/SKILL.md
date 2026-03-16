---
name: implementation-gap-analysis
description: Use when analyzing a codebase against requirements or a plan to understand what already exists, what needs modification, and what needs to be built from scratch. Use as part of create-implementation-plan Step 3, or to verify progress against an existing plan mid-implementation.
---

# Implementation Gap Analysis

Produces a structured **Reuse / Modify / Create** inventory by comparing requirements to the current codebase. Output feeds directly into the file structure map in `create-implementation-plan` Step 5.

## When to Use

- **Pre-planning** — as part of `create-implementation-plan` Step 3 (Analyze the codebase)
- **Mid-implementation** — verify which plan tasks are complete vs pending when resuming work
- **Scope validation** — confirm a feature is truly missing before building it

## Process

```
Analysis progress:
- [ ] Step 1: Gather inputs
- [ ] Step 2: Run codebase analysis per feature area
- [ ] Step 3: Classify findings into Reuse / Modify / Create
- [ ] Step 4: Produce the report
```

### Step 1: Gather inputs

Collect the source of truth for what needs to be built:

- Spec file (e.g. `docs/specs/*.md`) or requirements from conversation
- Existing plan file if verifying progress (`docs/plans/*.md`)
- Technical context from `.cursor/rules/`, `package.json`, `tsconfig.json`

Extract concrete requirements with IDs (REQ-001, etc.). Assign IDs if none exist.

### Step 2: Run codebase analysis per feature area

Group requirements by feature area (e.g. auth, dashboard, API layer).

Launch one `codebase-analyzer` subagent per group — run them in parallel:

```
Task(
  subagent_type="codebase-analyzer",
  prompt="Analyze codebase for these requirements:
    - REQ-001: [requirement]
    - REQ-002: [requirement]
  Technical context: [stack, conventions from .cursor/rules]
  Search scope: [target directories if known]",
  readonly=true
)
```

Merge results from all subagents into a single inventory.

### Step 3: Classify findings

| Category | What it includes |
| --- | --- |
| **Reuse as-is** | Components, utils, types that already satisfy the requirement |
| **Modify** | Code that partially satisfies — include file path + line range |
| **Create** | New files required — propose paths following existing conventions |

For **mid-implementation mode**: also classify each plan task as Complete / In Progress / Not Started based on code evidence.

### Step 4: Produce the report

```markdown
## Gap Analysis Report

### Requirements Coverage

| Req ID | Requirement | Status | Notes |
| ------ | ----------- | ------ | ----- |
| REQ-001 | User can log in with email/password | Modify | API stub exists, form missing |
| REQ-002 | Login form validates email format | Create | No schema found |

### File Inventory

#### Reuse as-is
- `src/components/ui/button.tsx` — Primary action button (REQ-001)
- `src/lib/http-client.ts` — Existing HTTP client

#### Modify
- `src/lib/api.ts:42-55` — Add login endpoint for REQ-001

#### Create
- `src/features/auth/login-form.tsx` — Login form (REQ-001, REQ-002)
- `src/features/auth/login-form.test.tsx` — Tests for login form
- `src/features/auth/login.schema.ts` — Validation schema (REQ-002)

### Implementation Status (mid-implementation only)

| Task | Status | Evidence |
| ---- | ------ | -------- |
| Task 1.1: Create login form | Complete | `src/features/auth/login-form.tsx` exists |
| Task 1.2: Add validation schema | Not Started | Schema file missing |
```

## Connection to create-implementation-plan

| Plan step | How gap analysis feeds it |
| --------- | ------------------------- |
| Step 3 — Analyze codebase | Run this skill; merge File Inventory into the Reuse/Modify/Create table |
| Step 5 — Map file structure | File Inventory maps directly to the Create/Modify/Reuse sections |
| Resuming work | Run in mid-implementation mode to find remaining tasks |
