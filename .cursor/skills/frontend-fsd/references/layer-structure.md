# Layer Structure Reference

Detailed folder structures, code examples, and naming conventions for each FSD layer.

---

## App Layer

App-wide initialization. Organized by segments only — no slices.

```text
app/
  providers/       ← Redux, React Query, Theme providers
  styles/          ← Global CSS, reset, theme variables
  router.tsx       ← Route configuration
  index.tsx        ← Entry point
```

```typescript
// app/router.tsx
import { HomePage } from '@/pages/home';
import { ProfilePage } from '@/pages/profile';

export const router = createBrowserRouter([
  { path: '/', element: <HomePage /> },
  { path: '/profile/:id', element: <ProfilePage /> },
]);
```

**Belongs in app:** Global providers, routing setup, global styles, error boundaries, analytics initialization.  
**Does not belong:** Feature-specific code, business logic, page-level UI.

---

## Pages Layer

Route-level composition. In v2.1, pages **own substantial logic** — they are not thin wrappers. Most code lives here in early project stages.

```text
pages/
  home/
    ui/
      HomePage.tsx
      HeroSection.tsx
      FeaturesGrid.tsx
    model/
      home-data.ts          ← Page-specific state + logic
    api/
      fetch-home-data.ts    ← Page-specific API calls
    index.ts
  profile/
    ui/
      ProfilePage.tsx
      ProfileForm.tsx
    model/
      profile.ts
    api/
      update-profile.ts
    index.ts
```

**Belongs in pages:** Page-specific UI, forms, validation, data fetching, state management, business logic, API integrations.  
**Does not belong:** Code genuinely reused in 2+ pages (extract only when team agrees).

**Typical page composition:**

```typescript
// pages/product-detail/ui/ProductDetailPage.tsx
import { Header } from '@/widgets/header';
import { AddToCart } from '@/features/add-to-cart';
import { Product } from '@/entities/product';

export const ProductDetailPage = ({ productId }) => {
  const product = useProductDetail(productId); // local hook
  return (
    <>
      <Header />
      <Product.Card data={product} />
      <AddToCart productId={productId} />
      <RelatedProducts products={product.related} /> {/* local component */}
    </>
  );
};
```

---

## Widgets Layer

Composite UI blocks reused across multiple pages. Add this layer **only when UI blocks appear in 2+ pages**.

```text
widgets/
  header/
    ui/
      Header.tsx
      Navigation.tsx
      UserMenu.tsx
    model/
      header.ts
    api/
      fetch-notifications.ts
    index.ts
  sidebar/
    ui/Sidebar.tsx
    model/sidebar.ts
    index.ts
```

**Belongs in widgets:** Nav bars, sidebars, dashboards, footers, complex card layouts combining multiple entities/features.  
**Does not belong:** Simple UI primitives (→ `shared/ui/`), single-use page sections (→ keep in page).

---

## Features Layer

Reusable user interactions. **Create only when used in 2+ places.**

```text
features/
  auth/
    ui/LoginForm.tsx
    model/auth.ts
    api/login.ts
    index.ts
  add-to-cart/
    ui/AddToCartButton.tsx
    model/cart.ts
    index.ts
```

Features consume entities and are composed in higher layers:

```typescript
// widgets/post-card/ui/PostCard.tsx
import { UserAvatar } from '@/entities/user';
import { LikeButton } from '@/features/like-post';
import { CommentButton } from '@/features/comment-create';

export const PostCard = ({ post }) => (
  <article>
    <UserAvatar userId={post.authorId} />
    <LikeButton postId={post.id} />
    <CommentButton postId={post.id} />
  </article>
);
```

---

## Entities Layer

Reusable business domain models. **Start without this layer. Create only when used in 2+ places.**

```text
// Minimal entity — model only (most common)
entities/user/
  model/user.ts       ← Types + domain logic
  index.ts

// Entity with UI (use with caution)
// NOTE: Entity UI should only be imported from higher layers — never from other entities
entities/product/
  model/product.ts
  ui/ProductCard.tsx
  index.ts
```

---

## Shared Layer

Infrastructure with no business logic. Segments only (no slices). Segments may import each other.

```text
shared/
  ui/       ← UI kit: Button, Input, Modal, Card
  lib/      ← Utilities: formatDate, debounce, classnames
  api/      ← API client, route constants, CRUD helpers, base types
  auth/     ← Auth tokens, login utilities, session management
  config/   ← Environment variables, app settings
  assets/   ← Images, fonts, icons
```

```typescript
// shared/ui/Button/Button.tsx
export const Button = ({ children, onClick, variant = 'primary' }) => (
  <button className={`btn btn-${variant}`} onClick={onClick}>{children}</button>
);

// shared/ui/Button/index.ts — per-component index for tree-shaking
export { Button } from './Button';
export type { ButtonProps } from './Button';
```

Shared **may** contain application-aware code (route constants, API endpoints, branding). It must **never** contain business logic, feature-specific, or entity-specific code.

---

## Segment Naming Conventions

Always use domain-based names:

```text
// BAD — Technical-role naming
model/types.ts     ← Which types?
model/selectors.ts

// GOOD — Domain-based naming
model/user.ts      ← User types + logic + store
model/order.ts     ← Order types + logic + store
api/fetch-profile.ts
```

If a segment has only one domain concern, the filename may match the slice name: `features/auth/model/auth.ts`.

---

## Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/app/*": ["src/app/*"],
      "@/pages/*": ["src/pages/*"],
      "@/widgets/*": ["src/widgets/*"],
      "@/features/*": ["src/features/*"],
      "@/entities/*": ["src/entities/*"],
      "@/shared/*": ["src/shared/*"]
    }
  }
}
```
