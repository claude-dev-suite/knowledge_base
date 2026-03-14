# Axios Basics

## Installation

```bash
npm install axios
```

## Creating an Instance

```typescript
import axios from 'axios';

const api = axios.create({
  baseURL: 'https://api.example.com/v1',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});
```

## HTTP Methods

```typescript
// GET
const { data } = await api.get<User[]>('/users');
const { data } = await api.get<User>('/users/123');
const { data } = await api.get('/users', {
  params: { status: 'active', page: 1 },
});

// POST
const { data } = await api.post<User>('/users', {
  name: 'John',
  email: 'john@example.com',
});

// PUT (full update)
const { data } = await api.put<User>('/users/123', {
  name: 'John Updated',
  email: 'john@example.com',
});

// PATCH (partial update)
const { data } = await api.patch<User>('/users/123', {
  name: 'John Updated',
});

// DELETE
await api.delete('/users/123');
```

## Request Configuration

```typescript
interface AxiosRequestConfig {
  url?: string;
  method?: 'get' | 'post' | 'put' | 'patch' | 'delete';
  baseURL?: string;
  headers?: Record<string, string>;
  params?: Record<string, any>;      // URL query params
  data?: any;                         // Request body
  timeout?: number;                   // Milliseconds
  withCredentials?: boolean;          // Send cookies
  responseType?: 'json' | 'text' | 'blob' | 'arraybuffer';
  signal?: AbortSignal;              // Cancellation
  validateStatus?: (status: number) => boolean;
  onUploadProgress?: (event: AxiosProgressEvent) => void;
  onDownloadProgress?: (event: AxiosProgressEvent) => void;
}
```

## Response Structure

```typescript
interface AxiosResponse<T> {
  data: T;                 // Response body
  status: number;          // HTTP status code
  statusText: string;      // HTTP status message
  headers: AxiosHeaders;   // Response headers
  config: AxiosRequestConfig;
}

// Usage
const response = await api.get<User>('/users/123');
console.log(response.data);    // User object
console.log(response.status);  // 200
```

## Error Handling

```typescript
import { AxiosError, isAxiosError } from 'axios';

try {
  await api.get('/users/123');
} catch (error) {
  if (isAxiosError(error)) {
    // HTTP error response
    if (error.response) {
      console.log(error.response.status);    // 404
      console.log(error.response.data);      // Error body
    }
    // Network error
    else if (error.request) {
      console.log('No response received');
    }
    // Request setup error
    else {
      console.log(error.message);
    }
  }
}
```

## Request Cancellation

```typescript
const controller = new AbortController();

// Start request
api.get('/users', { signal: controller.signal })
  .then(({ data }) => console.log(data))
  .catch((error) => {
    if (axios.isCancel(error)) {
      console.log('Request cancelled');
    }
  });

// Cancel request
controller.abort();
```

## File Upload

```typescript
const formData = new FormData();
formData.append('file', file);
formData.append('description', 'My file');

const { data } = await api.post('/upload', formData, {
  headers: {
    'Content-Type': 'multipart/form-data',
  },
  onUploadProgress: (event) => {
    const progress = Math.round((event.loaded * 100) / (event.total ?? 1));
    console.log(`Upload: ${progress}%`);
  },
});
```

## TypeScript Types

```typescript
import type {
  AxiosInstance,
  AxiosRequestConfig,
  AxiosResponse,
  AxiosError,
  InternalAxiosRequestConfig,
} from 'axios';

// Generic request
async function request<T>(config: AxiosRequestConfig): Promise<T> {
  const { data } = await api.request<T>(config);
  return data;
}
```

## Global Defaults

```typescript
// Set defaults for all requests
axios.defaults.baseURL = 'https://api.example.com';
axios.defaults.headers.common['Authorization'] = 'Bearer token';
axios.defaults.timeout = 10000;

// Instance defaults
api.defaults.headers.common['X-Custom'] = 'value';
```
