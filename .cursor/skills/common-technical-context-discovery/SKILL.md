---
name: technical-context-discovery
description: Use when starting any implementation task to establish technical context — checking project rules, codebase patterns, then external docs in priority order
---

# Technical Context Discovery

Before implementing, establish context in this priority order:

## Priority Order

### 1. Project Instructions (primary source of truth)

Search for `.cursor/rules/*.md`, `AGENTS.md`, and `*.instructions.md` in relevant directories. If found, follow strictly — instructions override general best practices.

### 2. Existing Codebase Patterns

If instructions don't cover it, analyze existing code: component structure, styling approach, state management, testing patterns, naming conventions, a11y. Use search/grep to find comparable implementations. Mirror patterns exactly — consistency > theory.

### 3. External Docs & Best Practices

Greenfield only. Use official framework docs, WCAG guidelines, industry standards. Document architectural decisions for future reference.

## Decision Rule

- **Instructions exist** → follow them strictly
- **No instructions, patterns exist** → mirror codebase patterns
- **Neither** → apply best practices, document decisions
- **Never introduce new patterns** unless explicitly asked

## Checklist

1. Search for project instructions (rules, AGENTS.md, \*.instructions.md)
2. Find comparable implementations via search/grep
3. Verify dependency versions (package.json, go.mod, etc.)
4. Document any architectural choices made
