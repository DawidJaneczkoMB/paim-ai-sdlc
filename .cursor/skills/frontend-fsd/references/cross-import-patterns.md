# Cross-Import Resolution Patterns

Strategies and code examples for resolving cross-imports between slices on the same layer. Always try strategies **in order** — earlier strategies are preferred.

---

## The Problem

Cross-imports occur when two slices on the same layer reference each other. This violates FSD's import rule. Resolve using the strategies below.

---

## Strategy 1: Merge Slices

If two slices always change together, they likely represent one concept.

**Indicators:** Changes to one almost always require changes to the other; developers confuse which slice owns what.

```text
// Before: two features always changing together
features/send-message/
features/message-list/

// After: one cohesive feature
features/messaging/
  ui/
    MessageInput.tsx
    MessageList.tsx
  model/
    message-draft.ts
    messages.ts
  index.ts
```

---

## Strategy 2: Extract Shared Logic to Entities

When multiple features share the same domain logic, move that logic to `entities/`. Keep UI and interaction-specific code in the feature.

```text
// Before: two features duplicate order logic
features/order-create/model/order.ts    ← duplicated
features/order-history/model/order.ts  ← duplicated

// After: shared domain in entities, UI stays in features
entities/order/model/order.ts          ← shared types + domain logic
features/order-create/model/order-form.ts   ← form-specific logic
features/order-history/model/order-display.ts
```

Only extract genuinely shared domain logic (types, validation, business calculations). Feature-specific state and API calls stay in the feature.

---

## Strategy 3: Compose in a Higher Layer (Inversion of Control)

The parent layer (pages or app) imports both slices and connects them. The slices never reference each other.

**React — Render Props:**

```typescript
// Problem: features/comment-list wants user avatars from features/user-profile
// Solution: pages/post composes both

// pages/post/ui/PostPage.tsx
import { CommentList } from '@/features/comments';
import { UserAvatar } from '@/entities/user';

const PostPage = ({ post }) => (
  <CommentList
    comments={post.comments}
    renderAuthor={(userId) => <UserAvatar userId={userId} />}
  />
);
```

**Any framework — Dependency Injection:**

```typescript
// features/notifications/model/notifications.ts
interface NotificationDeps {
  getUserName: (userId: string) => string;
}

export const createNotificationService = (deps: NotificationDeps) => ({
  formatNotification: (n) => `${deps.getUserName(n.userId)}: ${n.message}`,
});

// pages/dashboard/model/setup.ts — wire dependencies here
import { createNotificationService } from '@/features/notifications';
import { getUserName } from '@/entities/user';

export const notificationService = createNotificationService({ getUserName });
```

Use when the slices are genuinely independent and the connection is a composition concern.

---

## Strategy 4: @x Notation (Last Resort — Entities Only)

When none of the above apply, use `@x` to create explicit, controlled cross-imports **between entities only**.

```text
entities/
  user/
    @x/
      order.ts          ← Exposed specifically for the order entity
    model/user.ts
    index.ts
  order/
    model/order-summary.ts
    index.ts
```

```typescript
// entities/user/@x/order.ts — exposes only what order needs
export { getUserDisplayName } from '../model/user';

// entities/order/model/order-summary.ts
import { getUserDisplayName } from '@/entities/user/@x/order';
```

**@x Rules:**

1. Document why @x is needed and why other strategies don't apply
2. Review periodically — requirements change
3. Minimize the exported surface area
4. Only use between entities — never for features/widgets/pages
5. Regular cross-imports (without @x) remain forbidden

---

## Excessive Entities — Root Cause of Most @x Problems

Most @x problems come from extracting entities too early. Signs:

- Multiple entities with @x on each other
- Entities used in only one page/feature
- Very thin entity slices (just a type + re-export)
- Frequent multi-entity updates for a single feature change

**Resolution:**

1. Audit entity usage — find single-use entities
2. Move them back to their consuming page or feature
3. Merge closely related entities (e.g., merge `order` + `order-item`)
4. Keep plain API response types in `shared/api/` — an entity requires reusable domain _logic_, not just types

---

## Decision Flowchart

```
Two slices on the same layer need to share code
  │
  ├─ Do they always change together?
  │   └─ YES → Strategy 1: Merge slices
  │
  ├─ Is the shared part domain logic (types, validation, business rules)?
  │   └─ YES → Strategy 2: Extract to entities
  │
  ├─ Is the connection a composition concern (UI assembly, data wiring)?
  │   └─ YES → Strategy 3: Compose in higher layer (IoC)
  │
  └─ None of the above, and both are entities?
      └─ YES → Strategy 4: @x notation
      └─ NO  → Reconsider slice boundaries
```
