# Codebase Analysis Report — <repository-name>

## Overview

| Field | Value |
|---|---|
| Repository | <repository-name> |
| Repository Type | Monorepo / Single System |
| Date | <date> |
| Analyzed By | <agent-or-user> |

## Repository Structure

### Type & Layout

<description-of-repository-type-and-high-level-layout>

### Key Directories

| Directory | Purpose | Layer/App |
|---|---|---|
| `<path>` | <purpose> | <layer-or-app-name> |

### Where Things Live

| Concern | Location |
|---|---|
| Dependencies | `<path>` |
| Scripts | `<path>` |
| Unit Tests | `<path>` |
| Integration Tests | `<path>` |
| E2E Tests | `<path>` |
| Infrastructure | `<path>` |

## Dependencies Analysis

### Core Dependencies

| Dependency | Version | Purpose | Up-to-date? |
|---|---|---|---|
| <dependency> | <version> | <purpose> | ✅ / ⚠️ Outdated / ❌ Critical |

### Outdated / Risky Dependencies

| Dependency | Current Version | Latest Version | Risk Level | Notes |
|---|---|---|---|---|
| <dependency> | <current> | <latest> | Low/Medium/High | <notes> |

## Available Scripts

### Application Scripts

| Script | Command | Description |
|---|---|---|
| <name> | `<command>` | <description> |

### Quality Assurance Scripts

| Script | Command | Description |
|---|---|---|
| <name> | `<command>` | <description> |

### Other Scripts

| Script | Command | Description |
|---|---|---|
| <name> | `<command>` | <description> |

## High-Level Architecture

### Architecture Diagram

<mermaid-or-text-diagram-of-system-architecture>

### Communication Patterns

| Pattern | Used Between | Protocol/Library |
|---|---|---|
| <pattern> | <components> | <protocol> |

## Backend Analysis

| Aspect | Details |
|---|---|
| Technology | <technology> |
| Framework | <framework> |
| Architecture Pattern | MVC / CQRS / DDD / Hexagonal / Layered / Other |
| Code Organization | Module-based / Responsibility-based |
| Database | <database> |
| ORM / Query Pattern | <orm-or-pattern> |
| Authentication | <auth-approach> |
| Authorization | <authz-approach> |
| Deployment | <deployment-approach> |

### Backend Modules/Services

| Module/Service | Path | Responsibility |
|---|---|---|
| <module> | `<path>` | <description> |

## Frontend Analysis

| Aspect | Details |
|---|---|
| Framework | <framework> |
| Communication Pattern | REST / GraphQL / WebSocket / Other |
| UI Library | <ui-library> |
| State Management | <state-management> |
| Authentication Library | <auth-library> |
| Query Library | <query-library> |
| Architecture Pattern | Atomic Design / Feature-based / Other |
| Deployment | <deployment-approach> |

### Frontend Apps/Packages

| App/Package | Path | Responsibility |
|---|---|---|
| <app> | `<path>` | <description> |

## Infrastructure Analysis

| Aspect | Details |
|---|---|
| IaC Tool | <terraform/pulumi/cloudformation/manual> |
| Hosting | Cloud / On-premise / Hybrid |
| Cloud Provider | <provider> |
| Containerized | Yes / No |
| Container Orchestration | <k8s/ecs/other> |
| Scalability | <assessment> |

## Third-Party Integrations

| Integration | Purpose | Coupling Level | Replaceability |
|---|---|---|---|
| <integration> | <purpose> | Tight / Moderate / Loose | Easy / Medium / Hard |

## Testing Approach

| Test Type | Present | Location | Framework | Coverage |
|---|---|---|---|---|
| Unit | ✅ / ❌ | `<path>` | <framework> | <coverage-%> |
| Integration | ✅ / ❌ | `<path>` | <framework> | <coverage-%> |
| E2E | ✅ / ❌ | `<path>` | <framework> | <coverage-%> |
| Contract | ✅ / ❌ | `<path>` | <framework> | <coverage-%> |

### Testing Strategy Assessment

<description-of-testing-strategy-and-gaps>

## Security Assessment

| Aspect | Implementation | Rating |
|---|---|---|
| Authentication | <description> | 🟢 Good / 🟡 Adequate / 🔴 Poor |
| Authorization | <description> | 🟢 Good / 🟡 Adequate / 🔴 Poor |
| Data Validation | <description> | 🟢 Good / 🟡 Adequate / 🔴 Poor |
| Secrets Management | <description> | 🟢 Good / 🟡 Adequate / 🔴 Poor |
| Endpoint Security | <description> | 🟢 Good / 🟡 Adequate / 🔴 Poor |
| Database Security | <description> | 🟢 Good / 🟡 Adequate / 🔴 Poor |

### Publicly Accessible Endpoints

| Endpoint | Method | Purpose | Risk |
|---|---|---|---|
| <endpoint> | <method> | <purpose> | Low/Medium/High |

## Dead Code & Duplications

### Dead Code Found

| # | Type | Location | Description |
|---|---|---|---|
| 1 | Unused Import / Function / Component / File | `<file-path>` | <description> |

### Duplications Found

| # | Type | Locations | Description | Recommendation |
|---|---|---|---|---|
| 1 | Duplicated Function / Component / Logic | `<file-path-1>`, `<file-path-2>` | <description> | <extract-to/merge-into> |

### Similar Components (Merge Candidates)

| # | Components | Locations | Similarity | Recommendation |
|---|---|---|---|---|
| 1 | <component-a>, <component-b> | `<path-a>`, `<path-b>` | <what-is-similar> | <how-to-merge> |

## Proposed Improvements

### 🔴 Critical

| # | Area | Description | Impact |
|---|---|---|---|
| 1 | <area> | <description> | <impact> |

### 🟡 Should Implement

| # | Area | Description | Impact |
|---|---|---|---|
| 1 | <area> | <description> | <impact> |

### 🟢 Nice to Have

| # | Area | Description | Impact |
|---|---|---|---|
| 1 | <area> | <description> | <impact> |

## Summary

| Category | Count / Status |
|---|---|
| Total Dependencies | <count> |
| Outdated Dependencies | <count> |
| Dead Code Items | <count> |
| Duplications Found | <count> |
| Critical Improvements | <count> |
| Non-Critical Improvements | <count> |
| Nice-to-Have Improvements | <count> |
| Overall Health Rating | 🟢 Good / 🟡 Adequate / 🔴 Needs Attention |
