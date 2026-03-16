---
name: tailwind
description: Use when configuring Tailwind CSS v4, setting up themes, working with responsive design, container queries, dark mode, CVA variants, arbitrary values, or the cn() utility. Triggers on @theme, tailwind.config, cva, container queries, or any Tailwind reference task.
---

# Tailwind CSS v4

**Official Docs**: [Tailwind CSS v4](https://tailwindcss.com/docs)

## CSS-First Configuration

Configure via `@theme` in CSS. Do not use `tailwind.config.js`.

```css
@import 'tailwindcss';

@theme {
  --color-primary: oklch(0.5 0.2 250);
  --color-secondary: oklch(0.6 0.15 300);
  --font-sans: 'Inter', system-ui, sans-serif;
}
```

Override an entire namespace with `initial` before redefining:

```css
@theme {
  --color-*: initial;
  --color-primary: oklch(0.5 0.2 250);
  --color-surface: oklch(0.98 0 0);
}
```

Use `@theme inline` when referencing other CSS variables to avoid resolution issues:

```css
@theme inline {
  --font-sans: var(--font-inter);
}
```

## cn() Utility

Use `cn()` only when needed:

```tsx
// BAD - no conditions, cn() is unnecessary
<div className={cn('flex items-center gap-2')} />

// GOOD - static classes
<div className="flex items-center gap-2" />

// GOOD - conditional classes
<div className={cn('rounded-lg border', isActive && 'border-blue-500')} />

// GOOD - merging with external className prop
<button className={cn('px-4 py-2 bg-blue-500', className)} />
```

## Component Variants with CVA

Use `class-variance-authority` for reusable variant-based component styles.

```ts
import { cva } from 'class-variance-authority';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-lg font-medium transition-colors',
  {
    variants: {
      variant: {
        primary: 'bg-primary text-white hover:bg-primary/90',
        outline:
          'border-2 border-primary text-primary hover:bg-primary hover:text-white',
        ghost: 'hover:bg-gray-100 text-gray-700',
      },
      size: {
        sm: 'px-3 py-1.5 text-sm',
        md: 'px-4 py-2 text-base',
        lg: 'px-6 py-3 text-lg',
      },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  },
);
```

## Dynamic Values

Use the `style` prop for truly dynamic values that cannot be known at build time.

```tsx
// GOOD - runtime-dependent value via style, static utilities via className
<div className="rounded-lg bg-primary" style={{ width: `${percentage}%` }} />

// GOOD - CSS variable bridge for dynamic theming
<button
  style={{ '--bg': buttonColor, '--bg-hover': buttonColorHover }}
  className="bg-(--bg) hover:bg-(--bg-hover) rounded-md px-4 py-2"
/>
```

When a library (e.g. Recharts) requires inline style values, use `var()` constants there only:

```ts
const CHART_COLORS = { primary: 'var(--color-primary)', grid: 'var(--color-border)' } as const;
<XAxis tick={{ fill: CHART_COLORS.primary }} />
```

## Responsive Design

Mobile-first. Unprefixed utilities apply to all screen sizes.

```tsx
<div className="w-full md:w-1/2 lg:w-1/3" />
<nav className="hidden md:flex" />
<h1 className="text-2xl lg:text-4xl" />
```

### Container Queries

```tsx
<div className="@container">
  <div className="grid grid-cols-1 @sm:grid-cols-2 @lg:grid-cols-3 gap-4">
    {items.map(renderItem)}
  </div>
</div>
```

## Dark Mode

```tsx
<div className="bg-white text-gray-900 dark:bg-gray-900 dark:text-white" />
```

## State Variants

```tsx
<button className={cn(
  'bg-blue-500',
  'hover:bg-blue-600',
  'focus:ring-2 focus:ring-blue-400',
  'active:bg-blue-700',
  'disabled:opacity-50 disabled:cursor-not-allowed'
)} />

// Group hover: parent has `group`, child uses `group-hover:`
<a className="group">
  <span className="group-hover:underline">Read more</span>
</a>
```

## Arbitrary Values (Escape Hatch)

Use square brackets for one-off values not in the design system. Do not use for colors — extend the theme instead.

```tsx
// GOOD - one-off layout value
<div className="w-[327px] grid-cols-[1fr_2fr_1fr]" />
<div className="max-h-[calc(100dvh-var(--spacing-6))]" />

// BAD - color should be in theme
<div className="bg-[#1e293b]" />
```

## Source Configuration

```css
@import 'tailwindcss';
@source '../node_modules/@acmecorp/ui-lib';
@source not '../src/legacy';
```

Use `@source inline()` to safelist classes not present in source files:

```css
@source inline('{hover:,focus:,}underline');
```

## Managing Duplication

Do not extract utility sets into custom CSS classes. Extract components instead.

```tsx
// BAD
// .avatar { @apply inline-block h-12 w-12 rounded-full ring-2 ring-white; }

// GOOD - component
function Avatar({ src, alt }: { readonly src: string; readonly alt: string }) {
  return (
    <img
      className="inline-block size-12 rounded-full ring-2 ring-white"
      src={src}
      alt={alt}
    />
  );
}
```
