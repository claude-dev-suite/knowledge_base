# Axios Interceptors

## Request Interceptors

```typescript
import axios, { InternalAxiosRequestConfig } from 'axios';

const api = axios.create({ baseURL: 'https://api.example.com' });

// Add auth token
api.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Add request ID
api.interceptors.request.use((config) => {
  config.headers['X-Request-ID'] = crypto.randomUUID();
  return config;
});

// Log requests
api.interceptors.request.use((config) => {
  console.log(`[${config.method?.toUpperCase()}] ${config.url}`);
  return config;
});
```

## Response Interceptors

```typescript
import { AxiosResponse, AxiosError } from 'axios';

// Transform response
api.interceptors.response.use(
  (response: AxiosResponse) => {
    // Unwrap data envelope
    if (response.data?.data) {
      response.data = response.data.data;
    }
    return response;
  }
);

// Global error handling
api.interceptors.response.use(
  (response) => response,
  (error: AxiosError) => {
    if (error.response?.status === 401) {
      window.location.href = '/login';
    }
    if (error.response?.status === 403) {
      toast.error('Permission denied');
    }
    if (error.response?.status >= 500) {
      toast.error('Server error');
    }
    return Promise.reject(error);
  }
);
```

## Token Refresh

```typescript
let isRefreshing = false;
let failedQueue: Array<{
  resolve: (token: string) => void;
  reject: (error: Error) => void;
}> = [];

const processQueue = (error: Error | null, token: string | null) => {
  failedQueue.forEach((prom) => {
    error ? prom.reject(error) : prom.resolve(token!);
  });
  failedQueue = [];
};

api.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & {
      _retry?: boolean;
    };

    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error);
    }

    if (isRefreshing) {
      return new Promise((resolve, reject) => {
        failedQueue.push({ resolve, reject });
      }).then((token) => {
        originalRequest.headers.Authorization = `Bearer ${token}`;
        return api(originalRequest);
      });
    }

    originalRequest._retry = true;
    isRefreshing = true;

    try {
      const { data } = await axios.post('/auth/refresh', {
        refreshToken: localStorage.getItem('refreshToken'),
      });

      localStorage.setItem('accessToken', data.accessToken);
      localStorage.setItem('refreshToken', data.refreshToken);
      api.defaults.headers.common.Authorization = `Bearer ${data.accessToken}`;
      processQueue(null, data.accessToken);

      originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
      return api(originalRequest);
    } catch (refreshError) {
      processQueue(refreshError as Error, null);
      localStorage.clear();
      window.location.href = '/login';
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  }
);
```

## Remove Interceptor

```typescript
const requestInterceptor = api.interceptors.request.use((config) => config);
const responseInterceptor = api.interceptors.response.use((res) => res);

// Remove later
api.interceptors.request.eject(requestInterceptor);
api.interceptors.response.eject(responseInterceptor);
```

## Request Timing

```typescript
api.interceptors.request.use((config) => {
  (config as any).metadata = { startTime: Date.now() };
  return config;
});

api.interceptors.response.use((response) => {
  const duration = Date.now() - (response.config as any).metadata.startTime;
  console.log(`${response.config.url} - ${duration}ms`);
  return response;
});
```

## Retry Logic

```typescript
api.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const config = error.config as InternalAxiosRequestConfig & {
      _retryCount?: number;
    };

    if (!config || (config._retryCount ?? 0) >= 3) {
      return Promise.reject(error);
    }

    // Retry on network errors or 5xx
    const shouldRetry =
      !error.response || error.response.status >= 500;

    if (shouldRetry) {
      config._retryCount = (config._retryCount ?? 0) + 1;
      const delay = Math.pow(2, config._retryCount) * 1000;
      await new Promise((r) => setTimeout(r, delay));
      return api(config);
    }

    return Promise.reject(error);
  }
);
```

## Error Normalization

```typescript
interface NormalizedError {
  status: number;
  code: string;
  message: string;
}

api.interceptors.response.use(
  (response) => response,
  (error: AxiosError) => {
    const normalized: NormalizedError = {
      status: error.response?.status ?? 0,
      code: 'UNKNOWN',
      message: 'An error occurred',
    };

    if (error.response?.data) {
      const data = error.response.data as any;
      normalized.code = data.code ?? normalized.code;
      normalized.message = data.message ?? normalized.message;
    } else if (!error.response) {
      normalized.code = 'NETWORK_ERROR';
      normalized.message = 'Network error';
    }

    return Promise.reject(normalized);
  }
);
```
