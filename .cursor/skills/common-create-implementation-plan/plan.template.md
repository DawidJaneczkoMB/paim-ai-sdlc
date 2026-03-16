# [Task Name] — Implementation Plan

## Task Details

| Field       | Value                                    |
| ----------- | ---------------------------------------- |
| Title       | [task title]                             |
| Description | [brief description]                      |
| Priority    | [priority]                               |
| Source      | [link to spec, research file, or ticket] |

## Goal

[One sentence: what this plan builds and why it matters.]

## Architecture

[2-3 sentences about the technical approach, key patterns, and design decisions.]

## Requirements & Constraints

- **REQ-001**: [requirement derived from spec]
- **CON-001**: [constraint — e.g., must use existing auth middleware]
- **SEC-001**: [security requirement]
- **PAT-001**: [pattern to follow from project conventions]

## Current Implementation Analysis

### Reuse As-Is

Existing code that will be used without changes:

- `[component/function]` — `[file-path]` — [how it will be used]

### Modify

Existing code that needs changes:

- `[component/function]` — `[file-path:lines]` — [what needs to change]

### Create

New code that needs to be built:

- `[component/function]` — `[proposed-file-path]` — [what it does]

## File Structure

### Create

- `[exact/path/to/new-file.ts]` — [single responsibility description]
- `[exact/path/to/new-file.test.ts]` — [what it tests]

### Modify

- `[exact/path/to/existing.ts:line-range]` — [what changes]

## Open Questions

| #   | Question   | Answer   | Status          |
| --- | ---------- | -------- | --------------- |
| 1   | [question] | [answer] | Resolved / Open |

## Implementation Plan

### Phase 1: [Phase Name]

**Goal:** [What this phase achieves. Must be independently testable.]

#### Task 1.1 — [CREATE/MODIFY] [Task Name]

**Files:**

- Create: `exact/path/to/file.ts`
- Test: `exact/path/to/file.test.ts`

**Definition of Done:**

- [ ] Failing test written and verified
- [ ] Implementation passes all tests
- [ ] Types check, lint clean
- [ ] Committed with conventional message

**Steps:**

- [ ] **Step 1: Write the failing test**

```[language]
// complete, copy-pasteable test code
```

- [ ] **Step 2: Run test — verify failure**

Run: `[exact command]`
Expected: FAIL — "[expected failure reason]"

- [ ] **Step 3: Implement**

```[language]
// complete, copy-pasteable implementation code
```

- [ ] **Step 4: Run tests — verify pass**

Run: `[exact command]`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add [files]
git commit -m "[type](scope): [description]"
```

#### Task 1.2 — [CREATE/MODIFY] [Task Name]

[Same structure as above]

### Phase 2: [Phase Name]

[Same structure as above]

## Alternatives Considered

- **ALT-001**: [approach] — [why not chosen]

## Security Considerations

- [security consideration relevant to this task]

## Quality Assurance

Acceptance criteria checklist:

- [ ] [criterion from spec]
- [ ] All tests pass
- [ ] No type errors
- [ ] No lint violations
- [ ] Code review completed

## Risks & Assumptions

- **RISK-001**: [risk and mitigation]
- **ASSUMPTION-001**: [assumption]

## Improvements (Out of Scope)

Potential improvements not part of this task:

- [improvement description]

## Changelog

| Date   | Change               |
| ------ | -------------------- |
| [date] | Initial plan created |
