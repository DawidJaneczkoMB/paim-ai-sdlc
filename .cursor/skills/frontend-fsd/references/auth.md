# Authentication Patterns in FSD

## Overview

Authentication involves three concerns:

1. Collecting credentials from the user
2. Sending them to the backend
3. Storing the resulting token for subsequent requests

## Collecting credentials

### Dedicated login page (most common)

Create a slice on the Pages layer. Login and register forms are similar enough to share one slice:

```
pages/
└── login/
    ├── ui/
    │   ├── LoginPage.tsx
    │   └── RegisterPage.tsx
    ├── model/
    │   └── registration-schema.ts   # Client-side validation (e.g., Zod schema)
    └── index.ts
```

### Login dialog (reused across pages)

If the login UI can appear on any page, make it a widget:

```
widgets/
└── login-dialog/
    ├── ui/
    │   └── LoginDialog.tsx
    └── index.ts
```

## Client-side validation

Put validation schemas in the `model/` segment of the login page (or widget). Keep validation logic out of the UI component itself — import the schema into the component.

## Sending credentials to the backend

Create a request function that calls your login endpoint. Place it in:

- `shared/api/endpoints/auth.ts` — if you centralize all API calls in shared (recommended)
- `pages/login/api/` — if you prefer locality

For two-factor authentication (2FA): the login page can contain the 2FA step in the same slice (keep related pages together). Create a separate request function for the OTP endpoint alongside the login function.

## Storing the token

### Option A: In shared (recommended for most apps)

Store the token in `shared/auth` or within the API client in `shared/api`. This makes the token freely available to all request functions without any cross-layer complexity.

```
shared/
├── auth/
│   ├── use-auth.ts        # Token store (reactive) + current user
│   └── index.ts
└── api/
    └── client.ts          # API client reads token from shared/auth
```

Automatic token refresh can be implemented as middleware in the API client: intercept 401 responses, call the refresh endpoint, update the token store, retry the original request.

### Option B: In entities (for apps with complex user models)

Store the token in a `model/` segment of a `user` entity. The challenge: the API client in `shared/api` cannot import from `entities/` (it would violate the import rule). Solve this by one of:

1. **Pass the token to every request manually** — simple but tedious
2. **Expose via context or a global store key** — context set up in `app/`, key defined in `shared/api` so the client can read it
3. **Inject via subscription** — subscribe to token changes in the entity's store and push updates into the API client

### Option C: In pages/widgets — avoid this

Never store app-wide state (tokens, sessions) in pages or widgets. These layers are not globally accessible enough for auth state.

## Logout

Logout rarely needs its own page. The logout button is typically in a header widget. Pattern:

```
widgets/
└── header/
    ├── ui/
    │   └── Header.tsx
    ├── model/
    │   └── use-logout.ts      # Combines: call logout endpoint + clear token store
    └── index.ts
```

Keep the logout request function near the login request function (both in `shared/api` if that's where login lives).

## Automatic logout on token expiry

When a refresh request fails or a logout request fails, always clear the token store as a failsafe. This logic lives in:

- `shared/api` middleware (if token is in Shared)
- `entities/user/model/` (if token is in Entities)

If `shared/api` is getting too large mixing API and auth logic, split into `shared/api` (requests) and `shared/auth` (token management).
