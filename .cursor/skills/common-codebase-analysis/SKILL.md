---
name: codebase-analysis
description: Use when onboarding to an unfamiliar codebase, auditing repository health, investigating architecture and dependencies, or looking for dead code and duplications. Also use when asked to "analyse the codebase" or produce a codebase health report.
---

# Codebase Analysis

Structured 12-step audit of a repository covering architecture, dependencies, code style, testing, security, and improvement opportunities.

## When to Use

- **Onboarding** — understand a new or inherited codebase
- **Health audit** — assess dependency freshness, dead code, security posture
- **Pre-migration** — inventory before major refactors or upgrades
- **Due diligence** — evaluate code quality of an external project

**Not for requirement-specific analysis.** To classify code as reuse/modify/create against a set of requirements, use the `implementation-gap-analysis` skill with the `codebase-analyzer` subagent instead.

## Using the codebase-analyzer Subagent

For targeted deep dives during analysis (e.g. "find all auth-related code", "inventory the UI component library"), dispatch a `codebase-analyzer` subagent in readonly mode:

```
Task(
  subagent_type="codebase-analyzer",
  prompt="Search for all authentication and authorization code.
  Classify into: existing patterns, libraries used, security concerns.
  Search scope: [target directories]",
  readonly=true
)
```

Run multiple subagents in parallel for independent areas (backend vs frontend vs infra) to speed up large codebases.

## Analysis Process

Track progress with this checklist:

```
Analysis progress:
- [ ] Step 1: Repository structure
- [ ] Step 2: Dependencies
- [ ] Step 3: Available scripts
- [ ] Step 4: High-level architecture
- [ ] Step 5: Backend code
- [ ] Step 6: Frontend code
- [ ] Step 7: Infrastructure code
- [ ] Step 8: Third-party integrations
- [ ] Step 9: Testing approach
- [ ] Step 10: Security assessment
- [ ] Step 11: Potential improvements
- [ ] Step 12: Dead code & duplications
```

### Step 1: Repository structure

Identify repository type (monorepo vs single system). Map key directories: apps, packages, shared code, infra, tests. Locate dependency manifests and script definitions.

### Step 2: Dependencies

Audit all dependency files. Note frameworks, auth libraries, ORMs. Flag outdated or vulnerable packages with severity.

### Step 3: Available scripts

Catalog app-running scripts, QA scripts (test, lint, format, typecheck), build/deploy scripts.

### Step 4: High-level architecture

Map services, their communication patterns (REST, GraphQL, WebSocket, message queues), and external integrations. Produce an architecture diagram (Mermaid or text).

### Step 5: Backend code

Identify: technology/framework, architecture pattern (MVC, CQRS, DDD, Hexagonal, Layered), code organization (module-based vs responsibility-based), database + ORM/query pattern, auth mechanism, deployment approach.

### Step 6: Frontend code

Identify: framework, communication pattern, UI library, state management, query library, architecture pattern (atomic design, feature-based, FSD), deployment approach.

### Step 7: Infrastructure code

Identify: IaC tool or manual setup, hosting (cloud/on-prem), containerization, orchestration, scalability assessment.

### Step 8: Third-party integrations

List all third-party services. Assess coupling level (tight/moderate/loose) and replaceability (easy/medium/hard).

### Step 9: Testing approach

Identify test types present (unit, integration, e2e, contract). Assess strategy, coverage, and gaps.

### Step 10: Security assessment

Evaluate authentication, authorization, data validation, secrets management, endpoint security, database security. Identify publicly accessible endpoints.

### Step 11: Potential improvements

Categorize findings into:

- **Critical** — security vulnerabilities, breaking issues
- **Should implement** — technical debt, outdated patterns
- **Nice to have** — optimizations, DX improvements

### Step 12: Dead code & duplications

Search for: unused imports/functions/components/files, duplicated logic that can be extracted, similar components that can be merged, code only referenced in its own tests.

## Large Codebases

For large repos, create the report document early and update it incrementally per step. Use parallel `codebase-analyzer` subagents for independent areas.

## Report

- **Standalone use** — save analysis as `docs/codebase-analysis.md` following the [report template](codebase-analysis.example.md)
- **Called by another workflow** — embed findings into the caller's output format instead of creating a separate report, unless the user explicitly requests one

For monorepos, duplicate per-layer sections (Backend, Frontend, etc.) per app/package when they differ significantly. Label each clearly.
