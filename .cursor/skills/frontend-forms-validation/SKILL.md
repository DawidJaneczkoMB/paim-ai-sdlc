---
name: forms-validation
description: Use when building forms with Zod v4 and React Hook Form — schema definition, validation, Controller, useFieldArray, multi-step forms, server error mapping, or async defaults. Triggers on useForm, zodResolver, z.object, createPostSchema, or any form validation task.
---

# Forms Validation (Zod v4 + React Hook Form)

**Official Docs**: [Zod v4](https://zod.dev/llms.txt) | [React Hook Form](https://react-hook-form.com/)

## Schema Definition

Place schemas in `[name].schema.ts` files. Use namespace import.

```ts
// BAD
import { z } from 'zod';

// GOOD
import * as z from 'zod';
```

Derive TypeScript types from schemas. Never define form types manually.

```ts
const FORM_SCHEMA = z.object({
  email: z.email(),
  age: z.coerce.number<number>().positive(),
});

type FormData = z.infer<typeof FORM_SCHEMA>;
```

**Always define schemas at module level. Never create schemas inside React components.**

## Zod v4 API

### Top-level string formats

```ts
// BAD (deprecated in v4)
z.string().email();
z.string().uuid();
z.string().url();

// GOOD
z.email();
z.uuidv4();
z.url();
z.ipv4();
z.iso.datetime();
```

### Error customization

```ts
// BAD (v3 patterns)
z.string({ message: 'Required' });
z.string().min(5, { message: 'Too short' });

// GOOD
z.string({ error: 'Required' });
z.string({
  error: (issue) =>
    issue.input === undefined ? 'This field is required' : 'Must be a string',
});
z.string().min(5, { error: 'Too short' });
```

### Coercion for form inputs

HTML inputs always produce strings. Use `z.coerce` for numeric fields.

```ts
// BAD - will fail on string "42" from <input>
z.number().min(0);

// GOOD
z.coerce.number<number>().min(0, { error: 'Must be >= 0' });
```

### Object schemas

```ts
const BASE_SCHEMA = z.object({ email: z.email(), name: z.string().min(1) });

// Extend
const ADMIN_SCHEMA = z.object({
  ...BASE_SCHEMA.shape,
  role: z.literal('admin'),
});

// Pick / Omit / Partial
const PREVIEW_SCHEMA = BASE_SCHEMA.pick({ email: true });
const UPDATE_SCHEMA = BASE_SCHEMA.partial();
```

### Optional fields

Prefer `.nullish()` or `.or(z.literal(''))` — setting `.optional()` to empty string `""` can incorrectly trigger validation errors.

```ts
// BAD - empty string "" triggers validation error
z.email().optional();

// GOOD
z.email().nullish();
// or
z.email().or(z.literal(''));
```

### Discriminated unions for conditional shapes

```ts
z.discriminatedUnion('type', [
  z.object({ type: z.literal('exact'), value: z.string() }),
  z.object({ type: z.literal('range'), min: z.number(), max: z.number() }),
]);
```

### Cross-field validation

Use `.refine()` on the object. Always specify `path` to bind the error to the correct field.

```ts
const PASSWORD_SCHEMA = z
  .object({ password: z.string().min(8), confirmPassword: z.string() })
  .refine((data) => data.password === data.confirmPassword, {
    error: 'Passwords must match.',
    path: ['confirmPassword'],
  });
```

### Transforms and pipes

```ts
// TRANSFORM - after validation
z.string().transform((val) => val.toUpperCase());

// PREPROCESS - before validation
z.preprocess((val) => (val === '' ? undefined : val), z.email().optional());

// PIPE - schema parsing
z.string().transform(Number).pipe(z.number().int().positive());
```

### Runtime-dependent schemas

```ts
const API_KEY_SCHEMA = (orgId: string) =>
  z.object({
    apiKey: z
      .string()
      .refine((val) => validate(orgId, val), { error: 'Invalid key.' }),
  });

type ApiKeyForm = z.infer<ReturnType<typeof API_KEY_SCHEMA>>;
```

### Checkbox acceptance

```ts
z.object({ terms: z.literal(true, { error: 'You must accept the terms' }) });
```

### Parsing

```ts
// User input - handle gracefully
const result = FORM_SCHEMA.safeParse(data);
if (!result.success) console.error(z.prettifyError(result.error));

// Trusted input - throw on failure
const validated = FORM_SCHEMA.parse(requestBody);
```

## React Hook Form Integration

### Basic setup

Schema → infer type → zodResolver → always set `defaultValues`.

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const FORM_SCHEMA = z.object({
  email: z.email({ error: 'Invalid email.' }),
  amount: z.coerce.number<number>().positive({ error: 'Must be positive.' }),
});

type FormSchema = z.infer<typeof FORM_SCHEMA>;

function MyForm() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<FormSchema>({
    resolver: zodResolver(FORM_SCHEMA),
    defaultValues: { email: '', amount: 0 },
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span role="alert">{errors.email.message}</span>}
    </form>
  );
}
```

### Async default values

Use `reset()` after successful fetch. Never set `defaultValues` via `useState`.

```ts
// BAD
const [defaults, setDefaults] = useState({});
useEffect(() => {
  fetchUser().then(setDefaults);
}, []);
const form = useForm({ defaultValues: defaults });

// GOOD
const form = useForm<FormSchema>({
  resolver: zodResolver(SCHEMA),
  defaultValues: { name: '', email: '' },
});
useEffect(() => {
  fetchUser().then((data) => form.reset(data));
}, [form.reset]);
```

### register vs Controller

Use `register` for native HTML inputs. Use `Controller` only for third-party components without native ref forwarding.

```tsx
// Native input
<input {...register('email')} />

// Custom component
<Controller
  name="category"
  control={control}
  render={({ field, fieldState }) => (
    <CustomSelect {...field} isInvalid={!!fieldState.error} errorMessage={fieldState.error?.message} />
  )}
/>
```

### Server error mapping

```ts
async function onSubmit(data: FormSchema) {
  try {
    await submitForm(data);
  } catch {
    setError('email', { type: 'server', message: 'Email already in use.' });
  }
}
```

### useFieldArray

Use `field.id` as the React key. Never use the array index.

```tsx
const { fields, append, remove } = useFieldArray({ control, name: 'contacts' });

{
  fields.map((field, index) => (
    <div key={field.id}>
      <input {...register(`contacts.${index}.name` as const)} />
      {errors.contacts?.[index]?.name?.message && (
        <span>{errors.contacts[index].name.message}</span>
      )}
      <button type="button" onClick={() => remove(index)}>
        Remove
      </button>
    </div>
  ));
}
<button type="button" onClick={() => append({ name: '', email: '' })}>
  Add
</button>;
```

`useFieldArray` only supports arrays of objects. Wrap primitives: `[{ value: 'str' }]` not `['str']`.

### Multi-step forms

```ts
const STEP_1_SCHEMA = z.object({ name: z.string().min(1), email: z.email() });
const STEP_2_SCHEMA = z.object({ address: z.string().min(1) });
const FULL_SCHEMA = z.object({
  ...STEP_1_SCHEMA.shape,
  ...STEP_2_SCHEMA.shape,
});

async function nextStep() {
  const isValid = await trigger(['name', 'email']);
  if (isValid) setStep(2);
}
```

Use `shouldUnregister: true` to clear field data when a step unmounts.

### Watched fields

Watch specific fields, not the entire form. Watching everything causes unnecessary re-renders.

```ts
// BAD
const allValues = watch();

// GOOD
const email = watch('email');
```

## Critical Rules

- Always set `defaultValues` for all fields.
- Always use `z.infer<typeof SCHEMA>` — never manually duplicate types.
- Always specify `path` in `.refine()` for cross-field validations.
- Always use `field.id` (not index) as key in `useFieldArray`.
- Always spread `{...field}` in `Controller` render prop.
- Never mutate form values directly. Use `setValue()`.
- Never mix controlled and uncontrolled patterns on the same field.
- Never create schemas inside React components. Define at module level.
