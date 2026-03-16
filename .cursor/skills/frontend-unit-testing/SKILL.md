---
name: unit-testing
description: Use when writing, debugging, or structuring Vitest and React Testing Library unit tests — test naming, AAA pattern, query priority, userEvent, mocking, async testing, fake timers, test data factories, or custom render wrappers. Triggers on any Vitest or RTL task.
---

# Unit Testing (Vitest + React Testing Library)

**Official Docs**: [Vitest](https://vitest.dev/) | [React Testing Library](https://testing-library.com/docs/react-testing-library/intro)

## Test File Placement

Colocate test files next to the source file. Name them `<source-file>.test.ts(x)`.

```
# GOOD - colocated
src/
  components/
    UserCard.tsx
    UserCard.test.tsx
```

## Test Description Naming

All test descriptions must follow `it('should <behavior> when <condition>')`.

```ts
// BAD
it('renders the user list');
it('handles click');

// GOOD
it('should return parsed date as YYYY-MM when input is in ISO date format');
it('should disable submit button when form is invalid');
```

## AAA Pattern

Every test clearly separates three phases. One behavior per test.

```tsx
it('should increment count when plus button is clicked', async () => {
  // Arrange
  const user = userEvent.setup();
  render(<Counter initialCount={0} />);

  // Act
  await user.click(screen.getByRole('button', { name: /increment/i }));

  // Assert
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

## Black-Box Testing

Test observable behavior, never implementation details. Do not access internal state, internal methods, or CSS class names as a proxy for behavior.

```tsx
// BAD
expect(component.state.isOpen).toBe(true);
expect(wrapper.find('.internal-class').length).toBe(1);

// GOOD
expect(screen.getByRole('dialog')).toBeInTheDocument();
expect(screen.getByRole('button')).toBeDisabled();
```

## Query Priority

1. `getByRole` — accessible role + name
2. `getByLabelText` — form fields with labels
3. `getByPlaceholderText` — when no label exists
4. `getByText` — non-interactive content
5. `getByDisplayValue` — current form input value
6. `getByAltText`, `getByTitle`
7. `getByTestId` — last resort only

```tsx
// BAD
screen.getByText('Submit Form');
screen.getByTestId('submit-btn');

// GOOD
screen.getByRole('button', { name: /submit/i });
```

Use regex with case-insensitive flag to tolerate copy changes.

## Query Types: get vs query vs find

| Variant   | No match       | Use case                         |
| --------- | -------------- | -------------------------------- |
| `getBy`   | Throws         | Element must exist               |
| `queryBy` | `null`         | Asserting element does NOT exist |
| `findBy`  | Throws (async) | Element appears asynchronously   |

```tsx
// BAD - throws before .not can evaluate
expect(screen.getByText('Error')).not.toBeInTheDocument();

// GOOD
expect(screen.queryByText('Error')).not.toBeInTheDocument();

// GOOD - async appearance
const title = await screen.findByText('Dashboard');
```

## userEvent over fireEvent

Use `@testing-library/user-event` for all interactions.

```tsx
// BAD
fireEvent.click(screen.getByRole('button'));
fireEvent.change(input, { target: { value: 'text' } });

// GOOD - call setup() before render
const user = userEvent.setup();
await user.click(screen.getByRole('button'));
await user.type(screen.getByRole('textbox'), 'text');
await user.keyboard('{Enter}');
```

## Test Isolation

Reset mocks in `beforeEach`, not `afterEach`.

```tsx
describe('MyComponent', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });
});
```

RTL calls `cleanup` automatically. Do not call it manually.

## Mocking Boundaries

Mock at the boundary of your system: network calls, third-party libraries with side effects, environment APIs. Do not mock child components, internal hooks, or utility functions.

```tsx
// BAD - mocking child components
vi.mock('./ChildComponent', () => () => <div>Mock</div>);

// GOOD - mock network layer
vi.mock('@/service/api');
const mockedApi = vi.mocked(api);
mockedApi.fetchUsers.mockResolvedValue([{ id: '1', name: 'Ada' }]);
```

## Async Testing

Always await async operations:

```tsx
// BAD - floating promise
waitFor(() => {
  expect(screen.getByText('Data')).toBeInTheDocument();
});

// GOOD
await waitFor(() => {
  expect(screen.getByText('Data')).toBeInTheDocument();
});
```

Avoid multiple assertions in a single `waitFor`:

```tsx
// BAD - if first fails, second is never checked
await waitFor(() => {
  expect(screen.getByText('Title')).toBeInTheDocument();
  expect(screen.getByText('Description')).toBeInTheDocument();
});

// GOOD
const title = await screen.findByText('Title');
const description = await screen.findByText('Description');
```

Test loading, success, and error states:

```tsx
it('should show spinner when data is loading', () => {
  mockedApi.fetchData.mockImplementation(() => new Promise(() => {}));
  render(<DataFetcher />);
  expect(screen.getByRole('status')).toBeInTheDocument();
});

it('should render items when fetch succeeds', async () => {
  mockedApi.fetchData.mockResolvedValue({ items: ['Item 1'] });
  render(<DataFetcher />);
  expect(await screen.findByText('Item 1')).toBeInTheDocument();
});

it('should show error message when fetch fails', async () => {
  mockedApi.fetchData.mockRejectedValue(new Error('Network error'));
  render(<DataFetcher />);
  expect(await screen.findByRole('alert')).toBeInTheDocument();
});
```

## Fake Timers

```tsx
beforeEach(() => {
  vi.useFakeTimers();
});
afterEach(() => {
  vi.useRealTimers();
});

it('should debounce search when user types', () => {
  const onSearch = vi.fn();
  render(<SearchInput onSearch={onSearch} debounceMs={300} />);
  fireEvent.change(screen.getByRole('textbox'), { target: { value: 'query' } });
  expect(onSearch).not.toHaveBeenCalled();
  vi.advanceTimersByTime(300);
  expect(onSearch).toHaveBeenCalledWith('query');
});
```

Do not mix `vi.useFakeTimers()` with `waitFor` or `findBy`.

## Factory Functions for Test Data

```ts
import type { User } from '@/types';

function createMockUser(overrides: Partial<User> = {}): User {
  return {
    id: 'user-1',
    name: 'Test User',
    email: 'test@example.com',
    role: 'member',
    ...overrides,
  };
}
```

## Custom Render Wrapper

Create a fresh `QueryClient` per test.

```tsx
function createTestQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },
      mutations: { retry: false },
    },
  });
}

function renderWithProviders(ui: React.ReactElement) {
  const queryClient = createTestQueryClient();
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>,
  );
}
```

Never share a `QueryClient` across tests. Set `retry: false`, `gcTime: Infinity`.

## Data-Driven Tests with test.each

```tsx
test.each([
  { status: 'success', expectedClass: 'bg-green-500' },
  { status: 'error', expectedClass: 'bg-red-500' },
])(
  'should apply $expectedClass when status is $status',
  ({ status, expectedClass }) => {
    render(<StatusBadge status={status} />);
    expect(screen.getByRole('status')).toHaveClass(expectedClass);
  },
);
```

## Rerender for Prop Changes

```tsx
it('should refetch data when id prop changes', async () => {
  const fetchData = vi.fn().mockResolvedValue({ name: 'A' });
  const { rerender } = render(<Profile id="1" fetchData={fetchData} />);
  await waitFor(() => expect(fetchData).toHaveBeenCalledWith('1'));
  rerender(<Profile id="2" fetchData={fetchData} />);
  await waitFor(() => expect(fetchData).toHaveBeenCalledWith('2'));
});
```

## What NOT to Test

- Third-party library internals (React, router, query library)
- Static markup with no logic
- Implementation details (internal state, private methods, CSS classes)

## Critical Rules Summary

- Colocate test files: `Foo.tsx` → `Foo.test.tsx` in the same directory
- Name tests `should ... when ...`
- One behavior per test, follow AAA
- Prefer `getByRole`; `queryBy` for absence; `findBy` for async
- `userEvent.setup()` over `fireEvent`
- `vi.clearAllMocks()` in `beforeEach`
- Mock network and side-effect boundaries only
- Always `await` async assertions
- Do not mix fake timers with `waitFor`
- Fresh `QueryClient` per test, `retry: false`, `gcTime: Infinity`
- No snapshot tests unless strongly justified
