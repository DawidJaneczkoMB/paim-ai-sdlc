# Public API Reference

## What is the Public API?

Every slice exposes a contract — `index.ts` at the slice root. Code outside the slice may only import through this file. The file controls what is and isn't part of the public surface.

## Rules

1. **All external imports go through index.ts** — never import deep paths from another slice
2. **Export explicitly** — never `export * from`
3. **Export only what consumers actually need** — keep internals private
4. **Breaking changes to behavior = change in the public API** — if something's signature changes, update index.ts intentionally

## What to export from index.ts

| Type                               | Export?   | Reason                                             |
| ---------------------------------- | --------- | -------------------------------------------------- |
| Top-level UI component             | Yes       | Consumers render this                              |
| Public TypeScript types/interfaces | Yes       | Consumers need types                               |
| Public hooks meant for use outside | Sometimes | Only if the hook is the intended integration point |
| Internal model hooks               | No        | Internal implementation detail                     |
| Raw API request functions          | No        | Consumers should use the query/mutation hooks      |
| Internal helper functions          | No        | Not for external consumption                       |

## Examples

```ts
// GOOD — features/users-filter/index.ts — clean public API
export { UsersFilter } from './ui/UsersFilter';
export type { UsersFilterProps } from './ui/UsersFilter';

// BAD — wildcard exports hide the interface and expose internals
export * from './ui/UsersFilter';
export * from './model/use-users-selector';
export * from './api/use-some-data';
```

## Consuming a slice

```ts
// GOOD — Always import via the public API (the slice folder)
import { UsersFilter } from '@/features/users-filter';

// BAD — Never import internal paths
import { UsersFilter } from '@/features/users-filter/ui/UsersFilter';
```

## Why this matters for refactoring

When you rename `users-filter.tsx` → `clients-filter.tsx`:

- Without index.ts: you must update every import site in the codebase
- With index.ts: update only the re-export in `features/users-filter/index.ts`

In large projects this is critical.

## Cross-imports between entities (@x notation)

Entities cannot import from sibling slices. When entity B genuinely needs a type from entity A, use the `@x` notation:

```
entities/
├── artist/
│   ├── @x/
│   │   └── song.ts        ← special public API just for the song entity
│   ├── model/
│   │   └── artist.ts
│   └── index.ts
└── song/
    ├── model/
    │   └── song.ts
    └── index.ts
```

```ts
// entities/artist/@x/song.ts — exposes only what song entity needs
export type { Song } from '../model/song';

// entities/artist/model/artist.ts — imports via @x
import type { Song } from 'entities/song/@x/artist';
```

Read `@x/B.ts` as "A crossed with B" — entity A's API surface exposed specifically for entity B.

**Only use `@x` in the Entities layer.** On other layers, re-architecture to avoid cross-imports.

## Circular import trap

Never import from the slice's own index file within the slice:

```ts
// BAD — pages/home/ui/HomePage.tsx
import { loadUserStats } from '../'; // imports pages/home/index.ts
// which imports HomePage.tsx → circular!

// GOOD — pages/home/ui/HomePage.tsx
import { loadUserStats } from '../api/load-user-stats'; // relative import
```

Rule: inside a slice, always use relative imports. Index files are for external consumers only.

## Barrel file performance in shared/

Having one massive `shared/ui/index.ts` that re-exports 50 components can:

- Break tree-shaking (bundler pulls in components you don't use)
- Slow down the dev server on large projects

Solution: one index file per component in `shared/ui/` and `shared/lib/`:

```ts
// GOOD — consumers import the specific component subfolder
import { Button } from '@/shared/ui/button';
import { TextField } from '@/shared/ui/text-field';
```
