# FSD Folder Structure Reference

## Complete project tree

```
src/
├── app/
│   ├── routes/           # Router configuration
│   ├── store/            # Global store setup
│   ├── styles/           # Global CSS/theme
│   └── providers/        # Context providers wrapping the app
│
├── pages/
│   ├── home/
│   │   ├── ui/
│   │   │   └── HomePage.tsx
│   │   └── index.ts
│   ├── post-feed/
│   │   ├── ui/
│   │   │   └── PostFeedPage.tsx
│   │   ├── api/
│   │   │   └── get-posts.ts
│   │   └── index.ts
│   └── login/
│       ├── ui/
│       │   ├── LoginPage.tsx
│       │   └── RegisterPage.tsx
│       ├── model/
│       │   └── registration.schema.ts
│       └── index.ts
│
├── widgets/
│   ├── territory-management/
│   │   ├── ui/
│   │   │   └── TerritoryManagement.tsx
│   │   └── index.ts
│   └── login-dialog/
│       ├── ui/
│       │   └── LoginDialog.tsx
│       └── index.ts
│
├── features/
│   ├── users-filter/
│   │   ├── ui/
│   │   │   ├── UsersFilter.tsx
│   │   │   ├── UsersSelector.tsx
│   │   │   └── AssignToSelector.tsx
│   │   ├── model/
│   │   │   ├── use-users-selector.ts
│   │   │   └── use-assign-to-selector.ts
│   │   ├── api/
│   │   │   └── use-some-data.ts
│   │   └── index.ts
│   └── users-search/
│       ├── ui/
│       │   └── UsersSearch.tsx
│       ├── model/
│       │   └── use-users-search.ts
│       └── index.ts
│
├── entities/
│   ├── user/
│   │   ├── ui/
│   │   │   └── UserCard.tsx
│   │   ├── model/
│   │   │   └── user.ts      # Type definitions, validation schema
│   │   ├── @x/
│   │   │   └── post.ts      # Cross-import API for the post entity
│   │   └── index.ts
│   └── post/
│       ├── ui/
│       │   └── PostCard.tsx
│       ├── model/
│       │   └── post.ts
│       ├── api/
│       │   ├── post.queries.ts
│       │   ├── get-post.ts
│       │   └── get-posts.ts
│       └── index.ts
│
└── shared/
    ├── api/
    │   ├── client.ts         # Base API client instance
    │   ├── query-client.ts   # React Query client config
    │   └── index.ts
    ├── auth/
    │   ├── use-auth.ts       # Auth token/session store
    │   └── index.ts
    ├── ui/
    │   ├── button/
    │   │   └── index.ts      # Separate index per component!
    │   ├── text-field/
    │   │   └── index.ts
    │   └── modal/
    │       └── index.ts
    ├── lib/
    │   ├── dates/
    │   │   └── index.ts
    │   └── colors/
    │       └── index.ts
    ├── config/
    │   └── env.ts            # Environment variables
    ├── routes/
    │   └── paths.ts          # Route constants
    └── i18n/
        └── index.ts
```

## Key structural rules

### App and Shared have NO slices

Both are organized directly into segments. In `shared/`, every top-level folder is a segment (`api`, `ui`, `lib`, etc.). Do not create business-domain subfolders in shared.

### shared/ui and shared/lib: use per-item index files

Do NOT create a single `shared/ui/index.ts` that re-exports everything. This breaks tree-shaking and creates large bundles. Instead:

```
shared/ui/
├── button/
│   └── index.ts    ← consumers import from "@/shared/ui/button"
├── modal/
│   └── index.ts    ← consumers import from "@/shared/ui/modal"
```

### Slice grouping in a folder is OK, but no code sharing

If you have related slices, you may group them:

```
features/
└── post/
    ├── like-post/
    │   └── index.ts
    ├── delete-post/
    │   └── index.ts
    └── edit-post/
        └── index.ts
    # BAD — NO shared-code.ts here — that breaks slice isolation
```

### Login and Register can share a page slice

When pages are very similar (login vs register), they can live in one slice and export both components from the same `index.ts`.

## Segment naming guidelines

Use purpose-oriented names, not type-oriented names:

| Good      | Bad (type-oriented)  |
| --------- | -------------------- |
| `model/`  | `hooks/`             |
| `ui/`     | `components/`        |
| `api/`    | `services/`          |
| `lib/`    | `utils/`, `helpers/` |
| `config/` | `constants/`         |

If you create a custom segment, its name must describe _what the code is for_, not _what kind of code it is_.
