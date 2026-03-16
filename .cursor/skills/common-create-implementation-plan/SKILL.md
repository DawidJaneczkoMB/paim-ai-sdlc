---
name: create-implementation-plan
description: Use when creating implementation plans for features, refactoring, upgrades, or architecture changes. Triggers on plan requests, feature breakdowns, task decomposition, or when spec/requirements need converting into actionable steps.
---

# Creating Implementation Plans

Write thorough, executable implementation plans that an engineer with zero codebase context can follow autonomously. Every task must specify exact file paths, include complete code, and have verification steps.

**Core principles:** DRY. YAGNI. TDD. Frequent commits. Bite-sized tasks.

## Scope Check

If the spec covers multiple independent subsystems, it should be broken into separate plans — one per subsystem. Each plan must produce working, testable software on its own.

## Process

Copy this checklist and track your progress:

```
Planning progress:
- [ ] Step 1: Establish technical context
- [ ] Step 2: Understand the goal
- [ ] Step 3: Analyze the codebase
- [ ] Step 4: Ask clarifying questions
- [ ] Step 5: Map file structure
- [ ] Step 6: Decompose into phases and tasks
- [ ] Step 7: Write the plan document
- [ ] Step 8: Review the plan
```

### Step 1: Establish technical context

**MUST do before anything else.** Follow the technical context discovery priority:

1. Check `.cursor/rules/*.md` (or `.mdc`), `AGENTS.md`, `*.instructions.md` for coding standards, architecture patterns, naming conventions
2. Check `package.json`, `tsconfig.json`, or equivalent for stack and versions
3. Note testing framework, linting config, and existing test patterns

This context constrains every decision in the plan. Skip this and the plan will violate project conventions.

### Step 2: Understand the goal

Read the spec, research file, or user requirements. Identify:

- What is being built and WHY
- Acceptance criteria (explicit and implied)
- What is explicitly NOT in scope

### Step 3: Analyze the codebase

Use the `codebase-analyzer` subagent to search the codebase and classify existing code against requirements.

**How to invoke:**

1. Group requirements by feature area or module (e.g., auth, dashboard, API layer)
2. Launch one `codebase-analyzer` subagent per group — they run in parallel
3. Each invocation must include:
   - The requirements for that group (with IDs like REQ-001)
   - Technical context summary from Step 1 (stack, conventions, patterns)
   - Search scope — target directories to focus on (optional, omit to search everywhere)

**Example invocation (per group):**

```
Task(
  subagent_type="codebase-analyzer",
  prompt="Analyze codebase for these requirements:
    - REQ-001: User can log in with email/password
    - REQ-002: Login form validates email format
    - REQ-003: Failed login shows error message
  Technical context: React 19, TanStack Start, Vitest + RTL, co-located tests, kebab-case files.
  Search scope: src/features/auth/, src/components/ui/, src/lib/",
  readonly=true
)
```

4. Merge results from all subagents into a single Reuse/Modify/Create inventory

The subagent returns a structured table per category:

| Category        | What it finds                                                    |
| --------------- | ---------------------------------------------------------------- |
| **Reuse as-is** | Components, utils, types that already do what's needed           |
| **Modify**      | Code that's close but needs extension (file paths + line ranges) |
| **Create**      | New files needed (proposed paths following existing conventions) |

It also reports observed **naming conventions, file organization, common imports, and testing patterns** — feed these into Step 5.

### Step 4: Ask clarifying questions

Surface ambiguities. Do NOT proceed with assumptions. Common gaps:

- Missing error handling requirements
- Unclear data flow or state management
- Unspecified edge cases
- Security or accessibility requirements not mentioned

Wait for answers before continuing.

### Step 5: Map file structure

**Before writing tasks**, define every file that will be created or modified and what it's responsible for. This is where decomposition decisions get locked in.

Design principles:

- Each file has ONE clear responsibility
- Files that change together live together
- Follow existing project structure and naming conventions
- Prefer smaller, focused files over large multi-responsibility ones
- Split by responsibility, not by technical layer

```markdown
### Create

- `src/features/auth/login-form.tsx` — Login form component
- `src/features/auth/login-form.test.tsx` — Login form tests
- `src/features/auth/login.schema.ts` — Validation schema

### Modify

- `src/routes/index.tsx:15-30` — Add auth route
- `src/lib/api.ts:42-55` — Add login endpoint

### Reuse

- `src/components/ui/button.tsx` — Existing button component
- `src/lib/http-client.ts` — Existing HTTP client
```

This file map drives the task decomposition. Every file listed must appear in at least one task.

### Step 6: Decompose into phases and tasks

**Phase rules:**

- Each phase is independently runnable and immediately testable
- Each phase has all quality gates ready (tests pass, types check, lint clean)
- Phases build on each other but each produces working software
- A phase should be completable in one focused session

**Task granularity — each step is ONE action (2-5 minutes):**

| Step type          | What it is                                      |
| ------------------ | ----------------------------------------------- |
| Write failing test | Write the test with expected behavior           |
| Verify failure     | Run test, confirm it fails for the RIGHT reason |
| Implement          | Write minimal code to make the test pass        |
| Verify pass        | Run full test suite, confirm green              |
| Commit             | `git commit` with conventional message          |

**Every task MUST include:**

1. **Files** — exact paths for create/modify/test
2. **Code** — complete, copy-pasteable code (not "add validation")
3. **Commands** — exact run commands with expected output
4. **Definition of done** — checkboxes for verifiable criteria

See `./plan.template.md` for the full template structure.

### Step 7: Write the plan document

Save to `docs/plans/` directory using convention: `[purpose]-[component]-[version].md`

Purpose prefixes: `feature | refactor | upgrade | infrastructure | architecture | design`

### Step 8: Review the plan

Use the `plan-document-reviewer` subagent for each chunk (~1000 lines max):

1. Provide the plan file path and chunk to review
2. Provide the spec/requirements file path

If issues found → fix and re-review. If loop exceeds 3 iterations, surface to human.

## Task Structure Reference

````markdown
### Task N.M: [Action verb] [Component name]

**Files:**

- Create: `exact/path/to/file.ts`
- Modify: `exact/path/to/existing.ts:45-60`
- Test: `exact/path/to/file.test.ts`

- [ ] **Step 1: Write the failing test**

```typescript
import { render, screen } from '@testing-library/react';
import { LoginForm } from './login-form';

it('should show validation error when email is empty', async () => {
  render(<LoginForm />);
  // ... complete test code
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/features/auth/login-form.test.tsx`
Expected: FAIL — "LoginForm is not defined"

- [ ] **Step 3: Write minimal implementation**

```typescript
export function LoginForm() {
  // ... complete implementation code
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/features/auth/login-form.test.tsx`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/features/auth/login-form.tsx src/features/auth/login-form.test.tsx
git commit -m "feat(auth): add login form with email validation"
```
````

## Red Flags — Plans That Will Fail

| Red flag                      | Why it fails                                                       |
| ----------------------------- | ------------------------------------------------------------------ |
| "Implement the feature"       | Not a task. What file? What code? What test?                       |
| "Add validation similar to X" | Placeholder. Show the actual code.                                 |
| "Update styles as needed"     | Vague. Specify exact classes, files, changes.                      |
| No test steps between tasks   | TDD is not optional. Tests drive implementation.                   |
| Tasks without file paths      | Engineer wastes time figuring out WHERE to work.                   |
| Giant phases (10+ tasks)      | Break into smaller independently-testable phases.                  |
| No commit steps               | Changes pile up, making rollback and review harder.                |
| Empty template sections       | Every section exists for a reason. Fill it or justify removing it. |

## Common Mistakes

| Mistake                                  | Fix                                                                           |
| ---------------------------------------- | ----------------------------------------------------------------------------- |
| Skipping context discovery               | Always check `.cursor/rules` FIRST. Plans violating conventions get rejected. |
| File structure decided during tasks      | Map ALL files in Step 5 BEFORE writing tasks.                                 |
| Tests as afterthought                    | Tests FIRST, implementation SECOND.                                           |
| Overengineering                          | YAGNI. Only plan what the spec requires.                                      |
| Ignoring existing patterns               | Mirror existing codebase. Consistency > theory.                               |
| Monolithic plan for multi-system changes | Split into separate plans per subsystem.                                      |

## Connected Skills

- `code-review` — Final review phase after implementation
- `implementation-gap-analysis` — Verify what exists vs what needs building
- `technical-context-discovery` — Establish project conventions before planning
