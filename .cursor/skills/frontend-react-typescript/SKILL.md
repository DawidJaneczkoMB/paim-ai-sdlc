---
name: react-typescript
description: Use when implementing React 19 patterns — props typing, extending HTML elements, generic components, useTransition, useRef typing, context, custom hooks, compound components, or multi-step ref patterns. Triggers on React 19 component patterns, forwardRef migration, createRadioGroup, compound components, or slot props.
---

# React 19 + TypeScript — Patterns Reference

**Official Docs**: [React 19](https://react.dev/)

For always-enforced constraints (no React.FC, no forwardRef, no useFormState, on*/handle* naming), see the `frontend-react-typescript` rule.

## Props Typing

### Extending HTML Elements

Use `React.ComponentPropsWithoutRef<'element'>` to inherit native HTML attributes.

```tsx
type ButtonProps = {
  readonly variant: 'primary' | 'secondary';
} & React.ComponentPropsWithoutRef<'button'>;

function Button({ variant, className, children, ...props }: ButtonProps) {
  return (
    <button className={cn('rounded-md', className)} {...props}>
      {children}
    </button>
  );
}
```

### Generic Components

```tsx
type ListProps<TItem> = {
  readonly items: ReadonlyArray<TItem>;
  readonly renderItem: (item: TItem) => React.ReactNode;
  readonly keyExtractor: (item: TItem) => string;
};

function List<TItem>({ items, renderItem, keyExtractor }: ListProps<TItem>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}
```

## ref as Prop (React 19)

Pass `ref` as a regular prop. `forwardRef` is deprecated in React 19.

```tsx
type InputProps = {
  readonly ref?: React.Ref<HTMLInputElement>;
  readonly label: string;
} & React.ComponentPropsWithoutRef<'input'>;

function Input({ ref, label, ...props }: InputProps) {
  return (
    <div>
      <label>{label}</label>
      <input ref={ref} {...props} />
    </div>
  );
}
```

### useImperativeHandle with ref Prop

```tsx
type InputHandle = { readonly focus: () => void; readonly clear: () => void };

type FancyInputProps = {
  readonly ref?: React.Ref<InputHandle>;
  readonly label: string;
};

function FancyInput({ ref, label }: FancyInputProps) {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    clear: () => {
      if (inputRef.current) inputRef.current.value = '';
    },
  }));

  return <input ref={inputRef} aria-label={label} />;
}
```

## useTransition

In React 19, `startTransition` supports async functions.

```tsx
function DeleteButton({ postId }: { readonly postId: string }) {
  const [isPending, startTransition] = useTransition();

  function handleDelete() {
    startTransition(async () => {
      await deletePost(postId);
    });
  }

  return (
    <button onClick={handleDelete} disabled={isPending}>
      {isPending ? 'Deleting...' : 'Delete'}
    </button>
  );
}
```

## Hook Typing

### useRef

```tsx
// DOM ref - null initial, readonly .current
const inputRef = useRef<HTMLInputElement>(null);

// Mutable ref - non-null initial, mutable .current
const renderCount = useRef(0);
renderCount.current += 1;

// Timer ref
const timerRef = useRef<ReturnType<typeof setTimeout>>();
```

## Context Typing

Create a `null`-initialized context with a guarded hook. Never use `undefined!` or `as` to skip the null check.

```tsx
// BAD
const AuthContext = createContext<AuthContextValue>(undefined!);

// GOOD
const AuthContext = createContext<AuthContextValue | null>(null);

function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

## Custom Hooks

Return objects. Explicit return types for exported hooks.

```tsx
// BAD - tuple return is positional and fragile
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  return [value, () => setValue((v) => !v)];
}

// GOOD
function useToggle(initial = false) {
  const [isOpen, setIsOpen] = useState(initial);
  return {
    isOpen,
    toggle: () => setIsOpen((v) => !v),
    open: () => setIsOpen(true),
    close: () => setIsOpen(false),
  };
}
```

## Compound Components

Suit flexible-layout, mostly-static content (RadioGroup, TabBar, ButtonGroup). Do not use for fixed-layout components or primarily dynamic content.

### When NOT to Use

Use **props** when children follow a fixed layout or content is dynamic. Use **slot props** when layout order is fixed but content is flexible.

```tsx
// BAD - compound component for fixed-layout Select with dynamic data
<Select value={value} onChange={onChange}>
  {items.map((item) => (
    <Option value={item.value}>{item.label}</Option>
  ))}
</Select>;

// GOOD - props-based Select for dynamic options
function Select<TValue extends string | number>({
  value,
  onChange,
  options,
}: SelectProps<TValue>) {
  /* ... */
}

// GOOD - slot props for fixed-layout
type ModalDialogProps = {
  readonly header: React.ReactNode;
  readonly body: React.ReactNode;
  readonly footer?: React.ReactNode;
};
function ModalDialog({ header, body, footer }: ModalDialogProps) {
  return (
    <DialogRoot>
      <DialogHeader>{header}</DialogHeader>
      <DialogBody>{body}</DialogBody>
      <DialogFooter>{footer}</DialogFooter>
    </DialogRoot>
  );
}
```

### Factory Pattern for Type-Safe Compound Components

Use `createXGroup<T>()` to tie generic type parameters across parent and child. Default the generic to `never` to force consumers to provide the type.

```tsx
export function createRadioGroup<TValue extends GroupValue = never>(): {
  RadioGroup: (props: RadioGroupProps<TValue>) => React.ReactElement;
  RadioGroupItem: (props: RadioGroupItemProps<TValue>) => React.ReactElement;
} {
  return { RadioGroup, RadioGroupItem };
}

// Usage — type annotation happens once, all children inherit it
type ThemeValue = 'system' | 'light' | 'dark';
const Theme = createRadioGroup<ThemeValue>();

<Theme.RadioGroup value={value} onChange={onChange}>
  <Theme.RadioGroupItem value="system">🤖</Theme.RadioGroupItem>
  <Theme.RadioGroupItem value="light">☀️</Theme.RadioGroupItem>
  {/* TS error: '"wrong"' is not assignable to type 'ThemeValue' */}
  <Theme.RadioGroupItem value="wrong">🚨</Theme.RadioGroupItem>
</Theme.RadioGroup>;
```

## Props to State

Prefix with `initial` when copying a prop into state (communicates explicit intent).

```tsx
// BAD
function Editor({ content }: { readonly content: string }) {
  const [value, setValue] = useState(content); // won't update when prop changes
}

// GOOD
function Editor({ initialContent }: { readonly initialContent: string }) {
  const [value, setValue] = useState(initialContent);
}
```
