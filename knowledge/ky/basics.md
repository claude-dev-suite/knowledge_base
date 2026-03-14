# ky Basics

ky is a modern HTTP client based on Fetch API for browsers.

## Installation

```bash
npm install ky
```

## Basic Usage

```typescript
import ky from 'ky';

// Simple GET
const users = await ky.get('https://api.example.com/users').json<User[]>();

// With base URL
const api = ky.create({ prefixUrl: 'https://api.example.com' });
const users = await api.get('users').json<User[]>();
```

## HTTP Methods

```typescript
// GET
const users = await api.get('users').json<User[]>();
const user = await api.get('users/123').json<User>();

// POST
const user = await api.post('users', {
  json: { name: 'John', email: 'john@example.com' },
}).json<User>();

// PUT
await api.put('users/123', {
  json: { name: 'Updated', email: 'updated@example.com' },
});

// PATCH
await api.patch('users/123', { json: { name: 'Updated' } });

// DELETE
await api.delete('users/123');
```

## Instance Configuration

```typescript
const api = ky.create({
  prefixUrl: 'https://api.example.com',
  timeout: 10000,
  retry: {
    limit: 2,
    methods: ['get', 'put', 'delete'],
    statusCodes: [408, 429, 500, 502, 503, 504],
    backoffLimit: 3000,
  },
  headers: {
    'Accept': 'application/json',
  },
});
```

## Request Options

```typescript
await api.get('users', {
  searchParams: { status: 'active', page: 1 },
  headers: { 'X-Custom': 'value' },
  timeout: 5000,
  retry: 0,
  signal: controller.signal,
});

// POST options
await api.post('users', {
  json: { name: 'John' },    // JSON body
  body: formData,             // FormData
  headers: { 'X-Custom': 'value' },
});
```

## Hooks

```typescript
const api = ky.create({
  prefixUrl: 'https://api.example.com',
  hooks: {
    beforeRequest: [
      (request) => {
        const token = localStorage.getItem('token');
        if (token) {
          request.headers.set('Authorization', `Bearer ${token}`);
        }
      },
    ],
    afterResponse: [
      async (request, options, response) => {
        if (response.status === 401) {
          const newToken = await refreshToken();
          request.headers.set('Authorization', `Bearer ${newToken}`);
          return ky(request);
        }
        return response;
      },
    ],
    beforeRetry: [
      ({ retryCount }) => {
        console.log(`Retry attempt ${retryCount}`);
      },
    ],
    beforeError: [
      async (error) => {
        const { response } = error;
        if (response?.body) {
          error.message = (await response.json()).message;
        }
        return error;
      },
    ],
  },
});
```

## Response Methods

```typescript
const response = await api.get('users');

// Parse response (can chain directly)
const json = await response.json<User[]>();
const text = await response.text();
const blob = await response.blob();
const arrayBuffer = await response.arrayBuffer();
const formData = await response.formData();

// Shorthand
const users = await api.get('users').json<User[]>();
```

## Error Handling

```typescript
import ky, { HTTPError, TimeoutError } from 'ky';

try {
  await api.get('users');
} catch (error) {
  if (error instanceof HTTPError) {
    const body = await error.response.json();
    console.log(error.response.status); // 404
    console.log(body.message);          // Error message
  } else if (error instanceof TimeoutError) {
    console.log('Request timed out');
  }
}
```

## Request Cancellation

```typescript
const controller = new AbortController();

api.get('users', { signal: controller.signal })
  .json<User[]>()
  .catch((error) => {
    if (error.name === 'AbortError') {
      console.log('Request cancelled');
    }
  });

// Cancel
controller.abort();
```

## Extending Instance

```typescript
// Create new instance with additional options
const apiWithAuth = api.extend({
  hooks: {
    beforeRequest: [
      (request) => {
        request.headers.set('Authorization', 'Bearer token');
      },
    ],
  },
});
```

## TypeScript

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

// Typed response
const users = await api.get('users').json<User[]>();
const user = await api.get('users/123').json<User>();

// Type inference
type UsersResponse = Awaited<ReturnType<typeof api.get<User[]>>>;
```

## File Upload

```typescript
const formData = new FormData();
formData.append('file', file);

await api.post('upload', {
  body: formData,
  // Don't set Content-Type - ky handles it
});
```

## Comparison with fetch

```typescript
// Native fetch
const response = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'John' }),
});
const data = await response.json();

// ky equivalent
const data = await ky.post('https://api.example.com/users', {
  json: { name: 'John' },
}).json();
```
