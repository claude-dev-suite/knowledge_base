# Angular Routing

> Official Documentation: https://angular.dev/guide/routing

## Route Configuration

### Basic Routes Array

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: 'products', component: ProductListComponent },
  { path: 'products/:id', component: ProductDetailComponent },
  { path: '**', component: NotFoundComponent }   // Wildcard (must be last)
];
```

### Application Configuration with provideRouter

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding, withViewTransitions } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),     // Bind route params to component inputs
      withViewTransitions(),           // Enable view transitions API
    ),
  ]
};
```

### Router Features

| Feature Function | Purpose |
|-----------------|---------|
| `withComponentInputBinding()` | Bind route params, query params, data to component inputs |
| `withViewTransitions()` | Enable the View Transitions API for route animations |
| `withPreloading(PreloadAllModules)` | Preload all lazy routes after initial load |
| `withDebugTracing()` | Log all router events to console (dev only) |
| `withHashLocation()` | Use hash-based URL strategy (`/#/path`) |
| `withRouterConfig({...})` | Advanced router configuration options |
| `withNavigationErrorHandler(fn)` | Custom handler for navigation errors |
| `withInMemoryScrolling({...})` | Configure scroll restoration behavior |

```typescript
provideRouter(
  routes,
  withComponentInputBinding(),
  withPreloading(PreloadAllModules),
  withInMemoryScrolling({
    scrollPositionRestoration: 'top',
    anchorScrolling: 'enabled'
  }),
  withNavigationErrorHandler((error) => {
    console.error('Navigation error:', error);
    inject(Router).navigate(['/error']);
  })
)
```

---

## Lazy Loading

### Lazy Loading Standalone Components

```typescript
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
  },
  {
    path: 'settings',
    loadComponent: () => import('./settings/settings.component')
      .then(m => m.SettingsComponent)
  }
];
```

### Lazy Loading Child Routes

```typescript
export const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.adminRoutes)
  }
];

// admin/admin.routes.ts
export const adminRoutes: Routes = [
  { path: '', component: AdminDashboardComponent },
  { path: 'users', component: AdminUsersComponent },
  { path: 'users/:id', component: AdminUserDetailComponent },
  { path: 'settings', component: AdminSettingsComponent }
];
```

### Lazy Loading with Default Export

```typescript
// Shorter syntax using default exports
export const routes: Routes = [
  {
    path: 'profile',
    loadComponent: () => import('./profile/profile.component')
    // Works if the component is the default export
  }
];
```

---

## Route Guards (Functional)

Angular 15+ recommends functional guards over class-based guards.

### canActivate - Protect Route Access

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  // Redirect to login with return URL
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};

// Usage in routes
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent),
    canActivate: [authGuard]
  }
];
```

### Role-Based Guard

```typescript
export const roleGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  const requiredRoles = route.data['roles'] as string[];

  const user = authService.currentUser();
  if (!user) {
    return router.createUrlTree(['/login']);
  }

  const hasRole = requiredRoles.some(role => user.roles.includes(role));
  if (!hasRole) {
    return router.createUrlTree(['/unauthorized']);
  }

  return true;
};

// Usage
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes),
  canActivate: [authGuard, roleGuard],
  data: { roles: ['admin', 'superadmin'] }
}
```

### canDeactivate - Prevent Leaving with Unsaved Changes

```typescript
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (
  component,
  currentRoute,
  currentState,
  nextState
) => {
  if (component.hasUnsavedChanges()) {
    return window.confirm('You have unsaved changes. Leave anyway?');
  }
  return true;
};

// Component implementation
@Component({ /* ... */ })
export class EditFormComponent implements HasUnsavedChanges {
  isDirty = signal(false);

  hasUnsavedChanges(): boolean {
    return this.isDirty();
  }
}

// Route config
{
  path: 'edit/:id',
  component: EditFormComponent,
  canDeactivate: [unsavedChangesGuard]
}
```

### canMatch - Conditionally Match Routes

```typescript
export const featureFlagGuard: CanMatchFn = (route, segments) => {
  const featureService = inject(FeatureFlagService);
  return featureService.isEnabled(route.data?.['feature'] ?? '');
};

export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard-v2/dashboard.component')
      .then(m => m.DashboardV2Component),
    canMatch: [featureFlagGuard],
    data: { feature: 'new-dashboard' }
  },
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
    // Fallback when feature flag is off
  }
];
```

---

## Resolvers

Resolvers prefetch data before route activation.

```typescript
// user.resolver.ts
import { ResolveFn } from '@angular/router';
import { inject } from '@angular/core';

export const userResolver: ResolveFn<User> = (route, state) => {
  const userService = inject(UserService);
  const userId = route.paramMap.get('id')!;
  return userService.getById(userId);
};

// Route config
{
  path: 'users/:id',
  loadComponent: () => import('./user-detail/user-detail.component')
    .then(m => m.UserDetailComponent),
  resolve: { user: userResolver }
}

// Component - access resolved data via input (with withComponentInputBinding)
@Component({ /* ... */ })
export class UserDetailComponent {
  user = input.required<User>();  // Automatically bound from resolver
}
```

### Resolver with Error Handling

```typescript
export const productResolver: ResolveFn<Product | null> = (route, state) => {
  const productService = inject(ProductService);
  const router = inject(Router);
  const productId = route.paramMap.get('id')!;

  return productService.getById(productId).pipe(
    catchError(error => {
      console.error('Failed to load product:', error);
      router.navigate(['/products']);
      return of(null);
    })
  );
};
```

---

## Route Parameters

### Path Parameters

```typescript
// Route: { path: 'products/:id', component: ProductDetailComponent }

// Option 1: withComponentInputBinding (Preferred)
@Component({ /* ... */ })
export class ProductDetailComponent {
  id = input.required<string>();  // Bound from :id param

  product = computed(() => {
    // React to id changes
    return this.id();
  });
}

// Option 2: ActivatedRoute (when not using input binding)
@Component({ /* ... */ })
export class ProductDetailComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private destroyRef = inject(DestroyRef);

  product = signal<Product | null>(null);

  ngOnInit(): void {
    this.route.paramMap.pipe(
      map(params => params.get('id')!),
      switchMap(id => this.productService.getById(id)),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(product => this.product.set(product));
  }
}
```

### Query Parameters

```typescript
// URL: /products?category=electronics&sort=price

// Option 1: Input binding
@Component({ /* ... */ })
export class ProductListComponent {
  category = input<string>('');  // Bound from ?category=
  sort = input<string>('name');  // Bound from ?sort=
}

// Option 2: ActivatedRoute
@Component({ /* ... */ })
export class ProductListComponent {
  private route = inject(ActivatedRoute);

  ngOnInit(): void {
    this.route.queryParamMap.pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(params => {
      const category = params.get('category') ?? '';
      const sort = params.get('sort') ?? 'name';
      this.loadProducts(category, sort);
    });
  }
}
```

### Route Data and Title

```typescript
export const routes: Routes = [
  {
    path: 'about',
    component: AboutComponent,
    title: 'About Us',                    // Static title
    data: { animation: 'about', breadcrumb: 'About' }
  },
  {
    path: 'products/:id',
    component: ProductDetailComponent,
    title: productTitleResolver,          // Dynamic title resolver
    data: { breadcrumb: 'Product Detail' }
  }
];

// Title resolver
export const productTitleResolver: ResolveFn<string> = (route) => {
  const productService = inject(ProductService);
  const id = route.paramMap.get('id')!;
  return productService.getById(id).pipe(
    map(product => `${product.name} - Products`)
  );
};

// Custom title strategy
@Injectable({ providedIn: 'root' })
export class AppTitleStrategy extends TitleStrategy {
  private title = inject(Title);

  override updateTitle(snapshot: RouterStateSnapshot): void {
    const title = this.buildTitle(snapshot);
    this.title.setTitle(title ? `${title} | MyApp` : 'MyApp');
  }
}

// Register in providers
providers: [
  { provide: TitleStrategy, useClass: AppTitleStrategy }
]
```

---

## Nested Routes and Router Outlets

### Nested Route Configuration

```typescript
export const routes: Routes = [
  {
    path: 'settings',
    component: SettingsLayoutComponent,
    children: [
      { path: '', redirectTo: 'profile', pathMatch: 'full' },
      { path: 'profile', component: ProfileSettingsComponent },
      { path: 'security', component: SecuritySettingsComponent },
      { path: 'notifications', component: NotificationSettingsComponent },
      { path: 'billing', component: BillingSettingsComponent }
    ]
  }
];
```

### Layout Component with Router Outlet

```typescript
@Component({
  selector: 'app-settings-layout',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <div class="settings-layout">
      <nav class="sidebar">
        <a routerLink="profile"
           routerLinkActive="active"
           [routerLinkActiveOptions]="{ exact: true }">
          Profile
        </a>
        <a routerLink="security" routerLinkActive="active">Security</a>
        <a routerLink="notifications" routerLinkActive="active">Notifications</a>
        <a routerLink="billing" routerLinkActive="active">Billing</a>
      </nav>
      <main class="content">
        <router-outlet />
      </main>
    </div>
  `
})
export class SettingsLayoutComponent {}
```

### Named Outlets

```typescript
export const routes: Routes = [
  {
    path: 'app',
    component: AppLayoutComponent,
    children: [
      { path: 'dashboard', component: DashboardComponent },
      { path: 'chat', component: ChatPanelComponent, outlet: 'sidebar' },
      { path: 'help', component: HelpPanelComponent, outlet: 'sidebar' }
    ]
  }
];

// Layout template
@Component({
  template: `
    <div class="main">
      <router-outlet />
    </div>
    <aside class="sidebar">
      <router-outlet name="sidebar" />
    </aside>
  `
})
export class AppLayoutComponent {}

// Navigation to named outlet
// URL: /app/dashboard(sidebar:chat)
```

---

## Navigation

### Programmatic Navigation

```typescript
import { Router } from '@angular/router';

@Component({ /* ... */ })
export class NavComponent {
  private router = inject(Router);

  // Basic navigation
  goHome(): void {
    this.router.navigate(['/']);
  }

  // Navigate with params
  viewProduct(id: string): void {
    this.router.navigate(['/products', id]);
  }

  // Navigate with query params
  searchProducts(term: string): void {
    this.router.navigate(['/products'], {
      queryParams: { search: term, page: 1 },
      queryParamsHandling: 'merge'  // Preserve existing query params
    });
  }

  // Navigate with fragment
  goToSection(): void {
    this.router.navigate(['/docs'], { fragment: 'installation' });
  }

  // Replace current history entry
  replaceRoute(): void {
    this.router.navigate(['/new-page'], { replaceUrl: true });
  }

  // Navigate relative to current route
  goToChild(): void {
    this.router.navigate(['child'], { relativeTo: this.route });
  }

  // navigateByUrl - accepts full URL string
  goByUrl(): void {
    this.router.navigateByUrl('/products/123?tab=reviews#specs');
  }
}
```

### Template-Based Navigation

```html
<!-- Basic routerLink -->
<a routerLink="/products">Products</a>

<!-- With parameters -->
<a [routerLink]="['/products', product.id]">{{ product.name }}</a>

<!-- With query params -->
<a [routerLink]="['/products']"
   [queryParams]="{ category: 'electronics', sort: 'price' }"
   queryParamsHandling="merge">
  Electronics
</a>

<!-- Active link styling -->
<a routerLink="/dashboard"
   routerLinkActive="active font-bold"
   [routerLinkActiveOptions]="{ exact: true }">
  Dashboard
</a>

<!-- Check if link is active programmatically -->
<a routerLink="/products"
   routerLinkActive
   #rla="routerLinkActive">
  Products {{ rla.isActive ? '(current)' : '' }}
</a>
```

---

## Wildcard and Redirect Routes

```typescript
export const routes: Routes = [
  // Redirect root to dashboard
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },

  // Redirect old routes to new ones
  { path: 'old-products', redirectTo: 'products', pathMatch: 'full' },
  { path: 'old-products/:id', redirectTo: 'products/:id' },

  // Feature routes
  { path: 'dashboard', loadComponent: () => import('./dashboard/dashboard.component').then(m => m.DashboardComponent) },
  { path: 'products', loadComponent: () => import('./products/product-list.component').then(m => m.ProductListComponent) },
  { path: 'products/:id', loadComponent: () => import('./products/product-detail.component').then(m => m.ProductDetailComponent) },

  // Wildcard - catch all unmatched routes (must be last)
  { path: '**', component: NotFoundComponent }
];
```

### pathMatch Options

| Value | Behavior |
|-------|----------|
| `'full'` | Entire URL must match the path exactly |
| `'prefix'` | URL starts with the path (default) |

```typescript
// pathMatch: 'full' is required for empty path redirects
{ path: '', redirectTo: 'home', pathMatch: 'full' }

// pathMatch: 'prefix' (default) matches any URL starting with this segment
{ path: 'admin', loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes) }
```

---

## Router Events

```typescript
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';

@Component({ /* ... */ })
export class AppComponent {
  private router = inject(Router);
  loading = signal(false);

  constructor() {
    this.router.events.pipe(
      takeUntilDestroyed()
    ).subscribe(event => {
      if (event instanceof NavigationStart) {
        this.loading.set(true);
      }
      if (event instanceof NavigationEnd ||
          event instanceof NavigationCancel ||
          event instanceof NavigationError) {
        this.loading.set(false);
      }
    });
  }
}
```

---

## Best Practices

| Practice | Recommendation |
|----------|---------------|
| Use lazy loading | Always `loadComponent` / `loadChildren` for non-root routes |
| Use `withComponentInputBinding()` | Bind route params directly to component inputs |
| Prefer functional guards | Use `CanActivateFn` over class-based guards |
| Keep route files separate | Define routes in `*.routes.ts` files |
| Use title resolvers | Dynamic page titles improve accessibility and SEO |
| Preload strategically | Use `withPreloading(PreloadAllModules)` for small apps |
| Handle navigation errors | Use `withNavigationErrorHandler()` for global error handling |
| Organize by feature | Group related routes, guards, and resolvers by feature module |

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Missing `pathMatch: 'full'` on redirects | Redirect matches all routes | Add `pathMatch: 'full'` to empty-path redirects |
| Wildcard before other routes | Wildcard catches everything | Always place `**` route last |
| Not unsubscribing from route params | Memory leaks | Use `takeUntilDestroyed()` or input binding |
| Hardcoded URLs in components | Breaks on route changes | Use `routerLink` or `Router.navigate` with route arrays |
| Forgetting RouterOutlet import | Child routes do not render | Import `RouterOutlet` in standalone components |
| Guards returning `false` silently | User stuck with no feedback | Return `UrlTree` to redirect, or show a message |
| Resolver blocking navigation | Slow resolvers delay page load | Use loading states or `resource()` in components instead |
| Missing `RouterLink` import | `routerLink` directive does nothing | Import `RouterLink` in the standalone component |
