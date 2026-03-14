# Angular HttpClient

> Official Documentation: https://angular.dev/guide/http

## HttpClient Setup

### Configuring HttpClient with provideHttpClient

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors, withFetch, withXsrfConfiguration } from '@angular/common/http';
import { authInterceptor } from './interceptors/auth.interceptor';
import { loggingInterceptor } from './interceptors/logging.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, loggingInterceptor]),
      withFetch(),                          // Use fetch API instead of XMLHttpRequest
      withXsrfConfiguration({               // CSRF/XSRF protection
        cookieName: 'XSRF-TOKEN',
        headerName: 'X-XSRF-TOKEN'
      })
    )
  ]
};
```

### HttpClient Feature Functions

| Feature | Purpose |
|---------|---------|
| `withInterceptors([...])` | Register functional interceptors |
| `withInterceptorsFromDi()` | Enable class-based interceptors from DI |
| `withFetch()` | Use the Fetch API backend instead of XMLHttpRequest |
| `withXsrfConfiguration({...})` | Configure XSRF/CSRF protection |
| `withNoXsrfProtection()` | Disable XSRF protection |
| `withJsonpSupport()` | Enable JSONP requests |
| `withRequestsMadeViaParent()` | Pass requests through parent injector's interceptors |

---

## Making HTTP Requests

### GET Requests

```typescript
import { Component, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

@Component({
  selector: 'app-product-list',
  template: `
    @if (loading()) {
      <app-spinner />
    }
    @for (product of products(); track product.id) {
      <app-product-card [product]="product" />
    }
  `
})
export class ProductListComponent {
  private http = inject(HttpClient);

  products = signal<Product[]>([]);
  loading = signal(true);

  constructor() {
    // Simple GET request
    this.http.get<Product[]>('/api/products')
      .pipe(takeUntilDestroyed())
      .subscribe({
        next: (products) => this.products.set(products),
        error: (err) => console.error('Failed to load products:', err),
        complete: () => this.loading.set(false)
      });
  }
}
```

### GET with Query Parameters

```typescript
import { HttpParams } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);
  private baseUrl = '/api/products';

  search(filters: { category?: string; minPrice?: number; maxPrice?: number; page?: number; size?: number }): Observable<PaginatedResponse<Product>> {
    // Build params immutably
    let params = new HttpParams();

    if (filters.category) {
      params = params.set('category', filters.category);
    }
    if (filters.minPrice !== undefined) {
      params = params.set('minPrice', filters.minPrice.toString());
    }
    if (filters.maxPrice !== undefined) {
      params = params.set('maxPrice', filters.maxPrice.toString());
    }
    params = params.set('page', (filters.page ?? 0).toString());
    params = params.set('size', (filters.size ?? 20).toString());

    return this.http.get<PaginatedResponse<Product>>(this.baseUrl, { params });
  }

  // Alternative: pass params as object (simpler for basic cases)
  getByCategory(category: string): Observable<Product[]> {
    return this.http.get<Product[]>(this.baseUrl, {
      params: { category, inStock: 'true' }
    });
  }
}
```

### POST Requests

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private baseUrl = '/api/users';

  create(user: CreateUserDto): Observable<User> {
    return this.http.post<User>(this.baseUrl, user);
  }

  // POST with custom headers
  createWithAuth(user: CreateUserDto, token: string): Observable<User> {
    return this.http.post<User>(this.baseUrl, user, {
      headers: {
        'Authorization': `Bearer ${token}`,
        'X-Request-Source': 'web-app'
      }
    });
  }
}
```

### PUT, PATCH, DELETE

```typescript
@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);
  private baseUrl = '/api/products';

  // PUT - full replacement
  update(id: number, product: UpdateProductDto): Observable<Product> {
    return this.http.put<Product>(`${this.baseUrl}/${id}`, product);
  }

  // PATCH - partial update
  patchPrice(id: number, price: number): Observable<Product> {
    return this.http.patch<Product>(`${this.baseUrl}/${id}`, { price });
  }

  // DELETE
  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }

  // DELETE with body (some APIs require it)
  bulkDelete(ids: number[]): Observable<void> {
    return this.http.delete<void>(this.baseUrl, {
      body: { ids }
    });
  }
}
```

---

## Typed Responses

### Observe Options

```typescript
@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);

  // Default: observe body only
  getBody(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }

  // Full response (access headers, status)
  getFullResponse(): Observable<HttpResponse<Product[]>> {
    return this.http.get<Product[]>('/api/products', {
      observe: 'response'
    });
  }

  // Raw events (includes progress, sent, response events)
  getEvents(): Observable<HttpEvent<Product[]>> {
    return this.http.get<Product[]>('/api/products', {
      observe: 'events'
    });
  }
}
```

### Accessing Response Headers and Status

```typescript
@Injectable({ providedIn: 'root' })
export class PaginatedService {
  private http = inject(HttpClient);

  getPage(page: number): Observable<{ data: Product[]; total: number }> {
    return this.http.get<Product[]>('/api/products', {
      observe: 'response',
      params: { page: page.toString(), size: '20' }
    }).pipe(
      map(response => ({
        data: response.body ?? [],
        total: parseInt(response.headers.get('X-Total-Count') ?? '0', 10)
      }))
    );
  }
}
```

---

## Functional Interceptors

Functional interceptors are the preferred approach in Angular 17+.

### Authentication Interceptor

```typescript
// interceptors/auth.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, switchMap, throwError } from 'rxjs';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getAccessToken();

  // Skip auth for public endpoints
  if (req.url.includes('/public/') || req.url.includes('/auth/login')) {
    return next(req);
  }

  // Clone request and add auth header
  const authReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        // Attempt token refresh
        return authService.refreshToken().pipe(
          switchMap(newToken => {
            const retryReq = req.clone({
              setHeaders: { Authorization: `Bearer ${newToken}` }
            });
            return next(retryReq);
          }),
          catchError(refreshError => {
            authService.logout();
            return throwError(() => refreshError);
          })
        );
      }
      return throwError(() => error);
    })
  );
};
```

### Logging Interceptor

```typescript
export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const startTime = performance.now();

  console.log(`[HTTP] ${req.method} ${req.url}`);

  return next(req).pipe(
    tap({
      next: (event) => {
        if (event instanceof HttpResponse) {
          const duration = Math.round(performance.now() - startTime);
          console.log(`[HTTP] ${req.method} ${req.url} - ${event.status} (${duration}ms)`);
        }
      },
      error: (error: HttpErrorResponse) => {
        const duration = Math.round(performance.now() - startTime);
        console.error(`[HTTP] ${req.method} ${req.url} - ${error.status} (${duration}ms)`, error.message);
      }
    })
  );
};
```

### Caching Interceptor

```typescript
export const cachingInterceptor: HttpInterceptorFn = (req, next) => {
  // Only cache GET requests
  if (req.method !== 'GET') {
    return next(req);
  }

  const cache = inject(HttpCacheService);
  const cached = cache.get(req.url);

  if (cached) {
    return of(cached.clone());
  }

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        cache.set(req.url, event.clone());
      }
    })
  );
};

@Injectable({ providedIn: 'root' })
export class HttpCacheService {
  private cache = new Map<string, { response: HttpResponse<unknown>; timestamp: number }>();
  private ttl = 5 * 60 * 1000; // 5 minutes

  get(url: string): HttpResponse<unknown> | null {
    const entry = this.cache.get(url);
    if (!entry) return null;
    if (Date.now() - entry.timestamp > this.ttl) {
      this.cache.delete(url);
      return null;
    }
    return entry.response;
  }

  set(url: string, response: HttpResponse<unknown>): void {
    this.cache.set(url, { response, timestamp: Date.now() });
  }

  invalidate(url: string): void {
    this.cache.delete(url);
  }

  clear(): void {
    this.cache.clear();
  }
}
```

### Retry Interceptor

```typescript
import { retry, timer } from 'rxjs';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  // Only retry GET requests (safe to repeat)
  if (req.method !== 'GET') {
    return next(req);
  }

  return next(req).pipe(
    retry({
      count: 3,
      delay: (error, retryCount) => {
        // Don't retry client errors (4xx)
        if (error.status >= 400 && error.status < 500) {
          throw error;
        }
        // Exponential backoff: 1s, 2s, 4s
        const delayMs = Math.pow(2, retryCount - 1) * 1000;
        console.warn(`Retrying ${req.url} in ${delayMs}ms (attempt ${retryCount})`);
        return timer(delayMs);
      }
    })
  );
};
```

---

## Legacy Class-Based Interceptors

```typescript
// Use withInterceptorsFromDi() to enable class-based interceptors
provideHttpClient(
  withInterceptorsFromDi()
)

// Register in providers
{ provide: HTTP_INTERCEPTORS, useClass: LegacyAuthInterceptor, multi: true }
```

```typescript
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';

@Injectable()
export class LegacyAuthInterceptor implements HttpInterceptor {
  private authService = inject(AuthService);

  intercept(req: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    const token = this.authService.getAccessToken();
    const authReq = token
      ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
      : req;
    return next.handle(authReq);
  }
}
```

---

## Error Handling

### Service-Level Error Handling

```typescript
@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);

  getById(id: number): Observable<Product> {
    return this.http.get<Product>(`/api/products/${id}`).pipe(
      catchError(this.handleError<Product>('getById'))
    );
  }

  private handleError<T>(operation: string) {
    return (error: HttpErrorResponse): Observable<never> => {
      let message: string;

      if (error.status === 0) {
        message = 'Network error. Please check your connection.';
      } else if (error.status === 404) {
        message = 'Resource not found.';
      } else if (error.status === 403) {
        message = 'You do not have permission to access this resource.';
      } else if (error.status === 422) {
        message = error.error?.message ?? 'Validation error.';
      } else if (error.status >= 500) {
        message = 'Server error. Please try again later.';
      } else {
        message = `Unexpected error in ${operation}.`;
      }

      console.error(`${operation} failed:`, error);
      return throwError(() => new Error(message));
    };
  }
}
```

### Component-Level Error Handling

```typescript
@Component({
  selector: 'app-product-detail',
  template: `
    @switch (state()) {
      @case ('loading') { <app-spinner /> }
      @case ('error') {
        <div class="error">
          <p>{{ error() }}</p>
          <button (click)="reload()">Try Again</button>
        </div>
      }
      @case ('loaded') {
        @if (product(); as product) {
          <h1>{{ product.name }}</h1>
          <p>{{ product.price | currency }}</p>
        }
      }
    }
  `
})
export class ProductDetailComponent {
  private productService = inject(ProductService);
  private route = inject(ActivatedRoute);

  product = signal<Product | null>(null);
  error = signal<string | null>(null);
  state = signal<'loading' | 'loaded' | 'error'>('loading');

  constructor() {
    this.loadProduct();
  }

  private loadProduct(): void {
    const id = this.route.snapshot.paramMap.get('id')!;
    this.state.set('loading');

    this.productService.getById(+id).pipe(
      takeUntilDestroyed()
    ).subscribe({
      next: (product) => {
        this.product.set(product);
        this.state.set('loaded');
      },
      error: (err) => {
        this.error.set(err.message);
        this.state.set('error');
      }
    });
  }

  reload(): void {
    this.loadProduct();
  }
}
```

---

## Request and Response Headers

```typescript
import { HttpHeaders } from '@angular/common/http';

// Creating headers
const headers = new HttpHeaders()
  .set('Content-Type', 'application/json')
  .set('Accept', 'application/json')
  .set('X-Custom-Header', 'value');

// Using headers in request
this.http.get<Data>('/api/data', { headers });

// Shorthand with object
this.http.post<Data>('/api/data', body, {
  headers: { 'Content-Type': 'application/json' }
});

// Reading response headers
this.http.get<Data>('/api/data', { observe: 'response' }).pipe(
  map(response => {
    const etag = response.headers.get('ETag');
    const contentType = response.headers.get('Content-Type');
    const totalCount = response.headers.get('X-Total-Count');
    return {
      data: response.body!,
      etag,
      totalCount: totalCount ? parseInt(totalCount) : 0
    };
  })
);
```

---

## Upload Progress Tracking

```typescript
import { HttpEventType, HttpEvent } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class UploadService {
  private http = inject(HttpClient);

  uploadFile(file: File): Observable<{ progress: number; response?: UploadResult }> {
    const formData = new FormData();
    formData.append('file', file, file.name);

    return this.http.post<UploadResult>('/api/upload', formData, {
      reportProgress: true,
      observe: 'events'
    }).pipe(
      map((event: HttpEvent<UploadResult>) => {
        switch (event.type) {
          case HttpEventType.UploadProgress:
            const progress = event.total
              ? Math.round((100 * event.loaded) / event.total)
              : 0;
            return { progress };
          case HttpEventType.Response:
            return { progress: 100, response: event.body! };
          default:
            return { progress: 0 };
        }
      })
    );
  }
}

// Component usage
@Component({
  template: `
    <input type="file" (change)="onFileSelected($event)" />
    @if (uploadProgress() > 0) {
      <progress [value]="uploadProgress()" max="100">{{ uploadProgress() }}%</progress>
    }
  `
})
export class FileUploadComponent {
  private uploadService = inject(UploadService);
  uploadProgress = signal(0);

  onFileSelected(event: Event): void {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) return;

    this.uploadService.uploadFile(file).subscribe({
      next: ({ progress, response }) => {
        this.uploadProgress.set(progress);
        if (response) {
          console.log('Upload complete:', response);
        }
      },
      error: (err) => console.error('Upload failed:', err)
    });
  }
}
```

---

## Testing with HttpTestingController

```typescript
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

describe('ProductService', () => {
  let service: ProductService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
        ProductService
      ]
    });

    service = TestBed.inject(ProductService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify(); // Verify no outstanding requests
  });

  it('should fetch products', () => {
    const mockProducts: Product[] = [
      { id: 1, name: 'Widget', price: 9.99, category: 'tools' }
    ];

    service.getAll().subscribe(products => {
      expect(products).toEqual(mockProducts);
      expect(products.length).toBe(1);
    });

    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('GET');
    req.flush(mockProducts);
  });

  it('should send correct body on POST', () => {
    const newProduct = { name: 'Gadget', price: 19.99, category: 'electronics' };

    service.create(newProduct).subscribe(result => {
      expect(result.id).toBeDefined();
    });

    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newProduct);
    req.flush({ ...newProduct, id: 2 });
  });

  it('should handle errors', () => {
    service.getById(999).subscribe({
      error: (err) => {
        expect(err.message).toContain('not found');
      }
    });

    const req = httpMock.expectOne('/api/products/999');
    req.flush('Not Found', { status: 404, statusText: 'Not Found' });
  });
});
```

---

## Best Practices

| Practice | Recommendation |
|----------|---------------|
| Use `provideHttpClient()` | Standalone API, replaces `HttpClientModule` |
| Use functional interceptors | Simpler than class-based, tree-shakable |
| Enable `withFetch()` | Better performance, required for SSR streaming |
| Create service abstractions | Wrap HttpClient in domain-specific services |
| Handle errors consistently | Use interceptors for global handling, services for specific |
| Use typed responses | Always provide type parameter to `get<T>()`, `post<T>()` |
| Cancel in-flight requests | Use `takeUntilDestroyed()` or `switchMap` |
| Test with `HttpTestingController` | Mock HTTP calls in unit tests |

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Not subscribing | HTTP request never fires | Observables are lazy; you must `subscribe()` or use `async` pipe |
| Multiple subscriptions | Request fires multiple times | Use `shareReplay(1)` or subscribe once |
| Missing `provideHttpClient()` | `NullInjectorError: No provider for HttpClient` | Add `provideHttpClient()` to app config providers |
| Not verifying in tests | Unmatched requests go undetected | Call `httpMock.verify()` in `afterEach` |
| Interceptor order matters | Auth interceptor runs before logging | Interceptors execute in array order |
| Not handling network errors | `status: 0` crashes error handling | Check for `status === 0` (network/CORS error) |
| Returning `HttpResponse` instead of body | Consumer receives wrapper object | Use default `observe: 'body'` unless you need headers/status |
| Forgetting to unsubscribe | Memory leaks on component destroy | Use `takeUntilDestroyed()` or `DestroyRef` |
