# Angular Server-Side Rendering (SSR)

> Official Documentation: https://angular.dev/guide/ssr

## Overview

Angular SSR renders application pages on the server, producing HTML that is sent to the browser. This improves initial load performance, SEO, and social media sharing. Angular uses `@angular/ssr` (formerly Angular Universal) for server-side rendering and hydration.

---

## Setup

### Adding SSR to an Existing Project

```bash
ng add @angular/ssr
```

This command:
- Installs `@angular/ssr` package
- Creates `server.ts` entry point
- Updates `angular.json` with server build config
- Adds server-specific code to `app.config.server.ts`

### Generated Server Configuration

```typescript
// app.config.server.ts
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { provideServerRouting } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    provideServerRouting(serverRoutes)
  ]
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

```typescript
// app.config.ts (browser config)
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideClientHydration(withEventReplay()),
    provideHttpClient(withFetch())   // withFetch() recommended for SSR
  ]
};
```

---

## RenderMode

Angular provides three rendering modes for routes: Server, Client, and Prerender.

### Server Routes Configuration

```typescript
// app.routes.server.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  // Server-rendered on each request
  {
    path: 'dashboard',
    renderMode: RenderMode.Server
  },

  // Client-only rendering (no SSR)
  {
    path: 'admin/**',
    renderMode: RenderMode.Client
  },

  // Pre-rendered at build time (static HTML)
  {
    path: '',
    renderMode: RenderMode.Prerender
  },
  {
    path: 'about',
    renderMode: RenderMode.Prerender
  },

  // Pre-render with parameters
  {
    path: 'products/:id',
    renderMode: RenderMode.Prerender,
    async getPrerenderParams() {
      const productService = inject(ProductService);
      const ids = await productService.getAllIds();
      return ids.map(id => ({ id: id.toString() }));
    }
  },

  // Fallback for all other routes
  {
    path: '**',
    renderMode: RenderMode.Server
  }
];
```

### RenderMode Comparison

| Mode | When HTML is Generated | Use Case |
|------|----------------------|----------|
| `RenderMode.Server` | On each request (SSR) | Dynamic content, authenticated pages |
| `RenderMode.Client` | In the browser only (CSR) | Admin panels, highly interactive pages |
| `RenderMode.Prerender` | At build time (SSG) | Static pages, marketing pages, blogs |

---

## Hydration

Hydration is the process where Angular takes the server-rendered HTML and attaches event listeners and Angular functionality to it, making it interactive without re-rendering.

### Non-Destructive Hydration

```typescript
// app.config.ts
import { provideClientHydration } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration()   // Enables non-destructive hydration
  ]
};
```

Non-destructive hydration:
- Reuses existing server-rendered DOM nodes
- Avoids layout shift caused by re-rendering
- Preserves user interactions during hydration

### Event Replay

Event replay captures user interactions (clicks, inputs) that happen before hydration completes and replays them after hydration.

```typescript
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(
      withEventReplay()    // Replay events that occurred before hydration
    )
  ]
};
```

### Incremental Hydration

Incremental hydration allows deferring the hydration of specific parts of the page.

```typescript
@Component({
  selector: 'app-page',
  template: `
    <app-header />

    <main>
      <app-hero-banner />

      <!-- Defer hydration until visible in viewport -->
      @defer (on viewport; hydrate on viewport) {
        <app-product-grid [products]="products()" />
      }

      <!-- Defer hydration until user interacts -->
      @defer (on interaction; hydrate on interaction) {
        <app-comments [postId]="postId()" />
      }

      <!-- Defer hydration until idle -->
      @defer (on idle; hydrate on idle) {
        <app-recommendations />
      }

      <!-- Never hydrate on server (client-only) -->
      @defer (hydrate never) {
        <app-chat-widget />
      }
    </main>

    <app-footer />
  `
})
export class PageComponent {
  products = signal<Product[]>([]);
  postId = input.required<string>();
}
```

### Hydration Triggers

| Trigger | Description |
|---------|-------------|
| `hydrate on viewport` | Hydrates when component enters viewport |
| `hydrate on interaction` | Hydrates on first user interaction (click, keydown) |
| `hydrate on idle` | Hydrates when browser is idle |
| `hydrate on immediate` | Hydrates immediately after page load |
| `hydrate on timer(ms)` | Hydrates after specified delay |
| `hydrate on hover` | Hydrates when user hovers over the area |
| `hydrate never` | Never hydrates (stays as static HTML) |

---

## Platform Checks

Code that accesses browser-only APIs (window, document, localStorage) must be guarded for SSR compatibility.

### Using isPlatformBrowser / isPlatformServer

```typescript
import { Component, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

@Component({
  selector: 'app-analytics',
  template: `<div #container></div>`
})
export class AnalyticsComponent {
  private platformId = inject(PLATFORM_ID);

  ngOnInit(): void {
    if (isPlatformBrowser(this.platformId)) {
      // Safe to use browser APIs
      window.addEventListener('scroll', this.onScroll);
      this.initGoogleAnalytics();
    }

    if (isPlatformServer(this.platformId)) {
      // Server-only logic
      console.log('Rendering on server');
    }
  }

  ngOnDestroy(): void {
    if (isPlatformBrowser(this.platformId)) {
      window.removeEventListener('scroll', this.onScroll);
    }
  }

  private onScroll = (): void => {
    // Track scroll depth
  };

  private initGoogleAnalytics(): void {
    // Initialize GA
  }
}
```

### Using afterNextRender (Preferred)

`afterNextRender` is the preferred way to run browser-only code. It only executes in the browser, never on the server.

```typescript
import { Component, afterNextRender, viewChild, ElementRef } from '@angular/core';

@Component({
  selector: 'app-map',
  template: `<div #mapContainer class="map"></div>`
})
export class MapComponent {
  mapContainer = viewChild.required<ElementRef<HTMLDivElement>>('mapContainer');
  private map: google.maps.Map | null = null;

  constructor() {
    afterNextRender(() => {
      // This only runs in the browser, after the first render
      this.map = new google.maps.Map(this.mapContainer().nativeElement, {
        center: { lat: 40.7128, lng: -74.0060 },
        zoom: 12
      });
    });
  }
}
```

```typescript
@Component({
  selector: 'app-chart',
  template: `<canvas #canvas></canvas>`
})
export class ChartComponent {
  canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');
  private chart: Chart | null = null;

  data = input.required<ChartData>();

  constructor() {
    // Runs once after first render (browser only)
    afterNextRender(() => {
      this.chart = new Chart(this.canvas().nativeElement, {
        type: 'line',
        data: this.data()
      });
    });
  }
}
```

---

## Transfer State

Transfer state allows data fetched on the server to be transferred to the client, avoiding duplicate HTTP requests during hydration.

### Automatic Transfer State with HttpClient

When using `provideClientHydration()`, HttpClient responses are automatically transferred from server to client. No additional configuration needed.

```typescript
// This service works seamlessly with SSR transfer state
@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);

  getAll(): Observable<Product[]> {
    // During SSR: makes real HTTP request, caches response
    // During hydration: returns cached response without making a new request
    return this.http.get<Product[]>('/api/products');
  }
}
```

### Manual Transfer State

For non-HTTP data, use `TransferState` manually:

```typescript
import { TransferState, makeStateKey } from '@angular/core';

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Injectable({ providedIn: 'root' })
export class ProductService {
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);
  private http = inject(HttpClient);

  getProducts(): Observable<Product[]> {
    // Check if data was transferred from server
    const cached = this.transferState.get(PRODUCTS_KEY, null);
    if (cached) {
      this.transferState.remove(PRODUCTS_KEY); // Clean up
      return of(cached);
    }

    return this.http.get<Product[]>('/api/products').pipe(
      tap(products => {
        if (isPlatformServer(this.platformId)) {
          // Store for transfer to client
          this.transferState.set(PRODUCTS_KEY, products);
        }
      })
    );
  }
}
```

---

## Prerendering Static Routes

```typescript
// app.routes.server.ts
export const serverRoutes: ServerRoute[] = [
  // Static pages - prerendered at build time
  { path: '', renderMode: RenderMode.Prerender },
  { path: 'about', renderMode: RenderMode.Prerender },
  { path: 'pricing', renderMode: RenderMode.Prerender },
  { path: 'contact', renderMode: RenderMode.Prerender },

  // Blog posts with dynamic params
  {
    path: 'blog/:slug',
    renderMode: RenderMode.Prerender,
    async getPrerenderParams() {
      // Fetch all blog slugs to prerender
      const response = await fetch('https://api.example.com/blog/slugs');
      const slugs: string[] = await response.json();
      return slugs.map(slug => ({ slug }));
    }
  },

  // Product pages - prerender top products, SSR the rest
  {
    path: 'products/:id',
    renderMode: RenderMode.Prerender,
    fallback: RenderMode.Server,    // SSR for non-prerendered products
    async getPrerenderParams() {
      const response = await fetch('https://api.example.com/products/featured');
      const products: Product[] = await response.json();
      return products.map(p => ({ id: p.id.toString() }));
    }
  },

  // Everything else - SSR
  { path: '**', renderMode: RenderMode.Server }
];
```

---

## SSR with HttpClient

### Configuring HttpClient for SSR

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withFetch()    // Recommended for SSR - uses Node.js-compatible fetch
    ),
    provideClientHydration(withEventReplay())
  ]
};
```

### Handling Absolute URLs on Server

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { isPlatformServer } from '@angular/common';
import { PLATFORM_ID, inject } from '@angular/core';

export const baseUrlInterceptor: HttpInterceptorFn = (req, next) => {
  const platformId = inject(PLATFORM_ID);

  if (isPlatformServer(platformId) && req.url.startsWith('/')) {
    // Convert relative URLs to absolute for server-side requests
    const serverUrl = process.env['API_BASE_URL'] ?? 'http://localhost:4000';
    const absoluteReq = req.clone({ url: `${serverUrl}${req.url}` });
    return next(absoluteReq);
  }

  return next(req);
};
```

---

## SSR-Safe Services

```typescript
@Injectable({ providedIn: 'root' })
export class LocalStorageService {
  private platformId = inject(PLATFORM_ID);
  private memoryStorage = new Map<string, string>();

  getItem(key: string): string | null {
    if (isPlatformBrowser(this.platformId)) {
      return localStorage.getItem(key);
    }
    return this.memoryStorage.get(key) ?? null;
  }

  setItem(key: string, value: string): void {
    if (isPlatformBrowser(this.platformId)) {
      localStorage.setItem(key, value);
    }
    this.memoryStorage.set(key, value);
  }

  removeItem(key: string): void {
    if (isPlatformBrowser(this.platformId)) {
      localStorage.removeItem(key);
    }
    this.memoryStorage.delete(key);
  }
}
```

```typescript
// Using InjectionToken for SSR-safe window access
export const WINDOW = new InjectionToken<Window | null>('Window', {
  providedIn: 'root',
  factory: () => {
    const platformId = inject(PLATFORM_ID);
    return isPlatformBrowser(platformId) ? window : null;
  }
});

@Injectable({ providedIn: 'root' })
export class ScrollService {
  private window = inject(WINDOW);

  scrollToTop(): void {
    this.window?.scrollTo({ top: 0, behavior: 'smooth' });
  }

  getScrollPosition(): number {
    return this.window?.scrollY ?? 0;
  }
}
```

---

## Best Practices

| Practice | Recommendation |
|----------|---------------|
| Use `afterNextRender` for browser APIs | Never access `window`, `document`, `localStorage` directly |
| Enable `withFetch()` | Required for SSR streaming, better Node.js compatibility |
| Enable `withEventReplay()` | Captures user interactions during hydration |
| Prerender static pages | Use `RenderMode.Prerender` for marketing, about, pricing pages |
| Use `RenderMode.Client` for admin | Skip SSR for authenticated/interactive-heavy pages |
| Provide absolute URLs on server | Relative URLs fail on server; use interceptor for base URL |
| Use transfer state | Avoid duplicate HTTP requests during hydration |
| Use `@defer` with `hydrate` | Incrementally hydrate for better performance |
| Guard browser-only code | Always check platform before using browser APIs |

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Accessing `window` directly | `ReferenceError: window is not defined` on server | Use `isPlatformBrowser`, `afterNextRender`, or `WINDOW` token |
| Accessing `document` directly | Breaks on server | Use `Renderer2` or `afterNextRender` |
| Using `localStorage` in constructor | Fails during SSR | Guard with `isPlatformBrowser` or use SSR-safe wrapper |
| Relative API URLs | Server cannot resolve `/api/...` | Use interceptor to prepend base URL on server |
| Missing `provideClientHydration()` | No hydration, full re-render on client | Add `provideClientHydration()` to app config |
| Heavy computation in SSR | Slow Time to First Byte (TTFB) | Use `RenderMode.Client` for heavy pages, or prerender |
| Not using `withFetch()` | SSR streaming unavailable | Add `withFetch()` to `provideHttpClient()` |
| Hydration mismatch | Console warning, DOM is re-created | Ensure server and client render identical HTML |
| Third-party scripts in SSR | Scripts fail on server | Load third-party scripts with `afterNextRender` |
| Memory leaks on server | `setInterval` runs indefinitely | Always clear intervals; use `afterNextRender` for browser timers |
