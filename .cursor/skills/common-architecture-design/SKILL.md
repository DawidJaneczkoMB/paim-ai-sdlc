---
name: architecture-design
description: Use when designing solution architecture for a task — analyzing the codebase, resolving ambiguities, selecting patterns, and producing a structured implementation plan document.
---

# Architecture Design

## Architecture Design Process

Use the checklist below and track your progress:

```
Analysis progress:
- [ ] Step 1: Understand the goal of the task
- [ ] Step 2: Analyse the current codebase
- [ ] Step 3: Ask questions about ambiguous parts
- [ ] Step 4: Design a solution
- [ ] Step 5: Create an implementation plan document
```

**Step 1: Understand the goal of the task**
Thoroughly process the conversation history and any task research files to fully understand the business goal.

If the task references PDF documents (technical specs, API docs, architecture docs, compliance requirements), extract and review their content.

**Step 2: Analyse the current codebase**
Perform a codebase analysis to understand the current system in the context of the task. Use the `codebase-analysis` skill. Make sure to understand project and domain best practices.

**Step 3: Ask questions about ambiguous parts**
After getting a full picture, ask any remaining questions. Don't continue until you get all answers.

**Step 4: Design a solution**
Based on your findings, design the solution architecture.

Follow best security and software design patterns. The goal is a solution that is not over-engineered and easy to comprehend, while being scalable, secure, and maintainable.

Example patterns to consider (not limited to):

- Don't Repeat Yourself
- Keep It Simple Stupid
- Domain Driven Design
- Test Driven Design
- Modular / Hexagonal / Layered Architecture
- Queue / Messaging systems
- Single Responsibility
- CQRS

UI/UX patterns to follow:

- Atomic Design
- Accessibility patterns (WCAG)

Security best practices:

- OWASP TOP10

The design must meet quality assurance criteria — fully tested using a combination of e2e, unit, and integration tests.

Don't duplicate any work. Use the `implementation-gap-analysis` skill to verify what was already implemented and what should be added. Include the result in the final plan.

Divide the plan into small phases. Each phase must be runnable on its own and have all quality gates ready from the start. Each phase must have a task list with space to mark finished tasks.

The plan must include a code review phase at the end, fully done by the `code-reviewer` agent.

Don't provide deployment plans, code-pushing instructions, or repository code review instructions.

**Step 5: Create an implementation plan document**

Save the plan following the `./plan.example.md` template.

Don't add or remove any sections from the template. Follow the structure and naming conventions strictly.

## Connected Skills

- `codebase-analysis` - for analyzing existing architecture, components, and patterns
- `implementation-gap-analysis` - for verifying what was already implemented and what should be added
- `technical-context-discovery` - for establishing project conventions and existing patterns before designing
