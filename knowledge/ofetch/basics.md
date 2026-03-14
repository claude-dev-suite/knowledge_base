# ofetch Basics

ofetch is a universal fetch library that works in Node.js, browsers, and edge runtimes. It's the HTTP client used by Nuxt.

## Installation

```bash
npm install ofetch
```

## Basic Usage

```typescript
import { ofetch } from 'ofetch';

// Simple GET
const users = await ofetch<User[]>('https://api.example.com/users');

// With options
const user = await ofetch<User>('https://api.example.com/users', {
  method: 'POST',
  body: { name: 'John' },
});
```

## Creating an Instance

```typescript
import { ofetch } from 'ofetch';

const api = ofetch.create({
  baseURL: 'https://api.example.com',
  retry: 2,
  retryDelay: 500,
  timeout: 10000,
  headers: {
    'Accept': 'application/json',
  },
});

// Usage
const users = await api<User[]>('/users');
```

## HTTP Methods

```typescript
// GET
const users = await api<User[]>('/users');
const users = await api<User[]>('/users', { query: { status: 'active' } });

// POST
const user = await api<User>('/users', {
  method: 'POST',
  body: { name: 'John', email: 'john@example.com' },
});

// PUT
await api('/users/123', {
  method: 'PUT',
  body: { name: 'Updated', email: 'updated@example.com' },
});

// PATCH
await api('/users/123', {
  method: 'PATCH',
  body: { name: 'Updated' },
});

// DELETE
await api('/users/123', { method: 'DELETE' });
```

## Request Options

```typescript
interface FetchOptions {
  method?: string;
  body?: any;                    // Auto JSON.stringify for objects
  query?: Record<string, any>;   // URL query params
  headers?: Record<string, string>;
  timeout?: number;              // Milliseconds
  retry?: number;                // Retry count
  retryDelay?: number;           // Delay between retries
  signal?: AbortSignal;          // Cancellation
  responseType?: 'json' | 'text' | 'blob' | 'arrayBuffer';
  parseResponse?: (text: string) => any;  // Custom parser
  ignoreResponseError?: boolean;
}
```

## Interceptors

```typescript
const api = ofetch.create({
  baseURL: 'https://api.example.com',

  async onRequest({ request, options }) {
    // Add auth header
    const token = getToken();
    if (token) {
      options.headers = {
        ...options.headers,
        Authorization: `Bearer ${token}`,
      };
    }
    console.log(`[${options.method || 'GET'}] ${request}`);
  },

  async onRequestError({ request, error }) {
    console.error('Request failed:', error);
  },

  async onResponse({ response }) {
    console.log(`Response: ${response.status}`);
  },

  async onResponseError({ response }) {
    if (response.status === 401) {
      await redirectToLogin();
    }
  },
});
```

## Error Handling

```typescript
import { ofetch, FetchError } from 'ofetch';

try {
  await api('/users/123');
} catch (error) {
  if (error instanceof FetchError) {
    console.log('Status:', error.status);      // 404
    console.log('Status Text:', error.statusText);
    console.log('Data:', error.data);          // Response body
    console.log('Request:', error.request);
    console.log('Response:', error.response);
  }
}
```

## Query Parameters

```typescript
// Object syntax
const users = await api('/users', {
  query: {
    status: 'active',
    page: 1,
    limit: 10,
  },
});
// → /users?status=active&page=1&limit=10

// Arrays
const users = await api('/users', {
  query: {
    tags: ['javascript', 'typescript'],
  },
});
// → /users?tags=javascript&tags=typescript
```

## Request Cancellation

```typescript
const controller = new AbortController();

ofetch('/users', { signal: controller.signal })
  .catch((error) => {
    if (error.name === 'AbortError') {
      console.log('Request cancelled');
    }
  });

// Cancel
controller.abort();
```

## Node.js Usage

```typescript
// Works in Node.js without polyfills
import { ofetch } from 'ofetch';

const data = await ofetch('https://api.example.com/data');

// With native fetch options
const data = await ofetch('https://api.example.com/data', {
  // All native fetch options work
  cache: 'no-store',
  credentials: 'include',
});
```

## Nuxt Integration

```typescript
// In Nuxt, $fetch is built on ofetch and globally available

// pages/users.vue
const users = await $fetch<User[]>('/api/users');

// With runtime config
const config = useRuntimeConfig();
const users = await $fetch<User[]>(`${config.public.apiUrl}/users`);

// Custom instance
// composables/useApi.ts
export function useApi() {
  const config = useRuntimeConfig();

  return $fetch.create({
    baseURL: config.public.apiUrl,
    async onRequest({ options }) {
      const { token } = useAuth();
      if (token.value) {
        options.headers = {
          ...options.headers,
          Authorization: `Bearer ${token.value}`,
        };
      }
    },
  });
}

// Usage
const api = useApi();
const users = await api<User[]>('/users');
```

## Raw Response

```typescript
import { ofetch } from 'ofetch';

// Get raw response
const response = await ofetch.raw('/users');
console.log(response.status);
console.log(response.headers);
console.log(response._data);  // Parsed body
```

## TypeScript

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

// Typed responses
const users = await api<User[]>('/users');
const user = await api<User>('/users/123');

// Type-safe API client
const apiClient = {
  users: {
    list: () => api<User[]>('/users'),
    get: (id: string) => api<User>(`/users/${id}`),
    create: (data: Omit<User, 'id'>) =>
      api<User>('/users', { method: 'POST', body: data }),
    update: (id: string, data: Partial<User>) =>
      api<User>(`/users/${id}`, { method: 'PATCH', body: data }),
    delete: (id: string) =>
      api(`/users/${id}`, { method: 'DELETE' }),
  },
};
```

## Retry Logic

```typescript
const api = ofetch.create({
  baseURL: 'https://api.example.com',
  retry: 3,           // Max retries
  retryDelay: 1000,   // Delay between retries (ms)
  // Retries on: 408, 409, 425, 429, 500, 502, 503, 504
});

// Disable retry for specific request
await api('/users', { retry: false });
```

## Comparison with Native Fetch

```typescript
// Native fetch
const response = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'John' }),
});
if (!response.ok) throw new Error(`HTTP ${response.status}`);
const data = await response.json();

// ofetch equivalent
const data = await ofetch('https://api.example.com/users', {
  method: 'POST',
  body: { name: 'John' },  // Auto JSON
});
// Auto throws on non-2xx, auto parses JSON
```
