---
name: node
description: Use when building Node.js 22+ backends with TypeScript — native type stripping, async patterns, streams, error handling, graceful shutdown, caching, logging, profiling, testing, and environment config. Triggers on Node.js TypeScript projects, --experimental-strip-types, .ts files without a build step, ETL/stream pipelines, flaky node:test suites, stuck/hanging processes, or any Node.js backend task.
---

# Node.js Backend

## Overview

Domain-specific patterns for building robust, performant Node.js 22+ applications with TypeScript. Prefer **type stripping** over build tools (ts-node, tsx) — run `.ts` files directly without compilation.

## TypeScript with Type Stripping

Node.js 22.6+ strips types at runtime. No build step required.

Requirements:

- Use `import type` for type-only imports
- Use `const` objects instead of enums
- Avoid namespaces and parameter properties
- Use `.ts` extensions in imports

```ts
// greet.ts
import type { IncomingMessage } from 'node:http';

const greet = (name: string): string => `Hello, ${name}!`;
console.log(greet('world'));
```

```bash
node greet.ts                          # Node 22.19+, 23.6+, 24+
node --experimental-strip-types app.ts # Node 22.6–22.18
```

See [rules/typescript.md](rules/typescript.md) for full config and `tsconfig.json` setup.

## Common Workflows

**Graceful shutdown**: Register SIGTERM/SIGINT → stop accepting work → drain in-flight requests → close DB/cache connections → exit with code. See [rules/graceful-shutdown.md](rules/graceful-shutdown.md).

**Error handling**: Shared error base class → classify (operational vs programmer) → `process.on('unhandledRejection')` → propagate typed errors → log with context. See [rules/error-handling.md](rules/error-handling.md).

**Diagnosing flaky tests**: Isolate with `--test-only` → check shared state/timer deps → inspect async teardown order → fix root cause. See [rules/flaky-tests.md](rules/flaky-tests.md).

**Stuck processes / hanging `node --test`**: Isolate file → run with explicit timeout/reporter → inspect open handles via `why-is-node-running` or `SIGUSR1` → patch deterministic teardown. See [rules/stuck-processes-and-tests.md](rules/stuck-processes-and-tests.md).

**Profiling a slow path**: Reproduce under load → `--cpu-prof` → find hot functions → check stream backpressure → validate with benchmark. See [rules/profiling.md](rules/profiling.md) and [rules/performance.md](rules/performance.md).

## High-Priority: Streams + Caching Checklist

When the task mentions **CSV**, **ETL**, **ingestion pipelines**, **large file processing**, **backpressure**, **repeated lookups**, or **deduplicating concurrent async calls**:

1. Use `await pipeline(...)` from `node:stream/promises` (prefer over chained `.pipe()`).
2. Include at least one `async function*` transform when data is transformed in-stream.
3. Choose a cache strategy for repeated work:
   - `lru-cache` — bounded in-memory reuse, single process.
   - `async-cache-dedupe` — async request deduplication / stale-while-revalidate.
4. Show where backpressure is handled (implicitly via `pipeline()` or explicitly via `drain`).

Integrated ETL pattern:

- `createReadStream(input)` → `async function*` parser/transform → optional cached enrichment → `await pipeline(...)` to writable destination.

## Rule Files (Quick Reference)

| Topic                                   | File                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------ |
| TypeScript / type stripping             | [rules/typescript.md](rules/typescript.md)                               |
| Async/await & Promise patterns          | [rules/async-patterns.md](rules/async-patterns.md)                       |
| Streams & backpressure                  | [rules/streams.md](rules/streams.md)                                     |
| Error handling                          | [rules/error-handling.md](rules/error-handling.md)                       |
| Graceful shutdown                       | [rules/graceful-shutdown.md](rules/graceful-shutdown.md)                 |
| Testing (node:test)                     | [rules/testing.md](rules/testing.md)                                     |
| Flaky tests                             | [rules/flaky-tests.md](rules/flaky-tests.md)                             |
| Stuck processes & tests                 | [rules/stuck-processes-and-tests.md](rules/stuck-processes-and-tests.md) |
| Performance                             | [rules/performance.md](rules/performance.md)                             |
| Caching (lru-cache, async-cache-dedupe) | [rules/caching.md](rules/caching.md)                                     |
| Profiling & benchmarking                | [rules/profiling.md](rules/profiling.md)                                 |
| Logging & debugging                     | [rules/logging.md](rules/logging.md)                                     |
| Environment config & secrets            | [rules/environment.md](rules/environment.md)                             |
| ES Modules & CommonJS                   | [rules/modules.md](rules/modules.md)                                     |
| node_modules exploration                | [rules/node-modules-exploration.md](rules/node-modules-exploration.md)   |

## Common Mistakes

| Mistake                                     | Fix                                          |
| ------------------------------------------- | -------------------------------------------- |
| Using `ts-node` / `tsx`                     | Use native type stripping (`node app.ts`)    |
| Enums in type-stripped code                 | Replace with `const` object + `keyof typeof` |
| `.js` extensions in TS imports              | Use `.ts` extensions                         |
| Chained `.pipe()` without error handling    | Use `await pipeline(...)`                    |
| Ignoring `process.on('unhandledRejection')` | Always register async boundary handlers      |
| `process.exit()` without draining           | Implement graceful shutdown sequence         |
| Storing secrets in env without validation   | Validate with `zod` at startup               |
