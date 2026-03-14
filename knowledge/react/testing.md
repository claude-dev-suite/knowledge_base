# React Testing Reference

## Setup (Vitest + Testing Library)

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.ts',
  },
});

// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';

afterEach(() => cleanup());
```

## Basic Component Test

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';

describe('Button', () => {
  it('renders and handles click', async () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument();

    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click me</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });
});
```

## Query Methods (Priority Order)

```tsx
// 1. Accessible queries (preferred)
screen.getByRole('button', { name: /submit/i });
screen.getByRole('textbox', { name: /email/i });
screen.getByRole('heading', { level: 1 });

// 2. Label text (form fields)
screen.getByLabelText(/email address/i);

// 3. Placeholder
screen.getByPlaceholderText(/enter email/i);

// 4. Text content
screen.getByText(/welcome/i);

// 5. Display value (inputs)
screen.getByDisplayValue(/john@example.com/i);

// 6. Alt text (images)
screen.getByAltText(/user avatar/i);

// 7. Test ID (last resort)
screen.getByTestId('custom-element');

// Query variants
screen.getByRole('button');      // Throws if not found
screen.queryByRole('button');    // Returns null if not found
screen.findByRole('button');     // Returns Promise, waits
screen.getAllByRole('button');   // Returns array
```

## User Events

```tsx
import userEvent from '@testing-library/user-event';

it('handles form submission', async () => {
  const user = userEvent.setup();
  const handleSubmit = vi.fn();
  render(<LoginForm onSubmit={handleSubmit} />);

  await user.type(screen.getByLabelText(/email/i), 'test@example.com');
  await user.type(screen.getByLabelText(/password/i), 'secret123');
  await user.click(screen.getByRole('button', { name: /sign in/i }));

  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'secret123',
  });
});

// Keyboard navigation
await user.tab();
expect(screen.getByLabelText(/email/i)).toHaveFocus();

// Select and checkbox
await user.selectOptions(screen.getByRole('combobox'), ['dark']);
await user.click(screen.getByRole('checkbox'));
```

## Async Testing

```tsx
import { waitFor, waitForElementToBeRemoved } from '@testing-library/react';

it('loads and displays data', async () => {
  render(<UserProfile userId="123" />);

  // Initially shows loading
  expect(screen.getByText(/loading/i)).toBeInTheDocument();

  // Wait for data
  await waitFor(() => {
    expect(screen.getByText('John Doe')).toBeInTheDocument();
  });

  // Loading is gone
  expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
});

// Using findBy (combines getBy + waitFor)
const items = await screen.findAllByRole('listitem');
expect(items).toHaveLength(3);

// Wait for removal
await waitForElementToBeRemoved(() => screen.queryByText(/loading/i));
```

## Mocking

```tsx
// Mock function
const mockFn = vi.fn();
mockFn.mockReturnValue('default');
mockFn.mockResolvedValue({ data: [] });

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledWith('arg1');

// Mock module
vi.mock('@/lib/api', () => ({
  fetchUsers: vi.fn(() => Promise.resolve([{ id: 1, name: 'John' }])),
}));
```

## MSW (Mock Service Worker)

```tsx
// src/test/server.ts
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: 1, name: 'John' },
      { id: 2, name: 'Jane' },
    ]);
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 3, ...body }, { status: 201 });
  }),
];

export const server = setupServer(...handlers);

// src/test/setup.ts
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Override in tests
it('handles error', async () => {
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.json({ error: 'Server error' }, { status: 500 });
    })
  );
  render(<UserList />);
  await screen.findByText(/error/i);
});
```

## Testing Custom Hooks

```tsx
import { renderHook, act } from '@testing-library/react';

function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  return { count, increment };
}

it('increments count', () => {
  const { result } = renderHook(() => useCounter());

  expect(result.current.count).toBe(0);

  act(() => result.current.increment());

  expect(result.current.count).toBe(1);
});
```

## Testing with Providers

```tsx
function renderWithProviders(ui: React.ReactElement, options = {}) {
  function Wrapper({ children }) {
    return (
      <QueryClientProvider client={queryClient}>
        <AuthProvider>
          {children}
        </AuthProvider>
      </QueryClientProvider>
    );
  }
  return render(ui, { wrapper: Wrapper, ...options });
}

// Usage
it('shows user when authenticated', () => {
  renderWithProviders(<Header />, {
    preloadedState: { auth: { user: { name: 'John' } } },
  });
  expect(screen.getByText('John')).toBeInTheDocument();
});
```

## Accessibility Testing

```tsx
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

it('has no accessibility violations', async () => {
  const { container } = render(<LoginForm />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

## Best Practices

| Do | Don't |
|----|-------|
| Test behavior | Test implementation |
| Use accessible queries | Use test IDs everywhere |
| Use userEvent | Use fireEvent |
| Test error states | Only test happy path |
| Use MSW for API mocking | Mock fetch directly |
